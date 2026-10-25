# Tech Rules — Stack & Engineering Rules

> **This is the "how" of the app.** You own this file. Change a rule and the tutor will run an impact
> check on finished steps (see the journey skill, §6). Each rule has an id (`R-xx`) so lessons can cite it.
> The "what" lives in [PROJECT_BRIEF.md](PROJECT_BRIEF.md).

---

## 1. Tech stack

| Concern | Choice | Why | Alternatives |
|---------|--------|-----|--------------|
| Language | **Python 3.12** | Best AI/RAG ecosystem | Node.js/TypeScript |
| Package manager | **uv** | Fast, lockfile, reproducible | poetry, pip-tools |
| Web framework | **FastAPI** + Uvicorn | Async, typed, auto OpenAPI docs | Litestar |
| Realtime channel | **Socket.IO** (`python-socketio`, ASGI) + a plain **WebSocket** endpoint | Socket.IO gives rooms, acks, reconnection, Redis scaling; plain WS for tools that can't use Socket.IO | raw WS only |
| Validation / settings | **Pydantic v2**, `pydantic-settings` | One model language for API, events, config | — |
| Database | **PostgreSQL 16 + pgvector** | Relational data **and** vectors in one battle-tested DB | Qdrant, Weaviate |
| ORM / migrations | **SQLAlchemy 2.0 (async)** + **Alembic** | Mature, async, explicit | SQLModel |
| Cache / sessions / OTP / pub-sub | **Redis 7** | TTLs, atomic counters, Socket.IO adapter | Valkey |
| Background jobs | **arq** (async Redis queue) | Async-native, simple | Celery, Dramatiq |
| LLM provider | **Provider interface** — adapters for **Anthropic Claude**, **OpenAI**, **Ollama (local)** | Swap by config, avoid vendor lock-in | — |
| Default models (configurable) | Chat/answers: `claude-sonnet-5-5` · Routing & cheap tasks: `claude-haiku-5-5` | Quality where it matters, speed/cost elsewhere | OpenAI / local models |
| Embeddings | **Provider interface** — default **multilingual** model (e.g. `bge-m3` local via Ollama, or a hosted embedding API) | Arabic + English | OpenAI `text-embedding-3-*`, Voyage, Cohere |
| Reranking | Cross-encoder reranker behind an interface (local `bge-reranker` or hosted) | Better top-k precision | Cohere Rerank |
| HTTP client | **httpx** (async) + **tenacity** (retries) | Pooling, timeouts, retries | aiohttp |
| Email | Provider interface: **SMTP** (dev: Mailpit) / SendGrid | Swap per environment | SES |
| SMS | Provider interface: **Twilio** / console provider in dev | Swap per environment | Vonage, local gateway |
| Logging | **structlog** (JSON) | Machine-readable, contextual | — |
| Metrics / tracing | **Prometheus** + **OpenTelemetry** | Industry standard | — |
| Testing | **pytest**, pytest-asyncio, httpx test client, testcontainers | — | — |
| Quality | **ruff** (lint+format), **mypy** (types), pre-commit | — | — |
| Load testing | **Locust** | Socket + HTTP scenarios | k6 |
| Containers | **Docker** + **docker compose** (dev); multi-stage image (prod) | — | — |

## 2. Architecture rules

- **R-01 Layered, ports & adapters.** Dependencies point inward only:
  `channels/api → services (orchestrator, rag, verification) → domain → ports (interfaces)`.
  Adapters (LLM, embeddings, vector store, company API, email, SMS) implement ports and live in `app/adapters/`.
- **R-02 Every external dependency is behind an interface** with at least a real adapter and a fake/mock
  adapter for tests and local dev.
- **R-03 The LLM proposes, the code decides.** Tool execution goes through a policy check in code
  (verification level, confirmation, rate limits). Prompts are never the security boundary.
- **R-04 Stateless app processes.** All state lives in Postgres/Redis so we can run N workers behind a load balancer.
- **R-05 Configuration via environment** (12-factor) loaded into a typed `Settings` object; no config in code.
- **R-06 One socket event contract**, versioned and documented (`docs/protocol/`), independent of the channel library.

