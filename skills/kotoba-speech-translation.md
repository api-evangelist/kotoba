---
name: Simultaneously translate speech with Kotoba STS
description: >-
  Open the Kotoba speech-to-speech translation channel to ingest audio in one
  language and receive both an incremental source transcript and synthesized
  target-language audio over a single WebSocket, without splitting the pipeline
  into separate ASR + MT + TTS steps.
api: asyncapi/kotoba-sts-asyncapi.yml
generated: '2026-07-19'
method: generated
source: >-
  asyncapi/kotoba-sts-asyncapi.yml,
  https://docs.kotoba.tech/overview/capabilities/speech-to-speech,
  https://docs.kotoba.tech/overview/audio-formats,
  https://docs.kotoba.tech/overview/authentication
operations:
- sts-subscribe
- sts-publish
---

# Simultaneously translate speech with Kotoba STS

- **Channel:** `/sts`
- **Client → server** frames: operation `sts-subscribe`
- **Server → client** frames: operation `sts-publish`

Supported languages: `en`, `ja`, `ko`, `zh`, `es`.

> **Host caveat.** The published AsyncAPI declares its production server as
> `wss://dummy.api.kotobatech.ai/`, which is a placeholder left in the spec.
> The ASR channel uses `wss://api.kotobatech.ai/` and TTS uses
> `wss://tts.api.kotobatech.ai/`. Confirm the real STS host with Kotoba
> (`kotoba_product@kotoba.tech`) before wiring it — do not trust the `dummy`
> hostname.

## Authenticate the handshake

STS declares **HTTP Basic** on the handshake: `device_id` as the username and
the API key as the password, sent in the `Authorization` header. The bearer and
browser client-secret flows are documented at
<https://docs.kotoba.tech/overview/authentication>; see
`authentication/kotoba-authentication.yml`.

## Frames you send

- `voice_session.update` — configure the session: `input_audio_format`
  (`pcm16`, `float32`, `twilio`, `ogg/opus`), `output_device`, source and target
  language, sample rate (default **24000 Hz**, mono).
- `input_audio_buffer.append` — Base64 audio, ≤ **1 MiB** per frame; 20–40 ms
  chunks for realtime.
- `input_audio_buffer.commit` — finalize the current utterance.

## Frames you receive

- `voice_session.created` — session is live.
- `session.capabilities` — what this session actually supports. Read this
  before assuming a language pair or output format is available.
- `voice_session.updated` — configuration accepted.
- `input_audio_buffer.committed` — commit accepted.
- `conversation.item.created` — a new conversation item.
- `text.delta` — incremental **source-language** transcript.
- `audio.delta` — synthesized **target-language** audio, streamed as produced.
  Output formats are `pcm16`, `float32`, or `twilio`.
- `error` — `{ "type": "error", "error": "<message>" }`. Fatal.

## Operating rules

- Play `audio.delta` chunks as they arrive; buffering to the end of the
  utterance throws away the whole point of simultaneous translation.
- `text.delta` is the *source* transcript, not the translation — use it for
  captions of what the speaker said, not for the translated text.
- Feed audio at wall-clock pace and commit at natural utterance boundaries.
- No idempotency and no rate-limit headers on this surface; see
  `conventions/kotoba-conventions.yml`.
