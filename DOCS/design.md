# design.md — HLD, LLD and Database Design

Project: LLM-Based Mental Wellness & Counseling Support System. Backend: FastAPI + Neon PostgreSQL (with pgvector).
Scope: college final-year project, so the structure is deliberately compact (about 30 files) but keeps the layers that matter.

---

## 1. Design principles

| Principle | How it shows up in the code |
|---|---|
| Safety is the centerpiece | Risk detection (`ml/risk.py`) runs independently of the LLM. A flagged message never reaches free-form generation. |
| Fail-closed | If the risk model fails, the keyword net plus a conservative default decides. An error never means "no risk". |
| Auditable | Every turn stores emotion, intent and risk scores plus model versions. Risk events get human review labels. |
| Consent-first | Helplines-by-location, nearby search and data logging are gated by the `consents` table. |
| Privacy | Guest accounts, encrypted message content, hashed IPs, retention purge, account erasure. |
| Swappable | LLM sits behind `BaseLLMClient`; all SQL lives in services via `AsyncSession` (no repository layer, to keep the project small). |
| Scalable path | Stateless API, async I/O, Neon pooling, pgvector in the same DB. Split ML into its own service later without changing endpoints. |

---

## 2. High-Level Design (HLD)

### 2.1 Components

```
 Client (React / Streamlit)
        | HTTPS + JWT
        v
 +----------------------- FastAPI (stateless) -----------------------+
 |  routers/  (auth, chat, resources, admin)      <- thin HTTP layer  |
 |      |                                                             |
 |  services/ (auth, chat = dialogue manager, escalation, admin)      |
 |      |-- ml/nlu.py     emotion + intent + analysis pipeline        |
 |      |-- ml/risk.py    keyword net + classifier + combiner         |
 |      |-- ml/safety.py  crisis scripts + output filter              |
 |      |-- rag.py        embed -> pgvector retrieve -> prompt        |
 |      |-- llm.py        provider client + retry/timeout             |
 |      v                                                             |
 |  models.py / database.py  (SQLAlchemy async)                       |
 +--------------------------------------------------------------------+
        |                                  |
        v                                  v
  Neon PostgreSQL (+pgvector)        LLM API (Claude / Ollama)
```

### 2.2 One chat turn

1. `POST /api/v1/chat/{session_id}/messages` → JWT resolved by `security.get_current_user`.
2. `chat_service.get_owned_session` checks ownership; duplicate `client_msg_id` returns the stored reply.
3. User message is encrypted and saved.
4. `nlu.analyze_message` runs emotion, intent, keyword scan and risk model concurrently.
5. `risk.combine` produces a `RiskDecision` (OR logic, fail-closed, plus multi-turn trend).
6. Routing in `chat_service.handle_message`:
   - risk → `crisis_path`: scripted reply from `safety.build_crisis_reply` plus an `EscalationOffer`; a `risk_events` row is written; **LLM not called**.
   - no risk → `normal_path`: `rag.retrieve` → `rag.build_messages` → `llm` → `safety.check_response` → citations.
7. Assistant message and analysis are saved; `MessageOut` is returned.
8. If the user accepts the offer (`POST /chat/{id}/escalation`), `escalation_service.handle_choice` returns helplines, or nearby services if `location_lookup` consent exists.

### 2.3 Deployment

- API in one container (Render / Railway / HF Spaces / college VM).
- Neon: pooled URL for the app, direct URL for Alembic. Use a Neon branch for tests.
- Model weights in `models_store/` (git-ignored), loaded once in `lifespan`.

### 2.4 Cross-cutting

| Concern | Approach |
|---|---|
| Auth | JWT access (15 min) + rotating refresh tokens with reuse detection; roles `user`, `reviewer`, `admin`; guest mode |
| Rate limit | `security.rate_limit` dependency (in-memory; Redis later) |
| Resilience | LLM retry + timeout → `safe_fallback` reply; DB `pool_pre_ping` for suspended Neon compute |
| Config | Environment variables only (`config.py`) |
| Logging | Standard logging; never log message text |

---

## 3. Low-Level Design (LLD)

### 3.1 File map

```
mental-wellness-backend/
  design.md  steps.md  auth.md  architecture.md
  requirements.txt  .env.example
  app/
    main.py            app factory, lifespan, health
    config.py          Settings
    database.py        engine, get_db, Base
    models.py          13 ORM tables
    schemas.py         all Pydantic schemas
    security.py        hashing, JWT, encryption, auth deps, rate limit
    routers/           auth.py  chat.py  resources.py  admin.py
    services/          auth_service.py  chat_service.py  escalation_service.py  admin_service.py
    ml/                risk.py  nlu.py  safety.py
    rag.py             embeddings + retrieval + prompt
    llm.py             LLM client
  migrations/env.py    Alembic (direct Neon URL)
  scripts/             seed.py  train_models.py  run_eval.py
  data/                helplines.csv  kb_raw/
  tests/               test_auth.py  test_risk.py  test_chat_flow.py
```

### 3.2 Layer rules

| Layer | May call | Must not |
|---|---|---|
| routers | services, schemas, security deps | touch ML or write SQL |
| services | models, ml, rag, llm | import FastAPI request objects |
| ml / rag / llm | their own helpers | read the DB (rag receives a session) |

### 3.3 Risk decision logic (pseudo-flow)