### Target code layout
```
app/
  main.py               # app factory: FastAPI + Socket.IO mount
  core/                 # settings, logging, errors, security utils
  channels/             # socket.io + websocket gateways, event schemas
  api/                  # REST: admin, health, knowledge management
  services/
    orchestrator/       # conversation loop, routing, tool policy
    rag/                # ingestion, chunking, retrieval, rerank, answer
    verification/       # OTP, verification levels
    sessions/           # session + memory management
    tools/              # tool definitions (policy, claims, ...)
  domain/               # pure models & business rules (no I/O)
  ports/                # interfaces (Protocols)
  adapters/             # llm/, embeddings/, vectorstore/, company_api/, email/, sms/
  db/                   # SQLAlchemy models, repositories, migrations
  workers/              # arq background jobs
mock_company_api/       # standalone FastAPI mock of the insurance API
tests/                  # unit/, integration/, e2e/, evals/
docker/                 # compose files, Dockerfiles
docs/                   # this journey
```

## 3. Coding rules

- **R-10** Full type hints; `mypy --strict` on `app/` must pass.
- **R-11** `async` for all I/O; never call blocking libraries in the event loop (use `run_in_threadpool` if unavoidable).
- **R-12** Pydantic models at every boundary (socket events, REST, tool inputs/outputs, external API responses).
- **R-13** Dependency injection through FastAPI dependencies / a small container — no global singletons except settings and pools.
- **R-14** Small modules, clear names, docstrings on public functions; comments explain *why*, not *what*.
- **R-15** `ruff` clean, formatted; pre-commit runs ruff + mypy.
- **R-16** Errors: domain exceptions mapped to user-safe messages; never leak stack traces or internal data to the chat.

## 4. Security rules

- **R-20** No personal data and no action before OTP verification (verification level checked in code per tool).
- **R-21** Write actions require an explicit confirmation turn; confirmation is bound to the exact action payload (hash) and expires.
- **R-22** Contact details for OTP come **only** from the company API, never from chat input.
- **R-23** OTP codes stored **hashed** (HMAC) with TTL; attempt and send limits enforced atomically in Redis.
- **R-24** PII is masked in logs, traces and LLM-visible tool results where not needed (e.g. phone `••89`).
- **R-25** Secrets only via env / secret manager; never committed; `.env` in `.gitignore`.
- **R-26** Prompt-injection defense: retrieved documents and tool results are wrapped as data, system prompt states they are untrusted; tools validate their own inputs.
- **R-27** Socket clients authenticate (per-integration API key or signed JWT); rate limits per session and per IP.
- **R-28** Every tool call and its result status is written to an **audit log** (who, what, when, verification level).

## 5. Performance rules

- **R-30** Stream LLM tokens to the client; never wait for the full answer.
- **R-31** Every external call has an explicit **timeout**; retries only for idempotent calls, with backoff + jitter; circuit breaker on the company API.
- **R-32** Connection pooling for Postgres, Redis and HTTP clients; created once at startup.
- **R-33** Heavy work (document ingestion, sending email/SMS) runs in background workers, not in the request path.
- **R-34** Cache: embeddings of repeated queries, frequent retrieval results; use provider prompt caching for long static system prompts.
- **R-35** Vector index (HNSW) and metadata indexes on every filtered column; retrieval p95 < 100 ms.
- **R-36** Bound conversation context: summarize old turns; cap tokens per request.

## 6. Quality rules

- **R-40** Each step ends runnable and verified; tests accompany new logic.
- **R-41** Unit tests use fake adapters; integration tests use real Postgres/Redis (testcontainers or compose).
- **R-42** A RAG evaluation set (questions → expected sources/answers) runs in CI once Stage 8 exists.
- **R-43** Structured JSON logs with `request_id`, `session_id`, `trace_id` on every line.

## 7. Git rules

- **R-50** One commit per finished step: `step X.Y: <title>` (only when you approve).
- **R-51** Never commit `.env`, keys, or customer data. Sample data is synthetic.
