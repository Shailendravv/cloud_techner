# Setup Guide — Dev Profile (`APP_ENV=dev`)

This guide sets up the project in **dev mode only**: the index lives in Supabase
Postgres, embeddings come from Gemini, answers come from Groq, and every run is
traced to Langfuse. This is the same configuration the deployed Vercel function
runs.

Follow the steps in order. Steps 1–4 are one-time setup; steps 5–6 are what you
run each day.

| Piece | What dev mode uses |
| --- | --- |
| Store | Supabase Postgres + pgvector (`STORE_BACKEND=supabase`) |
| Embeddings | Gemini `gemini-embedding-001`, 768 dimensions |
| Generation | Groq |
| Tracing | Langfuse |
| Backend | FastAPI on `http://localhost:8000` |
| Frontend | Vite + React on `http://localhost:5173` |

**Prerequisites:** Python 3.10+, Node.js 18+, npm, and bash. On Windows use
**Git Bash** — not PowerShell or CMD. No Docker, nothing to install locally for
the database; Supabase is hosted.

---

## Step 0 — Collect four sets of credentials

Get all of these before you start; every one is free and none needs a card.

| Credential | Where | Used for |
| --- | --- | --- |
| `GROQ_API_KEY` | <https://console.groq.com/keys> | generating answers |
| `GEMINI_API_KEY` | <https://aistudio.google.com/apikey> | embeddings (Groq has no embeddings endpoint) |
| `SUPABASE_URL` + `SUPABASE_SERVICE_KEY` | Supabase → Project Settings → API | the index |
| `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` | Langfuse → Project Settings → API Keys | tracing |

Use the Supabase **`service_role`** key, not the anon key. It is server-side
only: row-level security is enabled on every table with no permissive policy, so
the anon key reads nothing by design. Never put the service key in the frontend.

---

## Step 1 — Create the database and run the migrations

1. Create a free project at <https://supabase.com>. Note the region and the
   database password prompt — you will not need the password, but the project
   takes a minute to provision.

2. Open **SQL Editor** in the Supabase dashboard and run each file from
   `backend/supabase/migrations/` **in numeric order**, one at a time. Paste the
   file contents, run, confirm success, then move to the next:

   | File | What it creates |
   | --- | --- |
   | `0001_schema.sql` | the `vector` extension, `documents` and `chunks` tables (embedding is `vector(768)`), the indexes and tsvector columns |
   | `0002_hybrid_search.sql` | `hybrid_search()` and `lexical_search()` — dense similarity, full-text ranking, RRF fusion and the metadata pre-filter in one statement |
   | `0003_cache_and_jobs.sql` | `answer_cache`, `ingestion_jobs`, `corpus_meta`, `bump_corpus_version()` |
   | `0004_lexical_or_search.sql` | `or_tsquery()` and a corrected `hybrid_search()` — the lexical arm returned nothing before this |
   | `0005_atomic_chunks.sql` | `replace_chunks()` — delete-and-insert in one transaction |

   Order matters: `0004` replaces the `hybrid_search` from `0002`, and running
   them out of order leaves you with the broken lexical arm.

3. Verify the schema landed. In the SQL editor:

   ```sql
   select table_name from information_schema.tables
   where table_schema = 'public' order by table_name;
   -- expect: answer_cache, chunks, corpus_meta, documents, ingestion_jobs

   select routine_name from information_schema.routines
   where routine_schema = 'public' order by routine_name;
   -- expect: bump_corpus_version, hybrid_search, lexical_search,
   --         or_tsquery, replace_chunks
   ```

   The tables are empty at this point. Step 4 fills them.

---

## Step 2 — Configure `backend/.env`

```bash
cd backend
cp .env.example .env
```

Then edit `backend/.env` so these are set. The file is gitignored.

