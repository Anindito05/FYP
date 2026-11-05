# steps.md — Step-by-Step Build Plan

Build in this order. Each step lists the files to fill in and a "done when" check. Replace each `raise NotImplementedError` as you reach it.

Timeline assumes one semester (about 14 weeks) for Milestone 1 (text). Voice is optional.

---

## Step 0 — Environment and Neon (Day 1)

1. Create a Neon project; copy the **pooled** and **direct** connection strings.
2. In the Neon SQL editor: `CREATE EXTENSION IF NOT EXISTS vector; CREATE EXTENSION IF NOT EXISTS pgcrypto;`
3. `python -m venv .venv && pip install -r requirements.txt`
4. `cp .env.example .env` and fill it (see `auth.md`).

Done when: `psql "<direct url>" -c "select 1"` works.

## Step 1 — App skeleton (Week 1)

Files: `config.py`, `database.py`, `main.py`.

1. `Settings` and `get_settings`.
2. `build_engine`, `get_db`, `check_db_health`.
3. `create_app`, `lifespan`, `/health/live`, `/health/ready`.

Done when: `uvicorn app.main:create_app --factory --reload` runs and `/api/v1/health/ready` is OK against Neon.

## Step 2 — Models and migrations (Week 1)

Files: `models.py`, `migrations/env.py`.

1. Add columns, FKs and indexes from `design.md` section 4.3.
2. Point Alembic at `DATABASE_URL_DIRECT`.
3. `alembic revision --autogenerate -m "initial"`, review, add the HNSW index by hand, `alembic upgrade head`.

Done when: all 13 tables exist in Neon.

## Step 3 — Security and auth (Week 2)

Files: `security.py`, `services/auth_service.py`, `routers/auth.py`, `schemas.py` (auth part), `scripts/seed.py` (`create_admin`).

1. Argon2 hashing, JWT, Fernet encryption.
2. Register, login, guest, refresh with rotation and reuse detection, logout, delete account.
3. `get_current_user`, `require_role`, `rate_limit`.

Done when: `tests/test_auth.py` passes.

## Step 4 — Safety layer BEFORE the LLM (Weeks 3–4)

Files: `ml/risk.py`, `ml/safety.py`, `scripts/train_models.py` (`train_risk_model`), `tests/test_risk.py`.

1. Keyword/regex net with a versioned pattern list and obfuscation normalisation.
2. Fine-tune the risk classifier on the pre-released Kaggle suicide/depression dataset; save to `models_store/risk/`.
3. `combine` (OR, fail-closed) and `escalation_trend`.
4. Crisis templates and `check_response`. Ask a faculty or counseling contact to review the wording.
5. Evaluation of recall, FNR, F2 and the ablation in `scripts/run_eval.py`.

Done when: `tests/test_risk.py` passes and the eval prints model-only, keyword-only and combined numbers.

## Step 5 — Emotion and intent (Week 4–5)

Files: `ml/nlu.py`, `scripts/train_models.py` (emotion and intent).

1. Fine-tune or load emotion (GoEmotions) and intent classifiers.
2. `analyze_message` runs all detectors concurrently in a thread pool.

Done when: one call returns emotion, intent and risk decision in well under a second on CPU.

## Step 6 — RAG (Weeks 5–6)

Files: `rag.py`, `services/admin_service.py` (`ingest_document`), `scripts/seed.py` (`ingest_kb_folder`).

1. Put WHO/NIMH/university psychoeducation pages in `data/kb_raw/` (public, non-personal only).
2. Chunk, embed, insert into `kb_chunks`; create the HNSW index.
3. `retrieve` with a minimum-similarity gate (return nothing rather than weak context).

Done when: 20 hand-written queries return relevant chunks and off-topic queries return none.

## Step 7 — LLM client (Week 6)

Files: `llm.py`.

1. One provider (Claude or Ollama) behind `BaseLLMClient`.
2. `with_resilience`: timeout, retry with jitter, raise `UpstreamLLMError`.

Done when: unplugging the network yields the fallback reply, not a 500.

## Step 8 — Dialogue manager and chat API (Weeks 7–8)

Files: `services/chat_service.py`, `services/escalation_service.py`, `routers/chat.py`, `routers/resources.py`, `tests/test_chat_flow.py`.

1. `handle_message`, `normal_path`, `crisis_path`, idempotency via `client_msg_id`.
2. Escalation offer and consent-gated helplines/nearby.
3. Seed verified helplines (`scripts/seed.py`).

Done when: a flagged message never calls the fake LLM, and nearby lookup is blocked without consent.

## Step 9 — Admin, review and metrics (Week 9)

Files: `routers/admin.py`, `services/admin_service.py`.

1. Reviewer queue and labels.
2. `live_metrics`, `run_risk_eval`, `run_groundedness_eval`.
3. Retention purge.

Done when: labelling 10 risk events changes `/admin/metrics`.

## Step 10 — Hardening (Weeks 9–10)

1. Adversarial test phrases (slang, misspellings, indirect statements, multi-turn escalation) added to the eval set.
2. Rate limits on `/auth/*` and chat; CORS allow-list.
3. Account-erasure test.
4. Small load test (for example Locust, 20–50 users).

Done when: the adversarial recall is reported honestly, including its weaknesses.

## Step 11 — Deploy (Week 11)

1. Deploy the app, set env vars (pooled URL).
2. Run `alembic upgrade head` with the direct URL.
3. Seed helplines, create the admin, ingest the KB.
4. Connect the frontend.

Done when: a new user can register, chat, trigger the crisis path with a test phrase, and see consent-gated helplines on the hosted URL.

## Step 12 — Optional Milestone 2: voice (Weeks 13–15)

Add a `voice.py` (Whisper STT, TTS) and one route that feeds `chat_service.handle_message`. Crisis replies stay the same scripted text.

## Step 13 — Report evidence (Weeks 15–16)

Export `eval_runs`: risk recall/FNR/F2 with ablation, groundedness, latency p50/p95, pilot ratings, offer-acceptance rate. Include the non-goals from `design.md` section 7.

---

Order: `config → DB → models → auth → RISK → emotion/intent → RAG → LLM → chat → admin/eval → hardening → deploy → voice`

If time runs short, cut voice, never the safety evaluation.
