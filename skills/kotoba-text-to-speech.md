---
name: Synthesize streaming speech with Kotoba TTS
description: >-
  Open the Kotoba text-to-speech WebSocket channel and stream synthesized audio
  back as soon as the model produces it, so you can pipe straight to a speaker,
  a WebRTC track, or an LLM-driven voice agent without waiting for the full
  utterance.
api: asyncapi/kotoba-tts-asyncapi.yml
generated: '2026-07-19'
method: generated
source: >-
  asyncapi/kotoba-tts-asyncapi.yml,
  https://docs.kotoba.tech/overview/capabilities/text-to-speech,
  https://docs.kotoba.tech/overview/audio-formats,
  https://docs.kotoba.tech/overview/authentication
operations:
- tts-subscribe
- tts-publish
---

# Synthesize streaming speech with Kotoba TTS

- **Channel:** `wss://tts.api.kotobatech.ai/v2/tts/ws`
- **Client → server** frames: operation `tts-subscribe`
- **Server → client** frames: operation `tts-publish`

Authenticate the handshake with `Authorization: Bearer $KOTOBA_API_KEY`.
Supported languages: `en`, `ja`, `ko`, `zh`, `es`. Available Japanese speakers:
`ja-man-m02-azawa` (male) and `ja-woman-f04-me` (female).

## Frames you send

- `open_session` — open the synthesis session; set language, speaker, and
  output format (`pcm16`, `float32`, or `twilio`).
- `response.create` — request synthesis of a piece of text. Each response is a
  distinct lifecycle.
- `response.cancel` — abandon the in-flight response. Use this the moment a
  user barges in on a voice agent; do not just stop reading the socket.

## Frames you receive

- `session.created` — session is live; carries the negotiated audio format.
- `response.created` — synthesis has started for your request.
- `audio_chunk` — Base64 audio, streamed as produced. Play these immediately.
- `response.done` — terminal frame carrying `status`, one of **`completed`**,
  **`cancelled`**, or **`failed`**. On `failed`, the nested `error` object
  (`{ code, message }`) is populated. Always branch on `status`; a `response.done`
  is not by itself a success.
- `timeout` — **non-fatal.** The synthesis worker has not produced an audio
  chunk within the configured window. The session stays open and the server will
  either resume streaming or escalate to a terminal frame. Do **not** tear down
  the connection on this frame.
- `error` — `{ "type": "error", "message": "...", "code": "..." }`. Fatal
  server-side error; the session is over.

## Operating rules

- Treat `timeout` and `error` as genuinely different: only `error` and
  `response.done{status:failed}` are terminal.
- One response at a time per session — wait for `response.done` (or send
  `response.cancel`) before issuing the next `response.create`.
- Output audio defaults to **24000 Hz** mono unless the session negotiates
  otherwise; `twilio` output is μ-law 8 kHz mono for Twilio Media Streams.
- The Python SDK exposes this as `client.tts.stream(...)` for streaming and
  `client.tts.synthesize(...)` for one-shot, with `AsyncKotobaClient` for
  concurrency (`pip install kotoba-sdk`).
- Error catalog: `errors/kotoba-problem-types.yml`.
