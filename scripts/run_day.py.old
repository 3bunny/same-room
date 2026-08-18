#!/usr/bin/env python3
"""
The Same Room — one prompt, one room, every day for a year.

Runs once per day. Generates today's image from a locked prompt, composes the
vertical frame, renders an 8-second Short, uploads it, and appends a log line.

Design rules that are enforced in code, not just in spirit:
  * The prompt file is hash-checked. If it changes, the run aborts.
  * A day that already has an archived image is never re-generated.
  * There is no retry-until-it-looks-good. One call, one result, it ships.
"""

from __future__ import annotations

import base64
import datetime as dt
import hashlib
import json
import os
import subprocess
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
ARCHIVE = ROOT / "archive"
BUILD = ROOT / "build"
LOG = ROOT / "log.jsonl"


# ----------------------------------------------------------------------------
# setup
# ----------------------------------------------------------------------------

def load_config() -> dict:
    return json.loads((ROOT / "config.json").read_text(encoding="utf-8"))


def load_locked_prompt(cfg: dict) -> str:
    raw = (ROOT / "prompt.txt").read_bytes()
    digest = hashlib.sha256(raw).hexdigest()
    if digest != cfg["prompt_sha256"]:
        sys.exit(
            "ABORT: prompt.txt does not match the locked hash.\n"
            f"  expected {cfg['prompt_sha256']}\n"
            f"  found    {digest}\n"
            "The prompt is the experiment. If it genuinely needs to change, that "
            "ends this run of the series and starts a new one."
        )
    return raw.decode("utf-8").strip()


def day_number(cfg: dict, today: dt.date) -> int:
    start = dt.date.fromisoformat(cfg["start_date"])
    return (today - start).days + 1


# ----------------------------------------------------------------------------
# generation
# ----------------------------------------------------------------------------

def generate_image(prompt: str, cfg: dict, out_path: Path) -> None:
    from google import genai

    client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
    model = cfg["image_model"]

    data = None

    # Newer SDKs expose an interactions surface with a convenience image accessor.
    try:
        interaction = client.interactions.create(model=model, input=prompt)
        data = base64.b64decode(interaction.output_image.data)
    except (AttributeError, NotImplementedError):
        pass

    # Fall back to generate_content and pull the first inline image part.
    if data is None:
        resp = client.models.generate_content(model=model, contents=prompt)
        for cand in resp.candidates:
            for part in cand.content.parts:
                inline = getattr(part, "inline_data", None)
                if inline and inline.data:
                    data = inline.data
                    if isinstance(data, str):
                        data = base64.b64decode(data)
                    break
            if data:
                break

    if not data:
        raise RuntimeError("No image data returned by the model.")

    out_path.write_bytes(data)


# ----------------------------------------------------------------------------
# composition
# ----------------------------------------------------------------------------

def _font(size: int, bold: bool = False):
    from PIL import ImageFont

    names = [
        "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf" if bold
        else "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
        "/usr/share/fonts/truetype/liberation/LiberationSans-Bold.ttf" if bold
        else "/usr/share/fonts/truetype/liberation/LiberationSans-Regular.ttf",
    ]
    for n in names:
        if Path(n).exists():
            return ImageFont.truetype(n, size)
    return ImageFont.load_default()


def _fit(img, width: int):
    w, h = img.size
    return img.resize((width, max(1, round(h * width / w))))