```
kw    = keyword_scan(normalize_text(text))
model = try predict_risk(text, last_3_turns) except -> None
level =
  high    if kw.severity == high or model.p >= T_high
  medium  if kw.hits or model.p >= T
  medium  if model is None and kw has any weak hit        # fail-closed
  none    otherwise
if level == none and escalation_trend(recent_levels): level = low
```
Thresholds are tuned on a validation split for a target recall, and the FNR is reported.

### 3.4 API surface (prefix `/api/v1`)

| Method | Path | Auth |
|---|---|---|
| POST | /auth/register, /auth/login, /auth/guest, /auth/refresh, /auth/logout | public / bearer |
| GET, DELETE | /users/me | user |
| POST, GET | /sessions, /sessions/{id}/messages; POST /sessions/{id}/close | user |
| POST | /chat/{id}/messages, /chat/{id}/escalation | user |
| POST | /feedback | user |
| GET, PUT | /consents | user |
| GET | /resources/helplines | public |
| GET | /resources/nearby | user + consent |
| POST, GET | /admin/kb, /admin/risk-events, /admin/risk-events/{id}/review, /admin/metrics, /admin/eval | admin / reviewer |
| GET | /health/live, /health/ready | public |

---

## 4. Database Design (Neon PostgreSQL)

### 4.1 Neon notes

- Run once: `CREATE EXTENSION IF NOT EXISTS vector; CREATE EXTENSION IF NOT EXISTS pgcrypto;`
- Pooled URL (`-pooler` host) for the app; direct URL for Alembic.
- asyncpg URL uses `?ssl=require` (not `sslmode`).
- Compute suspends when idle: first query may be slow → `pool_pre_ping` and a retry.
- Use Neon branches for dev/test isolation.

### 4.2 Relationships

```
users 1-* refresh_tokens        users 1-* consents
users 1-* chat_sessions 1-* messages 1-1 message_analysis
chat_sessions 1-* risk_events *-1 messages
messages 1-* feedback
kb_sources 1-* kb_chunks(vector)
helplines    audit_logs    eval_runs     (standalone)
```

### 4.3 Tables (all have `id uuid default gen_random_uuid()` and `created_at timestamptz default now()`)

| Table | Key columns | Indexes / constraints |
|---|---|---|
| users | email (nullable), password_hash, display_name, role, is_guest, is_active, region_code, locale, last_login_at, deleted_at | unique lower(email) where not null |
| refresh_tokens | user_id, family_id, token_hash, expires_at, revoked_at, replaced_by | unique token_hash; index family_id |
| chat_sessions | user_id, title, status, mode, started_at, ended_at, risk_peak_level, message_count | (user_id, started_at desc) |
| messages | session_id, role, content_enc bytea, turn_index, route, client_msg_id, latency_ms | unique (session_id, client_msg_id); (session_id, turn_index) |
| message_analysis | message_id (unique), emotion_label, emotion_scores jsonb, intent_label, risk_score, risk_level, risk_source, keyword_hits jsonb, model_versions jsonb | (risk_level) |
| risk_events | session_id, message_id, risk_level, action_taken, user_accepted_help, reviewed_by, review_label | partial index where reviewed_by is null |
| consents | user_id, type, granted, version, granted_at, revoked_at | unique (user_id, type, version) |
| helplines | country_code, name, phone, sms, url, languages[], hours, verified_at, is_active | (country_code, is_active) |
| kb_sources | title, publisher, url, license, checksum, ingested_at, is_active | unique checksum |
| kb_chunks | source_id, chunk_index, text, embedding vector(384), metadata jsonb | HNSW (embedding vector_cosine_ops) |
| feedback | message_id, user_id, rating 1..5, helpful, comment | (message_id) |
| audit_logs | actor_id, action, entity, entity_id, metadata jsonb | append-only |
| eval_runs | name, dataset, git_sha, model_versions jsonb, recall, precision, f2, fnr, groundedness, latency_p95_ms | (created_at) |

### 4.4 Privacy and retention

| Data | Handling |
|---|---|
| Message text | Encrypted at rest; purged after `LOG_RETENTION_DAYS` by `admin_service.purge_expired` |
| Analysis scores | Kept without text; used for research only if `research_use` consent exists |
| IP / user agent | Never stored raw |
| Account deletion | Soft delete then erase messages, sessions, feedback |

### 4.5 Scalability notes

- Keyset pagination only (no OFFSET).
- Commit the user message before the LLM call; never hold a DB transaction during the LLM call.
- HNSW is enough for up to about 10^6 chunks.
- When traffic grows: partition `messages` by month, add Redis for rate limits, move `ml/` behind its own service.

---

## 5. Uniqueness (what to claim in the report)

| Feature | Evidence you can report |
|---|---|
| Decoupled dual-signal risk layer (model OR keywords, fail-closed, trend) | Recall, FNR, F2; ablation model-only vs keyword-only vs combined |
| Reviewer labels on risk events | Live recall over time |
| Cited RAG answers | Groundedness score |
| Consent-first escalation | Offer-acceptance rate; no abrupt disconnect |
| Provider-agnostic LLM + fallback | Latency p95; fallback rate |
| Reproducible `eval_runs` | Results table tied to git sha and model versions |

## 6. Future work

Redis rate limiting and caching, a separate ML inference service, multilingual crisis scripts, voice (STT/TTS) as Milestone 2, a reviewer dashboard, and fine-tuning small open models.

## 7. Non-goals (state in the report)

Not a licensed therapist, not diagnostic, not a crisis line, does not contact authorities, and promises no confidentiality beyond what is implemented.
