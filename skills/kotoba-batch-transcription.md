---
name: Transcribe an audio file with Kotoba (batch REST)
description: >-
  Submit a finished audio file to Kotoba's transcription API and poll until the
  job completes, optionally with per-segment timestamps. Use this instead of the
  WebSocket channel when you have a file on disk and do not need partial results.
api: openapi/kotoba-transcription-openapi-original.yml
generated: '2026-07-19'
method: generated
source: >-
  openapi/kotoba-transcription-openapi-original.yml,
  https://docs.kotoba.tech/overview/capabilities/speech-to-text,
  https://docs.kotoba.tech/s2t/python-sdk
operations:
- submit-transcription-job-v-1-transcription-jobs-post
- get-transcription-job-v-1-transcription-jobs-job-id-get
---

# Transcribe an audio file with Kotoba (batch REST)

Kotoba's batch path is a two-call submit-and-poll job model on
`https://api.kotobatech.ai`. Access is **private alpha** — the host answers
`503` unless your key is on the allowlist. Request access from
`kotoba_product@kotoba.tech`.

## Authenticate

Send the API key as a bearer token on every request:

```
Authorization: Bearer $KOTOBA_API_KEY
```

Server-side only. Never put a long-lived Kotoba key in browser code. The Python
SDK (`pip install kotoba-sdk`) reads `KOTOBA_API_KEY` from the environment.

## Step 1 — submit the file

Call `submit-transcription-job-v-1-transcription-jobs-post`
(`POST /v1/transcription_jobs`) as `multipart/form-data`:

| Field | Required | Notes |
| --- | --- | --- |
| `file` | yes | Audio file (MP3/WAV/MP4/WebM/etc.) |
| `language` | no | ISO-639-1, one of `en`, `ja`, `ko`, `zh`. Defaults to `ja` |
| `with_timestamps` | no | `false` by default. Set `true` to get `segments[]` |

A success returns **202** with `{ "job_id": "..." }`.

Only set `with_timestamps=true` when you actually need segment boundaries — it
adds tokenizer plus silero-VAD work on top of the lighter text-only path.

## Step 2 — poll for the result

Call `get-transcription-job-v-1-transcription-jobs-job-id-get`
(`GET /v1/transcription_jobs/{job_id}`) with the `job_id` from step 1.

**This endpoint returns HTTP 200 for both success and failure.** Branch on the
`state` discriminator, never on the status code:

- `state: "done"` → read `transcription` (the full text) and, when requested,
  `segments[]` where each entry is `{ text, start, end }` in seconds.
- `state: "error"` → read `error_message`.

Poll with backoff until `state` is one of those two terminal values.

## Error handling

- **422** — validation failure. The body is *not* RFC 9457 problem+json; it is
  `{ "detail": [ { "loc": [...], "msg": "...", "type": "..." } ] }`. `loc`
  locates the offending field. Most commonly a missing `file` part or a
  `language` outside `en`/`ja`/`ko`/`zh`.
- **503** — the private-alpha host is refusing unauthenticated or
  non-allowlisted traffic. This is an access problem, not a request problem.
- Full catalog: `errors/kotoba-problem-types.yml`.

## Retry rules

There is **no idempotency key** on this API. `POST /v1/transcription_jobs`
creates a new job on every call, so a blind retry of a submission that actually
succeeded will transcribe the file twice and bill twice. Retry the submit only
when you have no `job_id`; retry the poll freely, since `GET` is safe.

There is also no completion webhook — polling is the only way to learn that a
job finished.

## Related

- Conventions: `conventions/kotoba-conventions.yml`
- Live streaming alternative: `kotoba-live-transcription.md`
