# Data Exploration and Analysis Assistant

Upload a CSV, ask natural-language questions, and get plots + statistics + an LLM
interpretation. Claude reads a bounded profile of your data, selects from a fixed
menu of pandas/plot tools (histogram, scatter, correlation matrix), and writes the
interpretation — all in a single LLM pass.

Project 1 of a 3-project AI Engineering Workshop. This repo is a full-stack app:
a **FastAPI** backend and a **React + Vite** frontend.

## Run the webapp (two terminals)

You need **Python 3.12**, **Poetry**, **Node 18+**, and an **Anthropic API key**.

### 1. Configure your API key

```bash
cp .env.example .env
```

Then edit `.env` and set `ANTHROPIC_API_KEY=sk-ant-...`. The `sqlite` default for
`DATABASE_URL` works out of the box — no database to install. (Chat won't work
without a valid key; upload/profile will.)

### 2. Terminal A — backend (http://localhost:8000)

```bash
make install     # poetry install (first run only)
make dev         # uvicorn --reload; API docs at http://localhost:8000/docs
```

### 3. Terminal B — frontend (http://localhost:5173)

```bash
cd frontend
npm install      # first run only
npm run dev
```

Open **http://localhost:5173**. Upload a CSV, then ask questions like
*"show me the distribution of age"* or *"is income correlated with age?"*. The
dev server proxies `/api → http://localhost:8000`, so no CORS setup is needed.

## Try it without the frontend (curl)

```bash
# Make a tiny CSV to play with
printf 'age,income,city\n20,100,NY\n30,200,NY\n40,300,LA\n25,150,SF\n' > sample.csv

# Upload it → returns the dataset id
curl -X POST http://localhost:8000/datasets -F "file=@sample.csv;type=text/csv"
# -> {"id":"<dataset-uuid>","name":"sample.csv","n_rows":4,"n_cols":3}

# Start a chat over one or more datasets → returns the chat id (its own shareable link)
curl -X POST http://localhost:8000/chats \
  -H 'Content-Type: application/json' \
  -d '{"dataset_ids":["<dataset-uuid>"]}'
# -> {"id":"<chat-uuid>","datasets":[{"id":"<dataset-uuid>","name":"sample.csv"}]}

# Ask a question (needs ANTHROPIC_API_KEY set)
curl -X POST http://localhost:8000/chats/<chat-uuid>/messages \
  -H 'Content-Type: application/json' \
  -d '{"question":"show the distribution of income"}'
```

## Endpoints

| Method | Path                          | Purpose                                                        |
| ------ | ----------------------------- | -------------------------------------------------------------- |
| GET    | `/health`                     | Liveness check → `{"status":"ok"}`                             |
| POST   | `/datasets`                   | Upload a CSV (multipart `file`): validate → profile → persist  |
| GET    | `/datasets`                   | List all datasets (newest first)                               |
| GET    | `/datasets/{id}`              | Fetch a dataset by id                                          |
| POST   | `/chats`                      | Create a chat over one or more `dataset_ids`                   |
| POST   | `/chats/{chat_id}/messages`   | Ask a question → Claude selects tools → charts + interpretation |
| GET    | `/chats/{chat_id}`            | Chat history (charts re-rendered on the fly, never stored)     |

Re-uploading an identical CSV returns the existing dataset (deduplicated by
SHA-256 content hash) instead of creating a duplicate.

## Architecture at a glance

- **FastAPI** backend, **SQLAlchemy 2.0** models (`datasets`, `chats`,
  `chat_messages`, `analyses` — no `users` table; a dataset id *is* its
  shareable link).
- Uploaded data is stored verbatim as **raw CSV text** in `datasets.data_csv`
  (single durable copy, no object store). Charts are re-rendered on demand from
  `data_csv` + a message's `tool_calls`; nothing is stored as image files.
- `profile_dataframe()` builds a **bounded rich profile** (schema, `describe()`,
  cardinality-gated value counts, capped correlations, sample rows) within a
  token budget — this is what Claude reads.
