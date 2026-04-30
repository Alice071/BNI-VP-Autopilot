# BNI-Masta Runbook — install on a fresh Mac

## Prerequisites

- macOS with Homebrew installed
- A ChatGPT Plus/Pro/Team subscription (for Codex OAuth)
- A Cloudflare account + a domain you control (for named tunnel)
- An OpenRouter account (for transcription)
- A Recall.ai account (for Zoom bot joining + realtime events + transcript)
- A Google account (for `gog` Calendar / Sheets / Drive integration)
- Node 20+ (`brew install node`), Python 3.9+

A friendly walk-through of every account + API + plan tier is in
[SETUP.md](SETUP.md). Read that first if this is your first install.

## One-shot install

```bash
git clone git@github.com:<your-org>/BNI-VP-Autopilot.git
cd bni-masta
cp .env.example ~/.openclaw/secrets/bni-masta.env
chmod 600 ~/.openclaw/secrets/bni-masta.env
# Fill in the env file with your API keys (see .env.example for which fields,
# and SETUP.md for where each one comes from)
bash scripts/install.sh
```

`install.sh` does:

1. `brew install` — openclaw, cloudflared, ngrok, poppler, ffmpeg, jq, ocrmypdf, tesseract-lang, gh, steipete/tap/gogcli, yakitrak/yakitrak/obsidian-cli, uv
2. `uv tool install nano-pdf`
3. `openclaw onboard` if not already onboarded
4. `openclaw models auth login --provider openai-codex` (browser OAuth for GPT-5.4)
5. Creates dirs: `~/.openclaw/agents/bni-masta/`, `~/.openclaw/secrets/`
6. Copies this repo's `openclaw/agents/bni-masta/*` into `~/.openclaw/agents/bni-masta/`
7. Copies this repo's `vault/*` into `~/Documents/BNI AGENT/BNI AGENT/` (merges — doesn't overwrite existing `raw/` or `wiki/`)
8. Applies `openclaw/openclaw.json.template` → substitutes placeholders from the env file → writes to `~/.openclaw/openclaw.json` (backs up the existing one first)
9. Installs LaunchAgents:
   - `ai.bnimasta.recall-webhook.plist`
   - `com.cloudflare.bni-webhook-tunnel.plist`
10. `launchctl load` both
11. Restarts the OpenClaw gateway

## Manual steps (can't be scripted)

1. **Cloudflare named tunnel** — `cloudflared tunnel login`, then `cloudflared tunnel create bni-webhook`, then `cloudflared tunnel route dns bni-webhook <your-hostname>`. Update `~/.cloudflared/config-bni.yml` with the tunnel UUID.
2. **Google OAuth for `gog`** — once, browser: `gog auth credentials ~/path/to/client_secret.json && gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets`.
3. **Dedicated BNI Google Calendar** — create it at https://calendar.google.com → copy Calendar ID → paste into env as `BNI_CALENDAR_ID`.
4. **Recall.ai signup** — sign up at [recall.ai](https://www.recall.ai), grab the API key from the dashboard (the token itself, NOT the webhook signing secret). Note your region — most APAC accounts land on `ap-northeast-1`, US accounts on `us-west-2` or `us-east-1`. Set both `RECALL_API_KEY` and `RECALL_REGION` in env.
5. **Telegram bot** — create with `@BotFather` → copy token → paste into env.
6. **LINE bot** — create in LINE Developers Console → copy channelSecret + channelAccessToken → paste into env.
7. **Obsidian plugins** — install Dataview, Templater, Tasks, Calendar community plugins (see top-level OBSIDIAN-SETUP.md).
8. **OCR tooling for scanned PDFs** — `brew install ocrmypdf tesseract-lang` already done by install.sh. Chinese language pack installed.

## Smoke test

```bash
# Telegram bot alive?
curl -s "https://api.telegram.org/bot$BNI_BOT_TOKEN/getMe" | jq .result.username
# expected: "Bnimasta_bot"

# Webhook tunnel reachable?
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$RECALL_WEBHOOK_URL" \
  -H "content-type: application/json" -d '{"event":"ping"}'
# expected: 200

# Channel status + agent binding
openclaw channels status --probe
openclaw agents list
# expected: bni-masta listed with 2 routing rules

# Try a compile
echo "test" > "<vault-path>/raw/inbox/hello.md"
bash ~/.openclaw/agents/bni-masta/agent/skills/ingest-claude/compile.sh raw/inbox
```

## Restart services

```bash
# gateway
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway

# webhook
launchctl kickstart -k gui/$(id -u)/ai.bnimasta.recall-webhook

# cloudflared
launchctl kickstart -k gui/$(id -u)/com.cloudflare.bni-webhook-tunnel
```
