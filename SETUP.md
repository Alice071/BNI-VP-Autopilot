# SETUP — Accounts & APIs you'll need

This is the friendly walk-through of **every external account, subscription,
and API key** that the BNI-VP-Autopilot framework needs to run end-to-end.
It's the prerequisite to [RUNBOOK.md](RUNBOOK.md), which covers the
host-side install on a fresh Mac. Read this one first, then RUNBOOK.

> Most accounts here have a free tier or a generous trial — typical monthly
> running cost for a single chapter (4 Friday meetings + a few PDFs + voice
> notes) is **~$15–17/month**, of which $10 is Obsidian Sync (optional).
> See the cost summary at the bottom.

---

## 1. Required (the framework cannot function without these)

### 1.1 Recall.ai — meeting bot for Zoom

**What it does:** dispatches a participant bot to your Zoom meeting,
records audio + video, fires realtime webhooks for participant join/leave,
chat messages, and transcript chunks. Bundled streaming STT (no separate
transcription provider needed). The post-meeting pipeline is anchored on
Recall.ai's `bot.done` state.

**Sign up:** [recall.ai](https://www.recall.ai) → create account → Dashboard
→ **API Keys**.

**What to copy into your `.env`:**

| Env var | Value |
|---|---|
| `RECALL_API_KEY` | The API token shown in the dashboard. NOT the `whsec_…` webhook signing secret — that one is for account-level webhooks, which this framework doesn't use. |
| `RECALL_REGION` | The region your account was provisioned in. Check the dashboard or the API base URL Recall.ai shows you. Common values: `ap-northeast-1` (Tokyo, default for APAC), `us-west-2`, `us-east-1`. |
| `RECALL_WEBHOOK_URL` | Your public webhook URL — set this **after** Cloudflare Tunnel is up (§1.6). Format: `https://<your-subdomain>.<your-domain>/recall-webhook`. |
| `RECALL_WEBHOOK_TOKEN` | Optional. If set, the webhook listener requires `?token=<this>` on every POST — extra defense-in-depth on top of the unguessable hostname. The dispatcher appends it automatically. |

**Pricing:** ~$0.40–$0.50/hr/bot for typical Zoom recording + bundled
streaming STT. Free tier covers initial testing (a handful of meetings).
Current rates at [recall.ai/pricing](https://www.recall.ai/pricing).

**How the framework uses it:**

| File | Calls Recall.ai for |
|---|---|
| [openclaw/agents/bni-masta/skills/zoom-join/dispatch.mjs](openclaw/agents/bni-masta/skills/zoom-join/dispatch.mjs) | `POST /api/v1/bot/` → spawns the bot |
| [services/recall-webhook.mjs](services/recall-webhook.mjs) | Receives realtime `participant_events.*` + `transcript.data` |
| [services/lib/recall-chat.mjs](services/lib/recall-chat.mjs) | `POST /api/v1/bot/<id>/send_chat_message/` (in-meeting chat) |
| [services/lib/meeting-handlers.mjs](services/lib/meeting-handlers.mjs) | `GET /api/v1/bot/<id>/` (snapshot existing participants) |
| [openclaw/agents/bni-masta/skills/meeting-poll/poll.mjs](openclaw/agents/bni-masta/skills/meeting-poll/poll.mjs) | `GET /api/v1/bot/<id>/` (poll for `status_changes[-1].code === "done"`) |

If you want to swap providers (e.g. self-hosted [Vexa](https://github.com/Vexa-ai/vexa)),
those are the five files you'd adapt — the dispatch body shape, webhook
event names, and transcript-fetching endpoint are Recall.ai-specific.

### 1.2 OpenRouter — chat brain + transcription + OCR

**What it does:** unified API gateway that gives the framework access to
multiple frontier models with one key:

- **Claude Haiku 4.5** for in-meeting chat replies + post-meeting reports
  (~$0.18/Friday meeting on Haiku)
- **Gemini 2.5 Flash** for OCR (PDF handbook ingest) + audio transcription
  for voice notes (the operator sends to the bot)

**Sign up:** [openrouter.ai/keys](https://openrouter.ai/keys) → create key.

**Env var:** `OPENROUTER_API_KEY` (starts with `sk-or-v1-…`)

**Pricing:** pay-as-you-go, charged in OpenRouter credits. Realistic month:
$1–3 total across all uses.

> **Important:** All Claude/Haiku calls in this framework route through
> OpenRouter, **not** the Anthropic SDK directly. This is a deliberate
> convention — the AI News Broadcaster extension (deck.mjs), in-meeting
> chat responder, and post-meeting reports all use the OpenRouter
> `https://openrouter.ai/api/v1/chat/completions` endpoint with model id
> `anthropic/claude-haiku-4.5`.

### 1.3 Anthropic API — wiki compiler brain (via the `claude` CLI)

**What it does:** the wiki compiler (`claude` CLI invoked by
`ingest-claude` skill) reads `raw/` and rewrites `wiki/` markdown pages.
Distinct from the chat brain — runs on-demand, not per-message.

**Two ways to provide it:**

| Option | How |
|---|---|
| **Claude Pro/Max subscription** (recommended if you already have it) | `claude /login` once; the CLI uses your subscription quota. **No env var needed.** |
| **Anthropic API key** | Set `ANTHROPIC_API_KEY=sk-ant-…` in `.env`. Get one at [console.anthropic.com](https://console.anthropic.com). Pay-per-token; usually <$1/month for a chapter. |

If both are present, the CLI subscription wins.

### 1.4 Telegram — operator alert + DM control channel

**What it does:** the bot you DM commands to (`/zoom-join`, `/pdf-ingest`,
`/ingest-claude`, etc.) and that pushes you operator-only alerts (post-
meeting digests, escalations from scheduled tasks, error notifications).

**Sign up:**
1. Open `@BotFather` in Telegram
2. `/newbot`, pick a name + username
3. BotFather replies with the bot token

**Env vars:**

| Env var | How to get it |
|---|---|
| `BNI_BOT_TOKEN` | The token BotFather gave you, format `<digits>:<base64>` |
| `OPERATOR_TELEGRAM_ID` | Your own Telegram chat_id. Easy way: DM [@userinfobot](https://t.me/userinfobot), it replies with your `Id`. |

**Pricing:** free.

### 1.5 LINE Messaging API — Taiwan-side member channel

**What it does:** broadcast post-meeting attendance, AI news digests, and
1:1 chat replies to chapter members on LINE (the dominant chat platform
in Taiwan).

**Sign up:** [LINE Developers Console](https://developers.line.biz) → create
provider → create **Messaging API** channel.

**Env vars:**

| Env var | Where in the console |
|---|---|
| `LINE_CHANNEL_SECRET` | Channel basic settings → Channel secret |
| `LINE_CHANNEL_ACCESS_TOKEN` | Messaging API → Channel access token (long-lived) |
| `OPERATOR_LINE_ID` | Your own LINE userId (`U` + 32 hex chars). Captured automatically the first time you DM the bot — check the gateway log, then paste into env. |
| `LINE_TARGET_GROUP_IDS` | Comma-separated group IDs (uppercase `C` prefix) that the bot is in and should broadcast to. Captured the first time the bot sees a message in that group. |

**Pricing:** free up to 500 push messages/month — plenty for a single
chapter. See [LINE plans](https://www.linebiz.com/jp-en/service/line-account-connect/plan/).

> **Note on LINE's 1-OA-per-group rule:** LINE allows only ONE Messaging
> API bot account per group. If your bot can't enter a group (e.g. another
> bot is already there, or it's a cross-chapter VP forum), the framework
> falls back to **Pipeline #2 — LinePc**, which uses the operator's
> *personal* LINE account via Computer Use over LINE for Mac. That path
> needs no extra credentials but does need Claude Desktop installed on
> the operator's Mac.

### 1.6 Cloudflare — public webhook tunnel

**What it does:** exposes your Mac's `127.0.0.1:18821` (Recall.ai webhook
listener) and `127.0.0.1:18822` (lion-avatar HTML server) at a public
HTTPS URL via a [Cloudflare named tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).
No port-forwarding, no public IP, no NAT pain.

**Prerequisites:**

- A free Cloudflare account
- A domain you control (any TLD; you don't have to host the site there —
  just have Cloudflare manage DNS)

**One-time setup (browser flow):**

```bash
brew install cloudflared
cloudflared tunnel login            # browser, picks your CF account
cloudflared tunnel create bni-webhook
cloudflared tunnel route dns bni-webhook bni-webhook.<your-domain>.com
# Edit ~/.cloudflared/config-bni.yml — paste the tunnel UUID + hostname
```

The repo ships [scripts/cloudflared-config-bni.yml](scripts/cloudflared-config-bni.yml) as a
template (uses `<your-tunnel-uuid>` and `<your-webhook-host>` placeholders).

**Env vars:** none directly — but the webhook URL you set as
`RECALL_WEBHOOK_URL` must point at your tunnel hostname.

**Pricing:** free (Cloudflare named tunnels have no usage limits for
this kind of traffic).

### 1.7 ChatGPT subscription — legacy chat-brain fallback

**What it does:** the framework's *default* chat brain is now Claude
Haiku 4.5 via OpenRouter (§1.2). The original architecture used GPT-5.4
via OpenAI Codex OAuth as the chat brain, and it's still wired as a
fallback when `BNI_USE_OPENCLAW=1` is set.

**Sign up:** if you have ChatGPT Plus / Pro / Team, you can authenticate
the legacy fallback for free via OAuth:

```bash
openclaw models auth login --provider openai-codex
```

**Env vars:** none — handled by `~/.openclaw/auth-profiles/openai-codex.json`.

**Pricing:** included in your existing ChatGPT subscription. Skip this
if you only use the Haiku-via-OpenRouter default.

### 1.8 Google account — Calendar + Sheet + Drive

**What it does:**

- **Google Sheet** — chapter roster + traffic-light scores + 出席紀錄
  (attendance log). Sheet structure: 3 tabs (`<YourChapter>會員名單`,
  `紅綠燈`, `出席紀錄`) — see [openclaw/agents/bni-masta/skills/roster-sync/SKILL.md](openclaw/agents/bni-masta/skills/roster-sync/SKILL.md)
  for layout.
- **Google Calendar** — dedicated "BNI" calendar for meetings + 1-to-1s
  + 封閉會議.
- **Google Drive** — `BNI-Masta-Reports/` folder where rendered meeting-
  deck PDFs are uploaded with `--share=anyone-reader` for distribution.

**Sign up:** any Google account works. Create a fresh dedicated calendar
(don't reuse your personal one) so the agent doesn't see private events.

**One-time OAuth flow** (uses the bundled `gog` CLI, installed by `scripts/install.sh`):

```bash
# 1. Create an OAuth client in Google Cloud Console (Desktop app type).
#    Download client_secret.json.
gog auth credentials ~/path/to/client_secret.json

# 2. Browser-authorize on the Google account that owns the BNI Sheet/Calendar.
gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets
```

**Env vars:**

| Env var | What |
|---|---|
| `BNI_CALENDAR_ID` | The Calendar ID of your dedicated BNI calendar (looks like `c_…@group.calendar.google.com`). Find it in Google Calendar → calendar settings → "Integrate calendar". |
| `BNI_ROSTER_SHEET_ID` | The Sheet ID from the URL `docs.google.com/spreadsheets/d/<THIS>/edit`. |

**Pricing:** free under personal quotas.

---

## 2. Optional / per-feature

### 2.1 Apify — AI News Broadcaster Pipeline (#3) only

**What it does:** scrapes a curated list of Facebook AI-news pages on a
72-hour rolling window, feeds the results to OpenRouter Haiku for
curation + zh-TW translation, builds a 6-page PDF, and broadcasts it.

**Skip this if:** you're not running the AI News Broadcaster extension.
The extension lives at `openclaw/agents/bni-masta/extensions/ai-news-broadcaster/`
and is strictly additive — the rest of the framework runs without it.

**Sign up:** [apify.com](https://apify.com) → create account → get token
from Settings → Integrations.

**Env var:** `APIFY_TOKEN`.

**Pricing:** $5 free credit on signup, generous free tier. Scraping ~10
FB pages every 2 days uses pennies of compute.

### 2.2 Obsidian Sync — vault sync across devices

**What it does:** end-to-end-encrypted sync of the Obsidian vault
between your Mac (where the agent runs) and your iPhone / iPad / second
laptop, so you can browse the LLM-Wiki anywhere.

**Skip this if:** you only need to read the vault on the Mac the agent
runs on. The framework writes to `~/Documents/BNI AGENT/BNI AGENT/`
locally regardless.

**Sign up:** inside Obsidian → Settings → Sync.

**Pricing:** $10/month per Obsidian account (covers all your vaults +
all your devices).

---

## 3. Provisioning order (recommended)

Some accounts depend on others — here's a sane order:

1. **Telegram** (5 min) — needed for the smoke test at the end
2. **Cloudflare** (10 min) — domain DNS, then `cloudflared tunnel`
   commands. Only after this can you finalize `RECALL_WEBHOOK_URL`.
3. **OpenRouter** (2 min) — single key, paste, done
4. **Recall.ai** (5 min) — sign up, copy API key + region, set
   `RECALL_WEBHOOK_URL` to your CF Tunnel hostname
5. **LINE Developers** (15 min) — Messaging API channel, copy secret +
   token. The bot's `userId` capture happens later when you first DM it.
6. **Google** (15 min) — create dedicated calendar, create roster Sheet,
   run `gog auth credentials` + `gog auth add` once
7. **Anthropic** (2 min) — `claude /login` (uses Pro/Max subscription) OR
   create API key in console
8. **Apify** (2 min, optional) — only if running AI News Broadcaster
9. **Obsidian Sync** (2 min, optional) — only if you want vault on phone

After all 9 steps your `~/.openclaw/secrets/bni-masta.env` is fully
populated and `bash scripts/install.sh` should run cleanly. See
[RUNBOOK.md](RUNBOOK.md) for the install steps.

---

## 4. Cost summary

Approximate monthly cost for one chapter (4 × 90-min Friday meetings +
~10 PDF ingests + ~30 voice notes), assuming you already have a
ChatGPT Plus / Claude Pro subscription:

| Item | Estimate |
|---|---|
| Recall.ai (4 × 90-min meetings, ~6h total) | $2.40 – $3.00 |
| OpenRouter (Haiku chat replies) | ~$0.72 |
| OpenRouter (Gemini Flash OCR) | ~$1 |
| OpenRouter (Gemini Flash audio transcribe) | ~$1 |
| Anthropic API (wiki compiler, if not using Pro/Max) | <$1 |
| Apify (AI News Broadcaster, if enabled) | <$1 |
| Telegram / LINE / Cloudflare / Google APIs | $0 |
| Obsidian Sync (optional) | $10 |
| **Total** | **~$6 (no Obsidian Sync) – ~$17 (with everything)** |

ChatGPT Plus / Claude Pro subscriptions are sunk cost (you're on them
for other reasons), so they don't count toward the marginal monthly cost.

---

## 5. Where each key actually goes

Two files hold all the secrets — both should be `chmod 600` and never
committed:

| File | Holds | Generated by |
|---|---|---|
| `~/.openclaw/secrets/bni-masta.env` | Every `*_API_KEY`, `*_TOKEN`, ID listed above | You, manually copying from `.env.example` |
| `~/.openclaw/openclaw.json` | Telegram bot token + LINE Messaging API channel + gateway routing rules | Generated by `scripts/install.sh` from `openclaw/openclaw.json.template` substituting env vars |

OAuth profiles (also `chmod 600`, never commit):

| File | Generated by |
|---|---|
| `~/.openclaw/auth-profiles/openai-codex.json` | `openclaw models auth login --provider openai-codex` (browser) |
| `~/.openclaw/auth-profiles/gog.json` | `gog auth credentials` + `gog auth add` (browser) |
| `~/.cloudflared/<tunnel-uuid>.json` | `cloudflared tunnel login` + `cloudflared tunnel create …` (browser) |

If your `.env` and these four files are in place, the framework boots.