- **Judge-gated LLM loop** (issue #9): an analyst pass reads the profile, picks
  tools, and writes the prose; the backend **validates every tool call** against
  the profile and renders the charts; a separate judge pass then scores the
  attempt (0-100) against the actual rendered charts + stats. The loop repeats,
  feeding the analyst its own charts/stats and the judge's feedback, until the
  score clears a threshold or a pass cap is hit (default 3 passes) — a turn may
  take longer than a single API call as a result. Each answer carries a
  collapsible **loop trace** rendered with the tokenized UI — each pass is its
  own collapsible block with a color-coded judge-score badge, the analyst
  interpretation, judge feedback/gaps, and the revision fed to the next pass;
  model/tokens/latency/cost is demoted to a muted footer line, and the system
  prompt is never exposed. See `doc/project-1-llm-loop-design.md`.
- The frontend ships a **tokenized component system** (CSS-variable design
  tokens + a Radix-based primitive library) with light/dark themes and a
  refreshed Upload + Chat UI (issue #11, Slice A). The composer sends on
  **Enter** (Shift+Enter for a newline), the upload page has a styled file
  picker, and the raw-statistics panel offers a **Download JSON** export. A
  collapsible left sidebar lists previously uploaded datasets — click one to
  start a chat, or multi-select several.

See `doc/project-1-csv-analysis-assistant-design.md` for the approved design and
the numbered design decisions (D2–D5, D2 now superseded by the judge-gated loop
in `doc/project-1-llm-loop-design.md`) referenced throughout the code.

## Development

Backend (from repo root):

```bash
make check        # CI gate: ruff + black --check + mypy (strict) + pytest
make test         # pytest only (ephemeral SQLite, no API key needed)
make lint         # ruff
make format       # black (auto-fix)
make type-check   # mypy backend/app (strict)
make migrate      # apply Alembic migrations (alembic upgrade head)
make migration m="describe change"   # autogenerate a migration from model changes
```

Frontend (from `frontend/`):

```bash
npm test          # Vitest + React Testing Library (mocked fetch, no backend)
npm run build     # type-check (tsc) then production build
npm run type-check
```

Tests run against ephemeral SQLite; production uses Postgres via `DATABASE_URL`.

### Database migrations (Alembic)

The Postgres schema is managed by **Alembic** (`backend/alembic/`). On app
startup `init_db()` runs `alembic upgrade head`, so a fresh database is created
and an existing one is brought up to date automatically. To apply migrations
manually, run `make migrate`. After changing a model in `app/models.py`,
generate a migration with `make migration m="add foo column"`, review the
generated file under `backend/alembic/versions/`, then `make migrate`. (SQLite —
the local default and the test backend — skips Alembic and builds tables
directly from the ORM metadata.)

To reset a Postgres dev DB from scratch:

```bash
psql "$DATABASE_URL" -c "DROP SCHEMA IF EXISTS app CASCADE; DROP TABLE IF EXISTS public.alembic_version;"
make migrate
```

(The legacy `backend/db/schema.sql` is a Week-2 learning artifact — do not use
it to build the app database; it diverges from the ORM models.)

## Testing

### One-time: make the toolchain visible to your shell

Poetry and pyenv are not on `PATH` in a fresh terminal. Either prepend this to
each command, or add it to `~/.zshrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
eval "$(pyenv init -)"
```

The project pins Python **3.12** (`.python-version`) and uses **Poetry** for
deps; run `make install` first if you haven't. Run a single backend test file
with `poetry run pytest backend/tests/test_datasets.py -v`.

### Manual testing against a live server

Start the backend with `make dev` — serves on **http://127.0.0.1:8000** and
writes to a local SQLite `dev.db` (the default `DATABASE_URL`), so uploads and
chats persist across restarts. Delete `dev.db` to reset. Interactive API docs
(Swagger) are at **http://127.0.0.1:8000/docs** — you can upload a CSV and
try endpoints straight from the browser there.

Exercise the dataset + chat flow with the curl walkthrough above, then check
these error paths:

```bash
# unknown dataset id -> 404
curl -i http://127.0.0.1:8000/datasets/nope

# non-.csv file -> 400
echo hi > /tmp/notes.txt
curl -i -X POST http://127.0.0.1:8000/datasets -F "file=@/tmp/notes.txt;type=text/plain"

# header-only / no data rows -> 400
printf 'a,b\n' > /tmp/empty.csv
curl -i -X POST http://127.0.0.1:8000/datasets -F "file=@/tmp/empty.csv;type=text/csv"
```

| Case | Expected status |
| --- | --- |
| valid CSV upload | 200 |
| get existing dataset | 200 |
| get unknown id | 404 |
| non-`.csv` file | 400 |
| header-only CSV | 400 |
| over `MAX_UPLOAD_BYTES` | 413 |
| over `MAX_ROWS` | 413 |
| unknown dataset id in `POST /chats` | 404 |
| empty `dataset_ids` in `POST /chats` | 400 |

### Inspecting the stored profile

`DatasetOut` returns only `id/name/n_rows/n_cols` — the full bounded rich
profile that Claude reads is persisted in `datasets.profile_json` and isn't
exposed via an endpoint. To eyeball it (uses `dev.db`):

```bash
poetry run python -c "from app.db import SessionLocal; from app.models import Dataset; import json; s=SessionLocal(); d=s.query(Dataset).first(); print(json.dumps(d.profile_json, indent=2))"
```

## Configuration

Backend settings live in `app.config.Settings`, overridable per-environment via
`.env` (copy `.env.example`):

| Key | Purpose |
| --- | --- |
| `ANTHROPIC_API_KEY` | **Required for `/chat`.** Your Anthropic key. |
| `ANTHROPIC_MODEL` | LLM model id (default `claude-sonnet-5`). |
| `ANTHROPIC_MAX_TOKENS` | Max output tokens per call (default 4096). |
| `DATABASE_URL` | `sqlite:///./dev.db` by default; Postgres in production. |
| `MAX_UPLOAD_BYTES`, `MAX_ROWS` | Upload caps. |
| `CORS_ALLOW_ORIGINS` | Allowed frontend origins (prod only; dev uses the Vite proxy). |
| `PROFILE_MAX_CARDINALITY`, `PROFILE_MAX_CORR_COLS`, `PROFILE_TOP_CORR_PAIRS`, `PROFILE_SAMPLE_ROWS`, `PROFILE_TOKEN_BUDGET` | Profiler tunables. |
| `LLM_MAX_PASSES` | Max analyst passes per turn in the judge-gated loop (default 3). |
| `LLM_QUALITY_THRESHOLD` | Judge score (0-100) that stops the loop early (default 80). |
| `JUDGE_MODEL` | Model id for the judge call; empty (default) reuses `ANTHROPIC_MODEL`. |

The frontend reads `VITE_API_BASE` (see `frontend/.env.example`); it defaults to
`/api`, which the dev server proxies to the backend — no change needed for local
development.

## Documentation

- `doc/architecture.md` — student-oriented architecture overview with diagrams
- `doc/project-1-csv-analysis-assistant-design.md` — approved design + decisions
- `frontend/README.md` — frontend-specific dev notes
- `CLAUDE.md` — guidance for working in this repo