```bash
APP_ENV=dev

GROQ_API_KEY=gsk_...
GEMINI_API_KEY=...

SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_KEY=<service_role key>

LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

`APP_ENV=dev` is what selects `backend/config/dev.json` — the profile that sets
`STORE_BACKEND=supabase`, `EMBEDDER=gemini` and `EMBEDDING_DIMS=768`. Everything
in that file is committed and tunable; `.env` carries secrets only.

The backend loads `backend/.env` itself on import, so anything saved here works
for the server, the CLI and the ingest pipeline alike. A real environment
variable always wins over both `.env` and the profile.

---

## Step 3 — Backend virtual environment

```bash
cd backend

python3 -m venv .venv
source .venv/bin/activate          # Windows Git Bash: source .venv/Scripts/activate

pip install -r requirements.txt
```

**That is the only install command.** There is one requirements file and it
covers everything: the API server (`uvicorn`), the agent, the test suite, the
ingestion pipeline and the documentation site. The file is grouped into
commented sections if you want to see what each package is for.

It takes a few minutes the first time. Most of that is `fastembed`, which
pulls ~200 MB of onnxruntime for the offline embedder — that is what lets the
test suite and the evaluation harness run with no API key at all.

Nothing here installs a Supabase client: the store talks to PostgREST over
stdlib `urllib`, and the `supabase` package (and its gotrue/storage3/realtime
tree) is intentionally absent.

---

## Step 4 — Seed the database

This is the offline ingestion pipeline: it reads the twelve markdown runbooks
from `backend/runbooks/`, parses and chunks them, embeds each chunk with Gemini,
and upserts documents and chunks into Supabase. It never runs inside a request
handler.

Dry run first — this parses and chunks, writes nothing and embeds nothing, so it
costs no quota and is a cheap way to confirm your config is being read:

```bash
cd backend
source .venv/bin/activate

python -m ingest.pipeline --seed --profile dev --dry-run
```

Then the real run:

```bash
python -m ingest.pipeline --seed --profile dev
```

It prints a JSON report of what was parsed, embedded and written. It is
**idempotent on content hash** — re-running over unchanged documents re-embeds
nothing, writes nothing and consumes no Gemini quota. Use `--force` to re-embed
anyway, which is what you want after changing the chunking policy.

### Verify the seed

In the Supabase SQL editor:

```sql
select count(*) from documents;                         -- expect 12
select count(*) from chunks;                            -- expect roughly 70
select count(*) from chunks where embedding is null;    -- expect 0
```

**If that last query is not 0, your embeddings did not happen.** The usual cause
is `GEMINI_API_KEY` being unset or wrong when you seeded: the pipeline skips the
embed stage and writes chunks with null embeddings rather than failing. The
symptom afterwards is subtle — the lexical arm still works, so answers still
come back and nothing errors, but the dense arm is silently dead. Fix the key
and re-run with `--force`.

---

## Step 5 — Start the backend

```bash
cd backend
source .venv/bin/activate

python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Check it came up correctly:

```bash
curl http://localhost:8000/health
```

For a correct dev setup you want to see:

```json
{
  "status": "ok",
  "profile": "dev",
  "store": "supabase",
  "store_reachable": true,
  "corpus_docs": 12,
  "embedder": "gemini",
  "embedding_dims": 768,
  "groq_configured": true,
  "gemini_configured": true,
  "supabase_configured": true,
  "langfuse_configured": true
}
```

`/health` reports **what is actually mounted**, not what was requested — which
is the point of checking it. If Supabase is not configured the app falls back to
the files store rather than failing every request, and you will see
`"store": "files (supabase requested but not configured)"`. That is the one
line that distinguishes a working dev setup from one that looks fine and is
quietly reading markdown off disk.

Interactive API docs: <http://localhost:8000/docs>

---

## Step 6 — Start the frontend

In a second terminal tab:

```bash
cd frontend

echo "VITE_API_URL=http://localhost:8000" > .env     # first time only
npm install

npm run dev -- --port 5173 --strictPort
```

Open <http://localhost:5173>.

`VITE_API_URL` must match the port the backend is on — it is read at dev-server
start, so change it and restart Vite. The backend's CORS defaults already allow
`http://localhost:5173` and `http://127.0.0.1:5173`, so no CORS setup is needed
for local work. Add `CORS_ORIGINS` to `backend/.env` only when serving the UI
from somewhere else.

