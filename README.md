# Voice-AI RAG Backend

A multi-tenant voice agent: speak, and the answer comes back as speech, grounded
in that tenant's own documents — with barge-in, so you can interrupt it.

Browser → WebSocket → speech-to-text → hybrid retrieval → LLM → sentence-level
TTS → audio, in one FastAPI service, with pgvector, Redis and Postgres doing the
storage and rate limiting. Everything runs locally on CPU with no API keys, so
`docker compose up` is a working voice backend.

**Status: complete.** The pipeline, the demo page and the evaluation harness
work end to end. Deferred work is listed honestly in
[Known limitations](#known-limitations) and in `PROGRESS.md` — the important
ones are speech-onset barge-in on the local stack, asynchronous document
ingestion, row-level security, a real OpenTelemetry pipeline and a live cloud
deployment.

---

## Contents

- [Voice-AI RAG Backend](#voice-ai-rag-backend)
  - [Contents](#contents)
  - [Quick start](#quick-start)
  - [Try the demo](#try-the-demo)
  - [How a call flows](#how-a-call-flows)
  - [API](#api)
  - [RAG](#rag)
  - [Configuration](#configuration)
  - [Local mode vs cloud](#local-mode-vs-cloud)
  - [Barge-in](#barge-in)
  - [Security](#security)
  - [Performance](#performance)
  - [Testing](#testing)
  - [Repository layout](#repository-layout)
  - [Known limitations](#known-limitations)

---

## Quick start

Requires Docker (for Postgres + pgvector and Redis), [uv](https://docs.astral.sh/uv/),
and — for the local LLM — [Ollama](https://ollama.com/) with a model pulled:

```bash
ollama pull llama3.2:3b          # or set OLLAMA_MODEL to any model you have
cp .env.example .env             # optional: every value has a working default
docker compose up -d postgres redis
uv run alembic upgrade head      # once per fresh volume
uv run uvicorn app.main:app --reload
```

- API docs: <http://localhost:8000/docs>
- Health: `curl -s localhost:8000/health`
- Demo page: <http://localhost:8000/demo>

Only the two infrastructure containers are started here, so the API runs on the
host with `--reload` and you can edit it. For the fully containerised path let
compose run the API too — but run the migration *before* it starts, since the
schema is not created automatically:

```bash
docker compose up -d postgres redis
docker compose run --rm api uv run alembic upgrade head
docker compose up -d
```

The first turn downloads the Whisper weights and the Piper voice (~130 MB
combined) — `/health` stays fast, and the download is paid once, lazily, by the
first call rather than at boot.

Running without Docker (Postgres 16 with the `vector` extension and Redis
already running locally):

```bash
uv sync
uv run alembic upgrade head
OLLAMA_MODEL=llama3.2:3b uv run uvicorn app.main:app --reload
```

## Try the demo

The demo page is a single file (`app/static/demo.html`) served by the same
application: no npm, no build step, no CDN.

> **Open it through a server** — <http://localhost:8000/demo> — never by
> double-clicking the HTML. `getUserMedia()` only asks for a microphone on a
> *secure context*, and `http://localhost` counts as one while `file://` does
> not. A phone on your LAN cannot use it either; HTTPS is required there.

**Before you open the page**, create a tenant and copy its API key. The key is
shown exactly once, at creation, and only its SHA-256 is stored:

```bash
curl -s -X POST http://localhost:8000/tenants \
  -H 'Content-Type: application/json' \
  -d '{"name": "demo"}'
```

```json
{"id":"cf010712-9fb1-4d92-ac08-dddb476deaff","name":"demo","api_key":"avk_…"}
```

Optionally give it something to answer from — any `.txt`, `.md`, `.text`,
`.pdf`, `.docx`, `.html` or `.htm` (the extension selects the parser):

```bash
curl -s -X POST http://localhost:8000/documents \
  -H "X-API-Key: avk_…" \
  -F "file=@tests/fixtures/northwind_returns_faq.txt"
```

Then, in the page:

1. Paste the key, press **Start**, and allow microphone access.
2. Ask a question the documents answer — e.g. *"How long do I have to return a
   hardware device?"*. Watch the badge go `Listening… → Hearing you → Thinking →
   Speaking`, and read **Time to first sound** in the stats panel.
3. Press **Interrupt** (or talk over the answer, see below) — playback stops
   mid-sentence and the discarded audio is reported in the dialogue log.

The page shows what it can prove and nothing more: the state badge is the
*client's* own endpointing, the dialogue log records durations rather than
transcripts (the server does not send them), and **Turns / interrupts** counts
completed turns and barge-ins. A wrong or revoked key fails immediately and
visibly — the ticket request returns `401` and the page says so instead of
waiting for silence.

Two controls worth knowing:

- **Sample rate.** The server streams raw PCM16 with no rate in the protocol.
  Piper (the default) synthesises 22050 Hz; the selector defaults to that. Pick
  16000 for Deepgram's cloud STT, or whatever your TTS emits.
- **Half-duplex (checked by default).** While the assistant speaks, the page
  mutes the microphone so it cannot hear itself. That also means it cannot hear
  *you* — uncheck it, and use headphones, to test server-driven interruption.

## How a call flows

```
browser PCM16 16 kHz
      │
      ▼
 WebSocket /voice/ws  ──▶ bounded queue (backpressure, never an unbounded buffer)
      │
 VoiceSession ──▶ STT ──▶ final transcript
      │                        │
      │                   state: thinking
      ├──▶ retriever.search(tenant_id, text, top_k=5)
      │      └── HNSW cosine + Postgres full-text ──▶ RRF(k=60) fusion
      ├──▶ LLM stream ──▶ sentence chunker ──▶ TTS stream ──▶ binary audio ──▶ browser
      │
      └── interrupt: cancel the turn task, await it, then send {"type":"clear"}
```

Full diagrams, the module map, the data model and the reasoning behind each
choice are in [`docs/architecture.md`](docs/architecture.md).

## API

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| `GET` | `/health` | — | Liveness. Fast: models load lazily, never at import. |
| `POST` | `/tenants` | — | Create a tenant; returns the API key **once**. |
| `GET` | `/tenants/me` | `X-API-Key` | Current tenant and its limits. |
| `POST` | `/voice/ticket` | `X-API-Key` | Mint a 60-second WebSocket ticket. |
| `WS` | `/voice/ws?ticket=…` | ticket | The voice call. |
| `POST` | `/documents` | `X-API-Key` | Upload and index a document (multipart). |
| `GET` | `/documents` | `X-API-Key` | List the tenant's documents. |
| `GET` | `/documents/{id}` | `X-API-Key` | One document with its index status. |
| `GET` | `/demo` | — | The demo page. |

The WebSocket carries raw binary audio frames with no header: **PCM16, mono,
little-endian, 20 ms per frame**. The rate is fixed at **16 kHz inbound** and
**set by the TTS outbound** (22050 Hz for the default Piper voice, 16000 Hz for
the default ElevenLabs format) — the frame itself does not say which, so a
client has to know. The only JSON frame is a control message —
`{"type":"clear"}`, sent by the server when a turn is interrupted. Close codes:

| Code | Meaning |
| --- | --- |
| `4401` | Ticket missing, malformed or expired |
| `4408` | Idle: no audio for `WS_IDLE_TIMEOUT_SECONDS` (default 120) |
| `4429` | Too many concurrent calls for this tenant (default 3) |

```python
# pip install httpx websockets
import asyncio
import json

import httpx
import websockets


async def main() -> None:
    async with httpx.AsyncClient() as http:
        ticket = (await http.post("http://localhost:8000/voice/ticket",
                                  headers={"X-API-Key": "avk_…"})).json()["ticket"]
    url = f"ws://localhost:8000/voice/ws?ticket={ticket}"

    async with websockets.connect(url) as ws:
        async def send() -> None:
            with open("question_16k.wav", "rb") as f:         # mono 16 kHz PCM16
                while chunk := f.read(640):                  # 20 ms
                    await ws.send(chunk)

        async def receive() -> None:
            while True:
                msg = await ws.recv()
                if isinstance(msg, bytes):                    # assistant audio
                    ...
                else:                                        # {"type": "clear"}
                    print("interrupted:", json.loads(msg))

        async with asyncio.TaskGroup() as tg:
            tg.create_task(receive())
            tg.create_task(send())
```

Errors are JSON with a `detail` field and honest status codes: `201` on tenant
and document creation, `400` for an unsupported extension or a document that
yields no text, `401` for a bad or missing key, `404` for another tenant's
document, `413` for an upload over the size limit, `429` when a rate limit
trips. A provider that fails is mapped to `ProviderError` in its adapter, so no
vendor error shape escapes the adapter layer — but there is no error frame in
the WebSocket protocol, so a provider that dies mid-turn leaves the socket open
and silent, with the failure in the server log.

## RAG

**Ingestion** (`POST /documents`): parse → recursive character splitting into
~400-token chunks with 12 % overlap → embed with `BAAI/bge-small-en-v1.5`
(384-d, L2-normalised) → upsert. Identical chunks are skipped by content hash,
so re-uploading a file is cheap.

**Retrieval** (per turn, `top_k=5`): two independent candidate lists — cosine
over an HNSW index and Postgres full-text over a stored `tsvector`, 20
candidates each — fused with Reciprocal Rank Fusion (`k=60`). RRF combines
*ranks*, so no score normalisation is needed between two different retrievers.
Vector-only search misses exact tokens (order numbers, SKUs, names);
keyword-only misses paraphrase; that is the whole argument for two legs.

**Quality** is measured, not asserted —
[`docs/eval_report.md`](docs/eval_report.md), 20-question golden set:

| Metric | Result |
| --- | --- |
| Average recall | **1.000** |
| Full-recall questions | **20 / 20** |
| Expected document ranked first | **19 / 20** |
| Mean rank of the expected document | **1.05** |
| Average answer fact coverage | **0.908** |
| Retrieval p50 / p95 | **7.0 ms / 7.5 ms** |

Re-run it with `uv run python scripts/run_eval.py`. The report also records the
finding that most of this recall is *vector-only* (see
[Known limitations](#known-limitations)).

## Configuration

Every setting is an environment variable with a working default; see
`.env.example`.

| Variable | Default | Notes |
| --- | --- | --- |
| `ENV` | `local` | `production` switches secrets to AWS Secrets Manager. |
| `STT_PROVIDER` | `whisper` | `whisper` \| `deepgram` \| `fake` |
| `LLM_PROVIDER` | `ollama` | `ollama` \| `grok` \| `gemini` \| `fake` |
| `TTS_PROVIDER` | `piper` | `piper` \| `elevenlabs` \| `fake` |
| `EMBEDDINGS_PROVIDER` | `local` | `local` \| `vertex` |
| `STORAGE_PROVIDER` | `local` | `local` \| `s3` |
| `OLLAMA_MODEL` | `llama3.2:3b` | Must be pulled first: `ollama pull <model>`. |
| `WHISPER_MODEL` | `base` | `tiny` while iterating. |
| `PIPER_VOICE` | `en_US-lessac-medium` | Fetched on first use if `PIPER_MODEL_PATH` is unset. |
| `PIPER_LENGTH_SCALE` | `1.0` | >1 speaks more slowly. |
| `DEEPGRAM_ENDPOINTING_MS` | `300` | Cloud endpointing. |
| `WS_MAX_CONCURRENT_SESSIONS` | `3` | Per tenant; excess calls close with `4429`. |
| `WS_IDLE_TIMEOUT_SECONDS` | `120` | Silent calls close with `4408`. |
| `RATE_LIMIT_REQUESTS_PER_MINUTE` | `60` | Per tenant, Redis token bucket. |
| `JWT_SECRET` | `dev-secret-change-me` | **Must** be set in production; tickets are signed with it. |

Provider keys (`DEEPGRAM_API_KEY`, `ELEVENLABS_API_KEY`, `GROK_API_KEY`,
`GEMINI_API_KEY`) are read from the environment locally and from Secrets Manager
when `ENV=production`. None of them is ever logged or persisted.

## Local mode vs cloud

| | Local (default, free) | Cloud (production) |
| --- | --- | --- |
| **STT** | `whisper` = **batch**: buffered behind an amplitude VAD, transcribed only after you stop talking. No interim audio, so the LLM cannot start early. | `deepgram` = true streaming: interim transcripts *while* you speak, server-side endpointing, speech-onset events (and therefore barge-in). |
| **TTS** | `piper`: one finished sentence at a time, CPU/ONNX. | `elevenlabs`: streams audio out of partially written text. |
| **LLM** | `ollama`: a local model, no per-token cost. | `grok` / `gemini`: lower TTFT, higher quality, per-token billing. |
| **Hardware** | CPU only; one session holds a thread while Whisper/Piper run. | GPU/managed, horizontally scaled. |

This is a structural difference, not a tuning knob: local mode answers in
seconds, and a turn cannot be interrupted by voice. Switch with
`STT_PROVIDER=deepgram TTS_PROVIDER=elevenlabs …` when the call quality matters
more than the bill.

## Barge-in

Talking over the assistant is the one thing in a voice backend that is not a
sequence of `await`s: the user interrupts mid-turn, so a cancellation has to
land inside four cooperating generators (LLM → sentence chunker → TTS → socket)
and leave a conversation the *next* turn can continue.

- each turn is a single `asyncio.Task` (`VoiceSession._current_turn`), so an
  interrupt cancels one task instead of signalling four subsystems;
- the handler `await`s that cancelled task **before** sending
  `{"type":"clear"}`, so no audio can arrive after the clear and restart
  playback;
- a `_tap()` records every sentence *as it is handed to the TTS*, so an
  interrupted turn commits the answer the user actually heard — not the answer
  the model was still writing.

`tests/integration/test_voice_session_bargein.py` interrupts a real turn
between `FakeSTT`/`FakeLLM`/`FakeTTS` doubles and asserts all three properties.
The interrupt is a handshake, not a `sleep`, so it is deterministic on a loaded
CI box:

```bash
uv run pytest tests/integration/test_voice_session_bargein.py -v
```

**One honest limitation:** with the default local `whisper` adapter this cannot
be triggered by voice. Whisper is batch STT and never reports that the user
started speaking while the assistant is still talking; the state machine has no
`LISTENING -interrupt-> LISTENING` edge to fire. Barge-in needs a streaming STT
(`STT_PROVIDER=deepgram`). The demo's **Interrupt** button exercises the whole
client-side path — stop playback, discard the queue, return to listening — so
the behaviour is demonstrable today, but the *speech detection* is not.

## Security

- **API keys are shown once and stored hashed** (SHA-256). They are never
  logged, never returned again, and never written to the database in plaintext.
- **The WebSocket takes a ticket, not a key.** `POST /voice/ticket` mints a
  60-second HS256 JWT scoped to `ws_ticket`, so a key never travels in a query
  string where it would land in proxy logs and browser history.
- **Tenant context comes from the credential**, never from a request body —
  there is no `tenant_id` parameter anywhere in the API. Every query filters on
  it.
- **Rate limits and call caps** are Redis token buckets: 60 req/min per tenant,
  30 tenant creations/min per IP, 3 concurrent calls per tenant.
- **PII is redacted** before a transcript is stored or logged.
- **Secrets** come from the environment locally, from AWS Secrets Manager in
  production.
- **Not implemented:** Postgres row-level security. Isolation is enforced by
  the query layer today; RLS is defence-in-depth, deferred to before
  multi-tenant production traffic.

## Performance

Targets and measured numbers are tabulated in
[`docs/architecture.md` §5](docs/architecture.md#5-latency-budget). The short
version: retrieval p50 is 7 ms, LLM time-to-first-token p50 is 638 ms, and in a
real browser the first audio byte lands 1.2–1.5 s after the local endpoint
(2.9 s on a cold first turn, which pays the model download).

Per-stage spans (`stage`, `duration_ms`, `session_id`, `turn_id`, `first_ms`,
`outcome`) are emitted for every stage so these numbers can be re-measured
rather than believed.

## Testing

```bash
uv run ruff check .             # lint
uv run mypy app                 # types (59 files)
uv run pytest -m "not slow"     # 435 passed, 31 deselected — no downloads
uv run pytest -m slow           # 31 passed — real model downloads, needs network
```

Everything runs against doubles: `FakeSTT`, `FakeLLM` and `FakeTTS` implement
the same protocols as the real adapters, so the pipeline, barge-in and the
state machine are tested without a microphone, an API key or a network. The
`slow` marker covers the tests that download real model weights; CI runs the
fast lane only, and the golden-set eval is a separate, manual step.

## Repository layout

```
app/
  domain/      pure business logic: models, state machine, errors, ports
  adapters/    STT, LLM, TTS, embeddings, storage — one package per provider
  rag/         chunking, parsers, hybrid retriever, eval harness
  pipeline/    VoiceSession, sentence chunker, prompt builder
  api/         routes, auth dependencies, rate limiting, DTOs
  infra/       settings, DB, Redis, logging, telemetry, secrets, ORM, PII
  static/      demo.html — the browser demo, served by this app
  main.py      app factory, /demo route, lifespan
docs/          architecture, evaluation report, per-stage roadmaps and notes
scripts/       run_eval.py and other operator tools
tests/         unit, integration, contract, and the 20-question golden set
```

## Known limitations

Written down rather than glossed over.

**Voice and latency**

- **Local STT is batch, so there is no speech-onset barge-in.** Whisper
  transcribes only after you stop talking; the state machine has no
  `LISTENING -interrupt-> LISTENING` edge. Interruption by voice needs
  `STT_PROVIDER=deepgram`. The Interrupt button covers the client half today.
- **The local amplitude VAD is brittle.** Onset detection needs 300 ms of
  *consecutive* frames above a fixed RMS threshold (400). Speech with internal
  pauses, a quiet voice or a noisy room can fail to register an utterance —
  observed with synthetic speech in testing, and it is a fixed threshold, not
  an ML VAD.
- **Local turns take seconds, not milliseconds**, and the first one downloads
  ~130 MB of models.
- **The server streams faster than you can listen.** There is no playback
  flow control: a 15-second answer may finish generating in 4 seconds, so by
  the time you interrupt, the turn is already over and there is nothing to
  cancel. Interruption only helps while generation is still in flight.
- **The TTS sample rate is not negotiated.** The protocol carries raw PCM16 with
  no rate in the frame, so a client has to know it out of band; the demo page
  has a selector for that reason.
- **No transcripts reach the client.** The WebSocket protocol has no
  transcript or state frames, so the demo shows its own endpointing and
  durations. Shipping server transcripts needs a protocol change and a
  redaction pass in the socket path.
- **A provider failure mid-turn is silent to the client.** The turn task logs a
  `ProviderError` and stops; there is no error frame in the protocol, so the
  caller hears nothing and the socket stays open. The user-visible symptom is a
  conversation that stops answering.
- **The browser demo is not a telephony client.** No SIP/WebRTC signalling, no
  DTMF, no caller ID; `getUserMedia` means a browser, on a machine that can
  grant microphone access.

**RAG**

- **The full-text leg is nearly idle.** On the 20-question golden set it
  returned candidates for 1 of 20 questions, because `plainto_tsquery` ANDs
  every token and a multi-term question rarely has all its words in one chunk.
  Most of the measured recall is vector-only; the keyword leg exists for the
  query class the golden set does not cover.
- **Answer fact coverage is substring matching** over a normalised answer. It
  catches a wrong or empty answer and nothing subtler.
- **Ingestion is synchronous.** `POST /documents` parses, chunks, embeds and
  indexes inside the request, so a large PDF holds the connection open and a
  failure is an error response rather than a background job. The queue +
  webhook design is deferred, not forgotten.
- **No reranker.** Candidates are fused by rank only; a cross-encoder pass over
  the top 20 would improve precision on a larger corpus.
- **One embedder width.** `Vector(384)` is fixed by the chosen model; changing
  it is a migration, not a setting.

**Platform**

- **No row-level security**, no async ingestion, no automatic provider
  failover, and no live cloud deployment.
- **OpenTelemetry is in-process only.** Spans are emitted per stage but nothing
  exports them to a collector or a backend.
- **`deploy.yml` deploys nothing** — it holds the OIDC → ECR → ECS shape and an
  activation checklist for when an AWS account exists. Terraform modules are
  deferred for the same reason.
- **Compose does not run migrations automatically**; `alembic upgrade head` has
  to be run once per fresh volume.
- **CI does not exercise the real models.** Whisper, Piper, fastembed and
  Ollama are mocked in CI, so a provider adapter that stopped matching its
  protocol in a way the doubles allow would not be caught by the fast lane.
- **Concurrency is per-process.** Nothing coordinates sessions across replicas
  except the Redis call cap, so a horizontal scale-out needs a sticky-session
  or broker decision.
