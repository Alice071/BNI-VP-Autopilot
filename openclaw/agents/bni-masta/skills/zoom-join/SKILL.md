---
name: zoom-join
description: Send a Recall.ai bot named "BNI-Masta" to join a Zoom meeting and capture audio, video, participant events, and transcript. Use when the operator is attending a meeting hosted by someone else.
metadata:
  openclaw:
    emoji: "🤖"
    requires:
      env: [RECALL_API_KEY]
    triggers:
      - "/zoom-join <url_or_id> <pwd>"
      - "the operator pastes a Zoom link"
---

# zoom-join

Dispatches a [Recall.ai](https://www.recall.ai) participant bot to a Zoom meeting. Recall.ai handles the hard parts: joining as a participant, recording audio+video, producing speaker-diarized transcripts, emitting participant join/leave events. Webhooks fire at `~/.openclaw/agents/bni-masta/services/recall-webhook.mjs` which writes raw files and auto-triggers `resolve-attendance` + `ingest-claude`.

## Inputs

- `meeting_url` — full Zoom link OR 11-digit meeting ID
- `meeting_password` — optional if the link embeds pwd, required if standalone ID
- `scheduled_start` (optional ISO) — if the meeting is in the future; defaults to now
- `meeting_title` (optional) — e.g., "2026-04-22 封閉會議"; defaults to `今日會議 <YYYY-MM-DD>`

## Behavior

1. Normalize: if the operator pasted a `https://zoom.us/j/12345?pwd=...` URL, extract pwd from it.
2. POST to `https://<region>.recall.ai/api/v1/bot/` (region from `RECALL_REGION`, default `ap-northeast-1`; use `us-west-2` if your account is on US West) with:
   ```json
   {
     "meeting_url": "https://zoom.us/j/12345?pwd=...",
     "bot_name": "BNI-Masta",
     "recording_config": {
       "transcript": { "provider": { "recallai_streaming": {} } },
       "video_mixed_layout": "gallery_view_v2",
       "realtime_endpoints": [{
         "type": "webhook",
         "url": "https://<your-webhook-host>/recall-webhook",
         "events": [
           "participant_events.join", "participant_events.leave",
           "participant_events.update", "participant_events.speech_on",
           "participant_events.speech_off", "participant_events.webcam_on",
           "participant_events.webcam_off", "participant_events.chat_message",
           "transcript.data"
         ]
       }]
     }
   }
   ```
3. Save the returned `bot.id` to `raw/meetings/<date>/<bot_id>.bot.json` so webhooks can be correlated.
4. Emit phase line per SOUL: `✓ bot dispatched · waiting on meeting` — no future-tense "I'll ping you" preamble. The webhook will fire its own `✓ transcript ready` line when Recall.ai returns data.

## Notes

- Recall.ai pricing is region-specific; budget ~$0.40–$0.50/hr/bot. Free tier credits cover initial testing — see [recall.ai/pricing](https://www.recall.ai/pricing).
- `webhook_url` must be public — for dev use `ngrok http 18821` and update this value; for prod use Cloudflare Tunnel.
- Prod: Cloudflare Tunnel to `~/.openclaw/agents/bni-masta/services/recall-webhook.mjs` on port 18821.

## Implementation

Script: `./dispatch.mjs`. Run via `node dispatch.mjs <url> [pwd] [title]`.
