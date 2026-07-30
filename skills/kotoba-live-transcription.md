---
name: Stream live speech-to-text with Kotoba ASR
description: >-
  Open the Kotoba ASR WebSocket channel, push audio chunks as they arrive, and
  read transcription deltas back on the same connection. Use for microphones and
  any pipeline where the first words matter before the utterance ends.
api: asyncapi/kotoba-asr-asyncapi.yml
generated: '2026-07-19'
method: generated
source: >-
  asyncapi/kotoba-asr-asyncapi.yml,
  https://docs.kotoba.tech/overview/capabilities/speech-to-text,
  https://docs.kotoba.tech/overview/audio-formats,
  https://docs.kotoba.tech/overview/authentication
operations:
- asr-subscribe
- asr-publish
---

# Stream live speech-to-text with Kotoba ASR

One WebSocket connection carries the whole session as JSON frames.

- **Channel:** `wss://api.kotobatech.ai/v1/realtime`
- **Client → server** frames: operation `asr-subscribe`
- **Server → client** frames: operation `asr-publish`

Private alpha — request access from `kotoba_product@kotoba.tech`.

## Authenticate the handshake

Server-side (recommended): set `Authorization: Bearer $KOTOBA_API_KEY` on the
WebSocket handshake request.

Browser: you cannot set that header. Mint a short-lived client secret from your
backend via `POST https://api.kotobatech.ai/v1/realtime/transcription_sessions`,
hand it to the page, and open the socket with:

```
Sec-WebSocket-Protocol: realtime, kotoba-insecure-api-key.<CLIENT_SECRET>
```

There is no browser SDK yet — drive the socket directly.

## Frames you send

- `transcription_session.update` — configure the session. Set
  `input_audio_format` (`pcm16`, `float32`, `twilio`, or `ogg/opus`), sample
  rate, and channel count. Defaults are **24000 Hz, mono**.
- `input_audio_buffer.append` — one chunk of Base64-encoded audio in the event
  body. Cap each frame at **1 MiB** (~10 s at the default rate); **20–40 ms
  chunks** are the right size for realtime.
- `input_audio_buffer.commit` — close the current utterance and ask the server
  to finalize it.

## Frames you receive

- `transcription_session.created` — session is live. Correlate everything after
  this against the session object; there is no request-id header on this API.
- `transcription_session.updated` — your configuration was accepted.
- `input_audio_buffer.committed` — the commit was accepted.
- `conversation.item.created` — a new item entered the conversation.
- `transcription.delta` — **incremental transcript text.** Append these to
  render live captions.
- `transcription.completed` — the finalized transcript for the committed audio.
  Replace your accumulated deltas with this text rather than concatenating.
- `error` — `{ "type": "error", "error": "<message>" }`. Fatal; the session is
  over. Reconnect and resume from the last committed utterance.

## Language

Input languages are `en`, `ja`, `ko`, `zh` (ISO-639-1). The default is `ja`.

## Operating rules

- Send audio at roughly wall-clock pace. Bursting a long file through the
  realtime channel wastes the streaming path — use the batch REST job instead
  (`kotoba-batch-transcription.md`).
- Always `commit` before you expect a `transcription.completed`.
- There is no rate-limit header to read; throttling is enforced at the account
  level by the alpha allowlist.
- Error catalog: `errors/kotoba-problem-types.yml`. Conventions:
  `conventions/kotoba-conventions.yml`.
