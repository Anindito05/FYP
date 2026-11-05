# architecture.md — Codebase Architecture

How the code is organised, who depends on whom, and how to extend it. Read this before writing any implementation.

---

## 1. Layered architecture

```
 HTTP request
      |
 [ routers/ ]        parse + validate (schemas.py), auth deps, call ONE service function
      |
 [ services/ ]       business rules, transactions, orchestration
      |       \
      |        +--> [ ml/ ]  risk.py, nlu.py, safety.py     (pure Python / models, no DB)
      |        +--> [ rag.py ]                              (embeddings + pgvector queries)
      |        +--> [ llm.py ]                              (external API)
      v
 [ models.py + database.py ]   SQLAlchemy async -> Neon PostgreSQL
```

Dependencies only point downward. `ml/`, `llm.py` never import from `routers/` or `services/`.

---

## 2. Module map

| File | Responsibility | Depends on |
|---|---|---|
| `main.py` | Build app, lifespan (load models once), mount routers, health | config, database, routers, ml, rag, llm |
| `config.py` | Environment settings | none |
| `database.py` | Engine, `get_db`, `Base` | config |
| `models.py` | ORM tables | database |
| `schemas.py` | API contracts (Pydantic) | none |
| `security.py` | Hash, JWT, encrypt, `get_current_user`, `require_role`, `rate_limit` | config, database, models |
| `routers/auth.py` | Account endpoints | auth_service, security |
| `routers/chat.py` | Sessions, chat, escalation, feedback | chat_service, escalation_service |
| `routers/resources.py` | Helplines, nearby, consents | escalation_service |
| `routers/admin.py` | KB upload, risk review, metrics, eval | admin_service |
| `services/auth_service.py` | Register, login, rotate tokens, guest, erase | models, security |
| `services/chat_service.py` | **Dialogue manager**: the full chat turn | ml, rag, llm, models, security |
| `services/escalation_service.py` | Offer, consent gate, helplines, nearby | models, ml.safety |
| `services/admin_service.py` | Ingest KB, review, metrics, evaluation, purge | models, rag, ml |
| `ml/risk.py` | Keyword net, risk model, combiner, trend | none (model weights) |
| `ml/nlu.py` | Emotion, intent, concurrent `analyze_message` | ml/risk |
| `ml/safety.py` | Scripted crisis replies, output filter | none |
| `rag.py` | Embed, chunk, ingest, retrieve, prompt, citations | models, config |
| `llm.py` | `BaseLLMClient`, provider, retry/timeout | config |
| `scripts/*` | Seeding, training, evaluation CLI | services, ml |
| `tests/*` | Unit and flow tests | everything |

---

## 3. Sequence: normal turn

```
Client -> router.send_message
router -> chat_service.handle_message
  chat_service -> models: save encrypted user msg           (commit)
  chat_service -> nlu.analyze_message
        nlu -> [emotion | intent | keyword_scan | risk model]  (parallel)
  chat_service -> risk.combine  -> RiskDecision(none)
  chat_service -> rag.retrieve (pgvector)
  chat_service -> rag.build_messages
  chat_service -> llm.generate (with_resilience)
  chat_service -> safety.check_response
  chat_service -> models: save assistant msg + analysis     (commit)
router <- MessageOut(reply, citations, emotion)
```

## 4. Sequence: crisis turn

```
router -> chat_service.handle_message
  ... analyze_message -> risk.combine -> RiskDecision(high|medium)
  chat_service.crisis_path
     safety.build_crisis_reply           (template, no LLM)
     escalation_service.create_offer     (consent question)
     models: save risk_event + messages  (commit)
router <- MessageOut(route="crisis", escalation_offer)

later: Client -> router.respond_to_escalation
  escalation_service.handle_choice
     accept -> helplines (and nearby only if location_lookup consent)
     decline -> supportive follow-up, conversation continues
```

---

## 5. Conventions

- **Naming**: routers are nouns (`chat.py`), services end with `_service.py`, async everywhere that touches I/O.
- **Stub format** (every method): signature with typed incoming parameters and outgoing type, then a docstring stating purpose, `Incoming:` and `Outgoing:`.
- **Errors**: services raise `HTTPException` (401/403/404/429) or `UpstreamLLMError`; chat_service converts the latter into `safe_fallback`.
- **IDs**: UUID everywhere. **Time**: UTC `timestamptz`.
- **Transactions**: one `AsyncSession` per request via `get_db`. Commit the user message first, never hold a transaction open across the LLM call.
- **CPU-bound ML**: run in a thread pool inside `analyze_message`; never block the event loop.
- **No message text in logs.**
- **Namespace packages**: `routers/`, `services/`, `ml/` have no `__init__.py` (not needed on Python 3.3+). Add them if your tooling complains.

---

## 6. Where to change things

| Task | Touch |
|---|---|
| Add an endpoint | `schemas.py` (I/O) → function in a service → route in a router |
| Add a table | `models.py` → `alembic revision --autogenerate` → review → upgrade |
| Swap LLM provider | new class in `llm.py` + `build_llm_client` branch |
| Tune safety | `ml/risk.py` patterns/threshold, `ml/safety.py` templates; re-run `scripts/run_eval.py` |
| Add KB content | drop files in `data/kb_raw/`, run `scripts/seed.py` ingest task |
| Add a language | locale-specific crisis templates and helplines |
| Add voice (Milestone 2) | new `voice.py` (STT/TTS) + one route that calls `chat_service.handle_message` |

## 7. Growth path (without restructuring)

1. If a file passes about 300 lines, split it along the same function groups (for example `chat_service.py` into `session_service.py` and `dialogue_manager.py`).
2. Add a `repositories/` layer only when SQL starts repeating across services.
3. Move `ml/` behind an internal HTTP service when you need independent scaling; `analyze_message` is the only call site.
4. Replace in-memory rate limiting with Redis in `security.rate_limit`.

## 8. Testing strategy

| Test file | Proves |
|---|---|
| `test_risk.py` | OR logic, fail-closed behaviour, trend detection |
| `test_auth.py` | Register → login → refresh; reused refresh token rejected |
| `test_chat_flow.py` | Crisis turn never calls the LLM; nearby lookup blocked without consent |

Use a Neon test branch and a fake LLM; never test against real user data.