---

## Step 7 — Langfuse tracing

Tracing is **off unless both `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` are
set**. If you filled them in at step 2, it is already running.

### Getting the keys

1. Sign up at <https://cloud.langfuse.com> (EU) or <https://us.cloud.langfuse.com> (US)
2. Create an organization and a project
3. Project Settings → API Keys → Create new API keys
4. Copy the public (`pk-lf-...`) and secret (`sk-lf-...`) key

### `LANGFUSE_HOST` must match the region the keys came from

```bash
LANGFUSE_HOST=https://cloud.langfuse.com       # EU keys
LANGFUSE_HOST=https://us.cloud.langfuse.com    # US keys
```

Sending EU keys to the US origin fails authentication rather than quietly going
somewhere else, but since tracing is fire-and-forget you get no error — just no
traces. If your traces never appear, check this first.

### Where traces land

`LANGFUSE_TRACING_ENVIRONMENT` defaults to `APP_ENV`, so dev-mode traces land in
the **`dev`** environment in Langfuse and do not share dashboards with anything
else. Override it in `.env` if you want a different bucket.

### Verifying it works

1. `curl http://localhost:8000/health` → `"langfuse_configured": true`
2. Ask one question (step 8 below)
3. Open the Langfuse project → Tracing → Traces

A trace for one question has a tree of named observations, each typed for what
it did rather than all being undifferentiated spans:

| Observation | Type |
| --- | --- |
| `analyze-query` | span |
| `embed-query` | embedding |
| `search-lexical` | retriever |
| `search-dense` | retriever |
| `fuse-rankings` | chain |
| the grounding step | generation — carries model, token usage and therefore cost |
| the refusal gates | guardrail |

The ingest run from step 4 also traces (`extract-text`, `chunk-document`,
`embed-chunks`, `store-embeddings`), so if you seeded after configuring
Langfuse there is already a trace waiting.

### Turning it off without deleting keys

```bash
LANGFUSE_TRACING_ENABLED=false
```

Worth knowing: tracing is deliberately **silent on failure**. An observability
backend that can fail a request is worse than none, so every failure here costs
a trace and nothing else. That is why a bad key produces no error anywhere.

---

## Step 8 — Verify the whole thing end to end

Ask a question the corpus can answer:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "The checkout service is returning 500s after a deploy"}'
```

Ask one it cannot, which should be **declined**, not guessed at:

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How many vacation days do engineers get?", "explain": true}'
```

`"explain": true` returns the per-stage trace showing why it refused.

You can also drive it from the command line without the server running:

```bash
cd backend && source .venv/bin/activate
python -m agent "The checkout service is returning 500s after a deploy" --profile dev --trace
```

Then use the UI at <http://localhost:5173> and confirm answers and citations
render there too.

---

## Running it all with `start.sh`

`start.sh` can do steps 3, 5 and 6 in one command. In dev mode there is one
thing to know first.

```bash
APP_ENV=dev ./start.sh
```

The script installs `backend/requirements.txt` into `backend/.venv` itself, for
every profile, so a fresh clone needs nothing done by hand first. It stamps the
file's checksum after a successful install and skips pip while it matches —
only the first run is slow.

What the script does beyond that: prompts (masked) for `GROQ_API_KEY`,
`GEMINI_API_KEY`, `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` if any are missing
and offers to save them to `backend/.env`; writes `frontend/.env`; runs
`npm install`; starts uvicorn on 8000 and Vite on 5173; waits for both to be
healthy; prints a status table. `CTRL+C` shuts both down in reverse order. Logs
go to `logs/startup.log`, `logs/backend.log` and `logs/frontend.log`.

It does **not** run migrations or seed the database — steps 1 and 4 are yours to
do once.

Useful flags: `--backend-only`, `--frontend-only`, `--skip-install`,
`--install-only`, `--no-free-ports` (by default it kills whatever is holding
port 8000 or 5173), `--help`.

If you received `start.sh` by any route other than a git checkout and it fails
with `bad interpreter: /usr/bin/env bash^M`, that is CRLF line endings:
`sed -i 's/\r$//' start.sh`.

