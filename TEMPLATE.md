# Using this template

This repo is a **public template** of the system described in
[README.md](README.md). The original (BNI-Masta) was built and runs in
production for one specific chapter, by 馬驊（Alex Ma）— [蟬說](https://www.chanshuo.tw/) /
[痛點科技](https://painpoint-ai.com/) — at his BNI 副主席 role in Taiwan.
This template is a sanitized snapshot you can fork and adapt to your own
chapter.

> **這是一個 template repo。** 原版 BNI-Masta 是 馬驊 為自己分會打造、長期使用中的個人系統；
> 這個 repo 是把所有跟「<YourChapter> / 馬驊個人」綁定的東西抽掉的版本，給其他想做同樣事情的
> 副主席當起點。

---

## What you get

- **Three pipelines** (Post-meeting, LinePc Computer-Use fan-out, AI News Broadcaster) — full code
- **Two-brain architecture** (chat brain + Claude wiki compiler) — runtime topology
- **5-tier fuzzy attendance match** (`resolve-attendance`)
- **31 framework files** sha256-pinned + extension contract
- **Vault structure** (Obsidian + Karpathy LLM-Wiki pattern) — empty, no member PII
- **LaunchAgent plists** + Cloudflare Tunnel config + scheduled-task definitions
- **Runbooks + architecture docs** (RUNBOOK.md, ARCHITECTURE.md, CLAUDE.md, framework-documentation-prompt.md)

## What you must replace before running

This is the meaningful work — the placeholders below MUST be filled in for your
own chapter. Search across the repo for each pattern:

| Placeholder | Meaning | Where it appears |
|---|---|---|
| `<YourChapter>` | Your BNI chapter name (e.g. `BNI 大華分會`) | README, AGENTS, vault, deck/report skills |
| `<YourName>` | Your real name | bot display name, AGENTS.md, SOUL.md |
| `<YourTestGroup>` | LINE test group display name (3-person sandbox) | ai-news + post-meeting configs |
| `<YourCommunityGroup>` | LINE 社群 display name (open community) | ai-news config |
| `<YourChapterMainGroup>` | Your chapter's main LINE group display name | ai-news + post-meeting configs |
| `<your-line-bot-id>` | LINE bot username (e.g. `@xxxxxxx`) | docs |
| `<your-telegram-alert-bot>` | Telegram alert bot username | docs, code comments |
| `<your-line-group-id>` | LINE bot-leg group ID (uppercase `C` prefix) | env, docs |
| `<your-google-sheet-id>` | Roster Sheet ID | env, skills |
| `<your-webhook-host>` | Public hostname for Cloudflare Tunnel (e.g. `bni.example.com`) | env, plists, scripts |
| `<your-google-account>` | Google account that owns the BNI calendar + Sheet | runbook, skills |
| `<your-github-org>` / `<your-github-repo>` | Where you push your fork | docs |

Quick way to find them all:

```bash
grep -rn "<Your\|<your-" .
```

## Meeting bot — Recall.ai (recommended)

This template ships wired for **[Recall.ai](https://www.recall.ai)** — a
managed meeting-bot API that joins Zoom (and Meet / Teams), records audio +
video, emits realtime participant + chat events via webhooks, and produces
speaker-diarized transcripts using its bundled streaming STT. It's the
provider the code targets out-of-the-box: `services/recall-webhook.mjs`
parses Recall.ai's `realtime_endpoints` event shape, and
`skills/zoom-join/dispatch.mjs` POSTs to `https://<region>.recall.ai/api/v1/bot/`
with the documented v1 body (`recording_config.realtime_endpoints` +
`transcript.provider.recallai_streaming`).

**Pricing:** ~$0.40–$0.50/hr/bot for typical Zoom recording + transcription.
Free tier credits cover several test meetings.
See [recall.ai/pricing](https://www.recall.ai/pricing) for current rates.

**How it integrates with the pipeline:**

| Step | Component | What happens |
|---|---|---|
| Dispatch | `zoom-join/dispatch.mjs` | POST to Recall.ai → bot joins Zoom, returns `bot.id` |
| Realtime events | `services/recall-webhook.mjs` (port 18821, behind Cloudflare Tunnel) | Receives `participant_events.*` + `transcript.data` → appends to `raw/meetings/<date>/{participants,transcript}.jsonl` |
| In-meeting chat | `services/lib/recall-chat.mjs` → Recall.ai `/send_chat_message/` | Bot posts greetings, name nudges, @-mention answers |
| Bot done detection | `meeting-poll/poll.mjs` (every 60s) | Polls Recall.ai `/api/v1/bot/<id>/`; when `status_changes[-1].code === "done"` → downloads artifacts → runs `resolve-attendance` → `ingest-claude` → reports |

If you prefer a self-hosted / open-source alternative (e.g.
[Vexa](https://github.com/Vexa-ai/vexa)), you'll need to adapt the dispatch
body shape, webhook event names, and transcript fetching to that provider's
API. The realtime-endpoints / participant_events nesting is Recall-specific.

See [SETUP.md](SETUP.md) for step-by-step Recall.ai signup, region selection,
and webhook wiring.

## Setup checklist

See [SETUP.md](SETUP.md) for the per-account walk-through (every key in
`.env.example`, where to get it, what tier to use). [RUNBOOK.md](RUNBOOK.md)
covers the host-side provisioning. Short version:

1. Clone this repo
2. Replace placeholders (table above)
3. Copy `.env.example` to `~/.openclaw/secrets/bni-masta.env`, chmod 600, fill in real keys (see [SETUP.md](SETUP.md))
4. Install OpenClaw + login the chat brain: `openclaw auth add openai-codex`
5. Login Claude CLI: `claude /login`
6. Provision Recall.ai + Cloudflare Tunnel + LINE Messaging API + Telegram bot (see [SETUP.md](SETUP.md))
7. Boot LaunchAgents: `bash scripts/install.sh`
8. Smoke-test each pipeline (RUNBOOK.md has the commands)

## License

[**Apache License 2.0**](LICENSE) — fork freely. Attribution required: keep the [NOTICE](NOTICE) file and the copyright/attribution headers in derivative works (per Apache 2.0 §4(c)(d)). Includes patent grant (§3) and trademark clause (§6 — does not grant rights to use the "BNI-Masta" name or the original creator's brands).

Attribution back to 馬驊 / [蟬說](https://www.chanshuo.tw/) / [痛點科技](https://painpoint-ai.com/) is built into [NOTICE](NOTICE) and required by §4(d) when redistributing a fork.
