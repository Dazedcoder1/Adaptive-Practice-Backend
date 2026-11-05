# AdaptiveLearn AI - Backend

FastAPI modular monolith for the AdaptiveLearn AI platform (Student Portal + Super Admin Portal).
PostgreSQL (Neon) with pgvector, Redis, Celery, Cloudflare R2. Versioned REST API under `/api/v1`.

The web app is in the separate frontend repo: Dazedcoder1/Adaptive-Practice-Frontend.

## Layout

```
app/config/        settings, database, redis, security
app/modules/       domain modules (identity, curriculum, question_bank, practice, adaptive,
                   knowledge_model, revision, error_notebook, ai_tutor, rag, ...)
app/shared/        exceptions, middleware, permissions, events (outbox)
app/workers/       Celery app, outbox relay, consumers, AI tasks
app/integrations/  email, storage (R2), llm, embeddings
migrations/        Alembic
tests/             pytest (real Postgres + Redis)
scripts/           check_db, check_r2, create_superadmin, seed_sample_content
docs/operations.md admins, migrations, backups, background jobs, AI, security notes
infra/nginx/       production reverse proxy (serves the web app and API on one origin)
```

## Setup (Windows PowerShell)

Copy `.env.example` to `.env` in this folder and fill it in (Neon URLs, JWT secret, R2).

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
alembic upgrade head
python -m scripts.seed_sample_content
python -m scripts.create_superadmin --email you@example.com
uvicorn app.main:app --reload
```

API docs: http://localhost:8000/api/v1/docs

Redis is optional locally (rate limiting fails open): `docker compose up -d redis`.
Workers are optional while `PROCESS_EVENTS_INLINE=true`:

```powershell
celery -A app.workers.celery_app worker -Q default,ai,analytics -l info --pool=solo
celery -A app.workers.celery_app beat -l info
```

With `EMAIL_BACKEND=console`, verification / reset / consent emails are printed in this terminal.

### AI tutor (optional)

Works without any AI provider (answers come from approved study notes). To switch AI on, add to
`.env` and restart: `LLM_PROVIDER=openai` (or `gemini`, `anthropic`, `openai_compatible`),
`LLM_API_KEY=...`, and optionally `EMBEDDING_PROVIDER=openai` for semantic search.

## Tests

Needs a throwaway local Postgres with pgvector and a Redis (conftest refuses non-local databases):

```powershell
docker run -d --name ap-test-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=ap_test -p 5432:5432 pgvector/pgvector:pg16
docker run -d --name ap-test-redis -p 6379:6379 redis:7-alpine
$env:TEST_DATABASE_URL="postgresql://postgres:postgres@localhost:5432/ap_test"
pytest
```

CI (GitHub Actions) runs ruff, format check, migrations (up/check/down/up) and the test suite.
