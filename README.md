# Telegram Job Filter

Reads a Telegram job group, has an LLM score each post against your profile, and sends you
only the good matches.

```
Telegram group  ->  bot reads new posts  ->  LLM scores each one  ->  matches reach you on Telegram
```

You describe yourself once in `profile.md`. The bot does the scrolling.

## How it works

- It logs in to Telegram **as you** (using [Telethon](https://docs.telethon.dev)), so it can read
  public groups where you can't add a bot. It only reads. It never posts or joins anything.
- Each post goes to an LLM through [OpenRouter](https://openrouter.ai), together with your profile.
  The LLM says whether it's a real job and how well it fits (0-100).
- Posts that score at or above `MIN_SCORE` are sent to you. The rest are logged and dropped.
- Reposts are skipped, and posts that are too short are ignored.
- If the post is just a photo with a short caption, the bot reads the image too (needs a model
  with vision support).
- **No LLM? It still works.** With no `OPENROUTER_API_KEY`, or when the LLM call fails (no
  balance, bad key, outage), a keyword matcher scores the post instead. See
  [Keyword mode](#keyword-mode-no-llm).

## Requirements

- Python 3.10+
- A Telegram account that is a member of the job group
- A Telegram API ID and hash (free, from https://my.telegram.org -> API development tools)
- An OpenRouter API key (https://openrouter.ai/keys). Optional, see [Keyword mode](#keyword-mode-no-llm)
- Tesseract OCR. Optional, reads photo posts in keyword mode. See [Keyword mode](#keyword-mode-no-llm)

## Setup

```bash
# 1. Install
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 2. Create your config (only if .env does not exist yet)
cp env.example .env

# 3. Create your profile
cp profile.example.md profile.md
```

Then:

1. Fill in `.env`. The required values are `TG_API_ID`, `TG_API_HASH`, `TG_GROUP` and
   `OPENROUTER_API_KEY`. Every setting is explained in the file.
2. Fill in `profile.md`. This is what the scores are based on, so be specific about what you
   want, what you refuse, and which industries to avoid.
3. Run `.venv/bin/python bot.py`. The first run asks for your phone number and the login code
   Telegram sends you. This creates `jobbot_session.session`.

Matches arrive in your **Saved Messages**. For real push notifications, create a bot with
[@BotFather](https://t.me/BotFather), press Start on it, and set `NOTIFY_BOT_TOKEN` and
`NOTIFY_CHAT_ID` in `.env`.

## Keyword mode (no LLM)

Used when `OPENROUTER_API_KEY` is empty, and as an automatic fallback when the LLM call fails.
If the key is rejected or out of balance (HTTP 401/402/403), the bot skips the LLM for 10
minutes instead of failing on every post.

Set `INCLUDE_KEYWORDS` (roles, stack, "remote") and `EXCLUDE_KEYWORDS` in `.env`. For each post:

- It must look like an opening ("hiring", "lowongan", "loker", "dibutuhkan", requirements...).
  Job seekers ("open to work") and closed openings ("position closed") are rejected.
- Score: 1 include match = 70, 2 = 85, 3+ = 100. A job with no match scores 30.
  Broad words like `software engineer` or `javascript` also match backend posts. Keep them
  specific, add unwanted roles to `EXCLUDE_KEYWORDS`, or raise `MIN_SCORE` to 85 to need two matches.
- Title, company and how to apply are pulled out with simple patterns (email, link, `@handle`).
  They are best-effort. Company is often blank, and the "why" is just the matched words.
- Photo posts are read with [Tesseract](https://github.com/tesseract-ocr/tesseract)
  (`brew install tesseract tesseract-lang`, or `apt install tesseract-ocr tesseract-ocr-ind`).
  Without it, photo-only posts are skipped in keyword mode.

It is cruder than the LLM: no understanding of context like "no remote" or "juniors not
welcome". Matches are tagged `[keyword mode]` so you can tell which scorer decided.

## Test that delivery works

1. Set `MIN_SCORE=40` in `.env` and delete `seen.db`.
2. Start the bot. Some posts from the last `BACKFILL_HOURS` should reach you.
3. Set `MIN_SCORE` back to `70` and restart.

If nothing arrives, check the log for `Bot notify failed` (wrong chat id, or you haven't pressed
Start on your bot).

## Settings (`.env`)

| Setting | Default | What it does |
|---|---|---|
| `TG_GROUP` | none | Group to watch: `@username`, a t.me link, a numeric id, or part of the group's title |
| `TG_TOPICS` | all | For groups with topics: only read these topics (titles or ids, comma-separated) |
| `MIN_SCORE` | 70 | Minimum score to send you a post. Raise it for fewer results, lower it for more |
| `MIN_MESSAGE_LENGTH` | 60 | Posts with less text than this are skipped (unless they have a photo) |
| `BACKFILL_HOURS` | 12 | On start, also scan this many past hours. Set it above your longest expected downtime |
| `LLM_MODEL` | `anthropic/claude-haiku-5-5` | Any OpenRouter model. Use one with vision support if you want images read |
| `NOTIFY_BOT_TOKEN`, `NOTIFY_CHAT_ID` | empty | Send matches from your own bot instead of Saved Messages |

## Keep it running

The bot only works while it is running. Use an always-on machine or a small VPS.

- Quick start: `tmux new -s jobbot`, run the bot, then press `Ctrl+b d` to detach.
- Better: a systemd service (see below), so it survives crashes and reboots.

Keep `seen.db` on disk. It's how the bot remembers what it already handled.

## Deploy on Ubuntu (systemd)

Use a normal user, not root. Stop the copy on your laptop first, because two copies must never
run with the same Telegram login.

```bash
# 1. Get the code and install
git clone https://github.com/rizkiananto/telegram-group-ai-reader.git auto-read-job-group
cd auto-read-job-group
sudo apt install -y python3-venv
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 2. Create your config on the server (these files are not in git)
cp env.example .env && cp profile.example.md profile.md
nano .env && nano profile.md
chmod 600 .env

# 3. Log in to Telegram once, by hand (phone number + code), then stop with Ctrl+C
.venv/bin/python bot.py

# 4. Install the service
nano jobbot.service        # replace YOUR_USER and the paths
sudo cp jobbot.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now jobbot
```

Day to day:

```bash
systemctl status jobbot             # is it running?
journalctl -u jobbot -f             # live logs
git pull && sudo systemctl restart jobbot   # update
```

Do step 3 before step 4. The service can't type your login code, so without it the bot crashes
and systemd gives up after 5 tries.

**Log in again on the server.** Don't copy `jobbot_session.session` from your laptop. Telegram may
invalidate a session used from two places. If the server's login ever breaks, delete the session
file there and repeat step 3.

If the server is offline for a while, the bot catches up on the last `BACKFILL_HOURS` hours when
it returns. Set that above your longest expected downtime.

## Security

- `jobbot_session.session` is a full login to your Telegram account. Anyone who has it can read
  your chats and send messages as you. Keep it private.
- `.env` holds your API keys. Never commit it.
- Add these to `.gitignore` before your first commit: `.env`, `*.session`, `*.session-journal`,
  `seen.db`, `profile.md`, `.venv/`, `__pycache__/`
- On a server: `chmod 600 .env jobbot_session.session`, use SSH keys, and don't run the bot as root.
- `profile.md` is sent to OpenRouter and the model provider with every post. Don't put your phone
  number, email or address in it.
- Group posts are treated as untrusted. The prompt tells the model to evaluate them, not obey them.

## Costs

The Telegram side is free. The LLM cost is one call per new post that passes the length check,
plus one image per photo-only post. With a cheap model this is small. Set a spending limit on your
OpenRouter key.

## Limitations

- It uses a normal user account through the API. Read-only use is low risk, but it's still
  automation of a user account, so use it at your own risk.
- If sending to Telegram fails, the post is still marked as seen and won't be resent.
- Only photos are read, not PDFs or other attached files.
- Links inside posts are not opened. Only the post text and its photo are judged.
- Scores come from an LLM. Fix bad scores by making `profile.md` more specific.