def _centre(draw, y: int, text: str, font, fill, frame_w: int):
    box = draw.textbbox((0, 0), text, font=font)
    draw.text(((frame_w - (box[2] - box[0])) // 2, y), text, font=font, fill=fill)


def compose_frame(cfg: dict, day: int, today_img: Path, day1_img: Path | None,
                  date_str: str, out_path: Path) -> None:
    from PIL import Image, ImageDraw

    W, H = cfg["frame_width"], cfg["frame_height"]
    BG, INK, DIM = (13, 14, 16), (232, 230, 225), (138, 143, 149)

    frame = Image.new("RGB", (W, H), BG)
    draw = ImageDraw.Draw(frame)

    _centre(draw, 96, "THE SAME ROOM", _font(46, bold=True), INK, W)
    _centre(draw, 158, f"one prompt  ·  unchanged  ·  day {day} of {cfg['total_days']}",
            _font(28), DIM, W)

    main = _fit(Image.open(today_img).convert("RGB"), 1000)
    frame.paste(main, ((W - 1000) // 2, 290))
    y = 290 + main.size[1] + 34

    _centre(draw, y, f"DAY {day:03d}   ·   {date_str}", _font(34, bold=True), INK, W)
    y += 76

    if day1_img and day1_img.exists() and day > 1:
        tw = 486
        a = _fit(Image.open(day1_img).convert("RGB"), tw)
        b = _fit(Image.open(today_img).convert("RGB"), tw)
        top = min(y + 40, H - 300)
        frame.paste(a, (48, top))
        frame.paste(b, (W - 48 - tw, top))
        lab = _font(24, bold=True)
        draw.text((48, top + a.size[1] + 12), "DAY 001", font=lab, fill=DIM)
        draw.text((W - 48 - tw, top + b.size[1] + 12), "TODAY", font=lab, fill=DIM)

    _centre(draw, H - 92, f"prompt sha256 {cfg['prompt_sha256'][:16]}", _font(20), DIM, W)

    frame.save(out_path)


def render_video(cfg: dict, frame_path: Path, out_path: Path) -> None:
    # A static frame, deliberately. This is a channel about a room nobody moves.
    subprocess.run(
        [
            "ffmpeg", "-y", "-loglevel", "error",
            "-loop", "1", "-i", str(frame_path),
            "-f", "lavfi", "-i", "anullsrc=r=44100:cl=stereo",
            "-t", str(cfg["video_seconds"]),
            "-c:v", "libx264", "-preset", "medium", "-crf", "18",
            "-pix_fmt", "yuv420p", "-r", "30",
            "-c:a", "aac", "-b:a", "128k", "-shortest",
            str(out_path),
        ],
        check=True,
    )


# ----------------------------------------------------------------------------
# upload
# ----------------------------------------------------------------------------

def upload(cfg: dict, video: Path, day: int, date_str: str, prompt: str) -> str | None:
    if os.environ.get("SKIP_UPLOAD") == "1":
        print("SKIP_UPLOAD=1 — not uploading.")
        return None

    from google.oauth2.credentials import Credentials
    from googleapiclient.discovery import build
    from googleapiclient.http import MediaFileUpload

    creds = Credentials(
        None,
        refresh_token=os.environ["YT_REFRESH_TOKEN"],
        client_id=os.environ["YT_CLIENT_ID"],
        client_secret=os.environ["YT_CLIENT_SECRET"],
        token_uri="https://oauth2.googleapis.com/token",
        scopes=["https://www.googleapis.com/auth/youtube.upload"],
    )
    youtube = build("youtube", "v3", credentials=creds, cache_discovery=False)

    description = (
        f"Day {day} of {cfg['total_days']}.\n\n"
        "One prompt, describing one room, sent unchanged every day for a year. "
        "Nothing is edited. Nothing is re-rolled. Bad days stay up.\n\n"
        "THE LOCKED PROMPT\n"
        f"{prompt}\n\n"
        f"prompt sha256: {cfg['prompt_sha256']}\n"
        f"model: {cfg['image_model']}\n"
        f"generated: {date_str}\n\n"
        "Every render is committed to a public git repository on the day it is made, "
        "so the timeline is verifiable rather than claimed.\n\n"
        "#Shorts"
    )

    body = {
        "snippet": {
            "title": f"The Same Room — Day {day:03d}",
            "description": description,
            "tags": ["the same room", "ai art", "model drift", "generative", "shorts"],
            "categoryId": "22",
        },
        "status": {
            "privacyStatus": os.environ.get("YT_PRIVACY", "private"),
            "selfDeclaredMadeForKids": False,
        },
    }

    req = youtube.videos().insert(
        part="snippet,status",
        body=body,
        media_body=MediaFileUpload(str(video), chunksize=-1, resumable=True),
    )
    resp = req.execute()
    return resp.get("id")


# ----------------------------------------------------------------------------
# main
# ----------------------------------------------------------------------------

def main() -> None:
    cfg = load_config()
    prompt = load_locked_prompt(cfg)

    tz = dt.timezone(dt.timedelta(hours=9))  # the series runs on Seoul time
    now = dt.datetime.now(tz)
    today = now.date()
    date_str = today.isoformat()

    day = day_number(cfg, today)
    if day < 1:
        sys.exit(f"Series has not started yet (starts {cfg['start_date']}).")
    if day > cfg["total_days"]:
        print(f"Day {day} is past the end of the series. Nothing to do.")
        return

    ARCHIVE.mkdir(exist_ok=True)
    BUILD.mkdir(exist_ok=True)

    today_img = ARCHIVE / f"day-{day:04d}.png"
    if today_img.exists():
        print(f"Day {day} already archived at {today_img.name}. Not regenerating.")
        return

    print(f"Day {day} — generating with {cfg['image_model']}")
    generate_image(prompt, cfg, today_img)

    frame = BUILD / f"frame-{day:04d}.png"
    video = BUILD / f"day-{day:04d}.mp4"
    compose_frame(cfg, day, today_img, ARCHIVE / "day-0001.png", date_str, frame)
    render_video(cfg, frame, video)

    video_id = upload(cfg, video, day, date_str, prompt)

    entry = {
        "day": day,
        "date": date_str,
        "generated_at": now.isoformat(),
        "model": cfg["image_model"],
        "prompt_sha256": cfg["prompt_sha256"],
        "image": str(today_img.relative_to(ROOT)),
        "image_sha256": hashlib.sha256(today_img.read_bytes()).hexdigest(),
        "bytes": today_img.stat().st_size,
        "video_id": video_id,
        "privacy": os.environ.get("YT_PRIVACY", "private"),
        "qc": None,
    }
    with LOG.open("a", encoding="utf-8") as fh:
        fh.write(json.dumps(entry) + "\n")

    print(json.dumps(entry, indent=2))


if __name__ == "__main__":
    main()