---

## Reading the documentation

`backend/docs/` is not a folder of loose markdown — it is a browsable
documentation site (MkDocs + Material), and it is the fastest way to understand
the system before changing it. Everything it needs is already installed by
`requirements.txt`, so there is nothing more to set up.

```bash
cd backend && source .venv/bin/activate    # Windows: .venv/Scripts/activate

mkdocs serve
```

Then open <http://127.0.0.1:8000>.

**Port 8000 is also the backend's port.** If the API is already running, serve
the docs somewhere else:

```bash
mkdocs serve -a 127.0.0.1:8080
```

For a static copy you can zip, mail or host anywhere:

```bash
mkdocs build          # writes backend/site/
```

What is in there:

| Section | What it covers |
| --- | --- |
| Getting Started | the shortest path from a fresh clone to a first answer |
| Architecture | the pipeline stage by stage — retrieval, the metadata filter, the grounding gate, observability |
| Reference | every configuration key and environment variable, and what each one does |
| Deployment | running it as a serverless function |

The navigation lives in `backend/mkdocs.yml` and the pages themselves are plain
markdown under `backend/docs/`, so they are readable in an editor too if you
would rather not run the server.

---

## Troubleshooting

**`/health` says `"store": "files (supabase requested but not configured)"`**
`SUPABASE_URL` or `SUPABASE_SERVICE_KEY` is missing or malformed in
`backend/.env`. The app fell back to reading markdown off disk. Fix and restart.

**`/health` says `store_reachable: false`**
`store_error` in the same response names the cause. Most often the project is
paused — a free Supabase project pauses after seven idle days and unpausing is
manual in the dashboard.

**`corpus_docs: 0` with `store_reachable: true`**
The database is up but empty. You have not run step 4, or it wrote nothing.

**`POST /ask` returns 503**
No `GROQ_API_KEY`, or it is invalid. Retrieval and the refusal paths still work
without it; only the grounding step needs a model.

**Answers come back but are visibly worse than they should be**
Check `select count(*) from chunks where embedding is null;` — if that is not 0,
the dense arm is dead and you are getting lexical-only retrieval. Re-seed with
`--force` once `GEMINI_API_KEY` is valid.

**A vector dimension error from Postgres**
The schema is `vector(768)`, matching Gemini. Do not point `EMBEDDER=local` at
this database — bge-small produces 384 dimensions and will not fit the column.
In dev mode leave `EMBEDDER` to the profile.

**The same question returns an identical answer instantly, ignoring a change**
The dev profile enables the answer cache (`ANSWER_CACHE: true` in `dev.json`),
served from the `answer_cache` table with a 60-second TTL by default. Wait it
out, set `ANSWER_CACHE_TTL_S=0`, or `truncate answer_cache;`.

**`429` from Groq**
The free tier is roughly 30 requests and 8k tokens a minute. `POST /ask` is also
rate limited per client IP at 20/minute (`ASK_RATE_LIMIT_PER_MINUTE`, `0`
disables). The client retries with backoff up to `LLM_MAX_RETRIES`.

**No traces in Langfuse**
In order: is `langfuse_configured` true in `/health`; does `LANGFUSE_HOST` match
the key's region; is `LANGFUSE_TRACING_ENABLED` unset or true; are you looking at
the `dev` environment in the Langfuse UI. Failures here are silent by design.

**The UI cannot reach the API**
`VITE_API_URL` in `frontend/.env` must match the backend port, and Vite must be
restarted after changing it.

**`python -m venv` fails on Debian/Ubuntu**
`sudo apt install python3-venv`

---

## Reference

- `backend/.env.example` — every variable, with the reasoning for each
- `backend/config/dev.json` — the dev profile's non-secret settings (thresholds,
  model roles, retrieval mode); committed, and overridden by any real
  environment variable
- `backend/supabase/migrations/` — the schema, each file explaining why it exists
- `backend/docs/` — the full documentation site; `mkdocs serve` to browse it
- `backend/requirements.txt` — every Python dependency, grouped and commented
- `README.md` — what the system does and how it is evaluated
