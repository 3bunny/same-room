# The Same Room

One prompt, describing one room, sent unchanged every day for a year.
Nothing is edited. Nothing is re-rolled. Bad days stay up.

What accumulates is a time-lapse of the model itself moving underneath a fixed
input. The daily git commit is what makes the timeline verifiable rather than
merely claimed — which is the whole reason the project is worth doing.

---

## Your setup list

Everything else is already written. These are the steps that need your login.

### 1 — Make the channel

YouTube → profile picture → **Settings** → **Add or manage your channel(s)** →
**Create a channel**. Name it *The Same Room*. It becomes a Brand Account under
your existing Google account; you switch to it from the profile menu.

### 2 — Make the repository

New **private** repository on GitHub, then drop these files in. The daily render
gets committed back into `archive/`, so the repo is the archive.

> Make it **public** when you're ready. A public commit history is the proof that
> day 200 was really made on day 200. Private works fine mechanically, but you
> lose the thing that makes the experiment credible to anyone else.

### 3 — Gemini API key

[aistudio.google.com](https://aistudio.google.com) → **Get API key** → create it
in a Google Cloud project → **enable Cloud Billing on that project**.

Your Google AI Pro subscription does **not** cover this. Google's plan docs say
the consumer benefits apply only inside the AI Studio web interface; API keys are
billed separately. Expect roughly **$1 a month** at one image a day.

Then set a spending guard, because a loop that misfires shouldn't be able to cost
you real money: Cloud Console → **Billing** → **Budgets & alerts** → budget of
$5/month with an alert at 50%.

Add the key to the repo as a secret named `GEMINI_API_KEY`
(*Settings → Secrets and variables → Actions → New repository secret*).

### 4 — YouTube upload credentials

In the **same** Cloud project:

1. **APIs & Services → Library** → enable **YouTube Data API v3**
2. **OAuth consent screen** → External → fill in the basics → add your own Google
   address under **Test users**
3. **Credentials → Create credentials → OAuth client ID → Desktop app** → download
   the JSON as `client_secret.json`

Then run this once on your own machine:

```bash
pip install google-auth-oauthlib
python scripts/get_youtube_token.py client_secret.json
```

A browser opens. **When the account picker appears, choose the "The Same Room"
brand channel, not your personal one.** This is the single easiest thing to get
wrong, and the symptom is videos quietly landing on the wrong channel for a week
before anyone notices.

It prints three values. Add all three as repository secrets:
`YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`.

Do not commit `client_secret.json`.

### 5 — Set the start date

In `config.json`, set `start_date` to the date you want Day 1 to be. Everything
numbers itself from there.

### 6 — Dry run

Actions tab → **The Same Room — daily** → **Run workflow** → tick
*Generate and render but do not upload*.

That spends about three cents and uploads nothing. When it finishes, download the
`render` artifact and look at the mp4. If the room looks like a room, the machine
works.

### 7 — Go live

Run it again without the dry-run tick. The video uploads **private**, because
uploads through an app that hasn't passed Google's compliance audit can only ever
be private — that's a hard restriction, not a setting.

Submit the audit from the OAuth consent screen page. It takes about two to four
weeks. When it clears, add a repository **variable** `YT_PRIVACY` set to `public`
and every future day publishes on its own.

After that the cron takes over at 07:00 Seoul time daily and you do nothing.

---

## What's enforced in code

- **The prompt is hash-locked.** `prompt.txt` is checked against the SHA-256 in
  `config.json` on every run. Edit the prompt and the run aborts rather than
  quietly contaminating the series.
- **Days are idempotent.** If `archive/day-0042.png` exists, day 42 will never be
  regenerated, no matter how many times the workflow fires.
- **There is no retry loop.** One API call, one image, it ships. A disappointing
  day is data, not a bug.

## Layout

```
prompt.txt                  the locked prompt — do not edit
config.json                 start date, model, hash, frame size
scripts/run_day.py          the daily job
scripts/get_youtube_token.py  one-time, run locally
.github/workflows/daily.yml   the cron
archive/                    every render, committed the day it was made
log.jsonl                   one line per day, with image hashes
```

## Running it by hand

```bash
pip install -r requirements.txt
SKIP_UPLOAD=1 GEMINI_API_KEY=... python scripts/run_day.py
```
