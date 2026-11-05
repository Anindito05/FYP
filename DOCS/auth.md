# auth.md — How to Get Into the System and Run It

Part A: run the project on your machine. Part B: authenticate against the API.

---

## A. Run the backend locally

### 1. Prerequisites
Python 3.11+, a Neon account (free tier works), an LLM API key (or Ollama locally), Git.

### 2. Create the Neon database
1. Neon console → New project.
2. Connection details:
   - **Pooled connection ON** → copy → `DATABASE_URL`.
   - **Pooled connection OFF** → copy → `DATABASE_URL_DIRECT`.
3. Make both async-driver URLs ending in `?ssl=require`:
   `postgresql+asyncpg://USER:PASSWORD@HOST/neondb?ssl=require`
   (remove `sslmode=` and `channel_binding=` if present; asyncpg rejects them).
4. In the Neon SQL editor:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   CREATE EXTENSION IF NOT EXISTS pgcrypto;
   ```

### 3. Configure
```bash
cd mental-wellness-backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

| Variable | How to get it |
|---|---|
| `DATABASE_URL`, `DATABASE_URL_DIRECT` | Neon, as above |
| `JWT_SECRET` | `python -c "import secrets; print(secrets.token_urlsafe(64))"` |
| `FIELD_ENCRYPTION_KEY` | `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `LLM_PROVIDER`, `LLM_API_KEY`, `LLM_MODEL` | Your provider console (key can be empty for Ollama) |
| `RISK_THRESHOLD` | Start at 0.30, tune on the validation set |
| `CORS_ORIGINS` | Your frontend URL |

Never commit `.env`. Back up `FIELD_ENCRYPTION_KEY`: if it is lost, stored messages cannot be read.

### 4. Create tables and seed
```bash
alembic upgrade head                                   # uses DATABASE_URL_DIRECT
python -m scripts.seed                                 # helplines, admin, KB ingest tasks
```
(`scripts/seed.py` exposes `seed_helplines`, `create_admin`, `ingest_kb_folder`; call them from its `__main__` block as you implement it.)

### 5. Run
```bash
uvicorn app.main:create_app --factory --reload --port 8000
```
Docs: http://localhost:8000/docs. Health: http://localhost:8000/api/v1/health/ready

### 6. Tests
```bash
pytest -q        # point DATABASE_URL at a Neon test branch, not your dev data
```

Until you implement the stubs, endpoints raise `NotImplementedError`. Follow `steps.md`; Step 1 gets `/health/ready` working first.

---

## B. Authenticate against the API

### Model
- Access token: JWT, 15 minutes, sent as `Authorization: Bearer <token>`.
- Refresh token: 14 days, rotated on every use; reuse of an old one revokes the whole family.
- Roles: `user`, `reviewer`, `admin`.
- Guest mode: no email or password, for privacy-sensitive users.
- Passwords use Argon2id; message content is encrypted at rest.

### Endpoints (prefix `/api/v1`)

| Goal | Call | Body |
|---|---|---|
| Register | POST /auth/register | `{email, password, display_name, region_code, accepted_terms}` |
| Login | POST /auth/login | `{email, password}` |
| Guest | POST /auth/guest | `{region_code, accepted_terms}` |
| Refresh | POST /auth/refresh | `{refresh_token}` |
| Logout | POST /auth/logout | `{refresh_token}` + bearer |
| Who am I | GET /users/me | bearer |
| Delete account | DELETE /users/me | bearer |

### curl walkthrough
```bash
BASE=http://localhost:8000/api/v1

curl -X POST $BASE/auth/register -H 'Content-Type: application/json' \
  -d '{"email":"me@example.com","password":"StrongPassw0rd!","display_name":"Me","region_code":"IN","accepted_terms":true}'

TOKEN=$(curl -s -X POST $BASE/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"me@example.com","password":"StrongPassw0rd!"}' | jq -r .access_token)

SID=$(curl -s -X POST $BASE/sessions -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"mode":"text"}' | jq -r .id)

curl -X POST $BASE/chat/$SID/messages -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"content":"I am stressed about exams","client_msg_id":"c1"}'
```

### Admin and reviewer
1. Create the first admin with `create_admin` in `scripts/seed.py`.
2. Log in normally; the token carries `role=admin`.
3. Admin only: `/admin/kb`, `/admin/metrics`, `/admin/eval`.
4. Reviewer or admin: `/admin/risk-events` and `/admin/risk-events/{id}/review`.

### Public on purpose
`/auth/*`, `/resources/helplines`, `/health/*`. Helplines stay public so help is reachable even if login fails.

### Before going live
- [ ] Strong random `JWT_SECRET`; HTTPS only; CORS limited to your frontend.
- [ ] `.env` not in git; secrets set in the host's secret store.
- [ ] Helplines verified for each supported country.
- [ ] Rate limits enabled on `/auth/*` and chat.
- [ ] Disclaimer shown on first use: support tool, not therapy or a crisis line.

### Troubleshooting

| Symptom | Fix |
|---|---|
| asyncpg `ssl` / `sslmode` error | Use `?ssl=require`, drop `sslmode` and `channel_binding` |
| First request slow or times out | Neon compute was suspended; retry (`pool_pre_ping` helps) |
| Alembic fails or hangs | You used the pooled URL; use `DATABASE_URL_DIRECT` |
| `type "vector" does not exist` | Run `CREATE EXTENSION vector;` on that branch |
| 401 right after login | Access token expired (15 min); call `/auth/refresh` |
| Stored messages unreadable | `FIELD_ENCRYPTION_KEY` changed |
