# Build checklist

Reference: [client brief](client-brief.md) · [architecture](architecture.md)

## Where to start

**Not frontend-first, not backend-only — foundation first, then backend-heavy, then frontend on top of a working API.**

| Order | Why |
| ----- | --- |
| **1. Accounts + Supabase** | Auth, Postgres, and `pgvector` are shared by everything. Nothing else can wire up without a project. |
| **2. Backend schema + migrations** | Alembic owns the data model. Ingestion, retrieval, chats, and citations all depend on tables existing. |
| **3. Ingestion + retrieval (backend)** | The client brief is about *trusted, cited answers from filings*. Corpus in the DB and hybrid search are the product core — prove this before polishing UI. |
| **4. Auth + thin vertical slice** | Once the backend can verify a JWT and return a stub stream, scaffold the frontend chat shell and connect the two. |
| **5. LLM + grounding + citation UI** | PydanticAI agent, citation validation, then the frontend surfaces that make analysts trust answers. |
| **6. Deploy + pilot** | Railway, env config, and the 5-analyst pilot from the definition of done. |

Frontend and backend scaffolds can be started in parallel early, but **don't invest in chat UI before ingestion and retrieval work** — you'll be building against an empty corpus.

---

## Phase 0 — Prerequisites

- [x] Install Python 3.12+, [uv](https://docs.astral.sh/uv/), Node 20+, pnpm (`corepack enable`)
- [x] Create an [OpenAI API key](https://platform.openai.com/api-keys) (needed once embeddings/LLM are wired)
- [x] Skim [client brief](client-brief.md) and [architecture](architecture.md)

---

## Phase 1 — Supabase foundation

Guide: [supabase-setup.md](guides/supabase-setup.md)

- [x] Create a Supabase project (save DB password)
- [x] Collect Project URL, anon key, service_role key, project ref, **direct** DB connection string
- [x] Auth: email provider on; disable "Confirm email" for local dev if desired
- [x] Copy `backend/.env.example` → `backend/.env` and fill in Supabase + `DATABASE_URL`
- [x] Copy `frontend/.env.example` → `frontend/.env` and fill in Supabase + API URL

---

## Phase 2 — Backend scaffold

Guide: [backend-setup.md](guides/backend-setup.md)

**Already done:** `pyproject.toml`, `uv.lock`, Alembic skeleton (`alembic/`, `alembic.ini`)

- [x] `cd backend && uv sync`
- [x] Create `app/` package: `main.py`, `config.py` (fail fast on missing env)
- [x] Health route: `GET /health` returns OK
- [x] CORS from `ALLOWED_ORIGINS`
- [x] `uv run uvicorn app.main:app --reload` starts cleanly

---

## Phase 3 — Database schema

- [ ] SQLAlchemy models in `app/database/models.py` per [architecture](architecture.md#data-model):
  - `profiles`, `chat_threads`, `chat_messages`, `message_citations`
  - `source_documents`, `document_chunks` (embedding + `tsvector` columns)
- [ ] Wire `alembic/env.py` to app metadata + `settings.DATABASE_URL`
- [ ] Generate initial migration; **review** and add explicit ops for:
  - `create extension if not exists vector`
  - HNSW index on embeddings, GIN on `tsvector`
  - RLS enablement + policies
- [ ] `uv run alembic upgrade head` against Supabase

---

## Phase 4 — Sample corpus (local)

- [ ] Edit `USER_AGENT` in `data/download.py`
- [ ] `uv run data/download.py` from repo root (AAPL, MSFT, NVDA, AMZN, GOOGL 10-Ks 2021–2025)
- [ ] Confirm `data/downloads/` + `manifest.json` look right

---

## Phase 5 — Ingestion pipeline (backend)

This unlocks every example question in the client brief.

- [ ] `backend/ingest/`: parse downloaded filings → normalized Markdown
- [ ] Chunk with metadata (ticker, filing type/date, page/section, accession)
- [ ] Embed chunks via OpenAI; write `source_documents` + `document_chunks` to Supabase
- [ ] Idempotent re-run (skip or upsert already-ingested filings)
- [ ] Spot-check: a known passage from Apple 2024 10-K is queryable in the DB

---

## Phase 6 — Retrieval (backend)

- [ ] `app/retrieval/`: pgvector semantic search over `document_chunks.embedding`
- [ ] Postgres full-text search over `document_chunks.search_vector`
- [ ] Reciprocal Rank Fusion in Python to merge ranked lists
- [ ] Return `SourcePassage` objects with enough metadata to cite (company, filing, page, excerpt)
- [ ] Unit tests: fusion logic, passage shape, empty-corpus behavior

---

## Phase 7 — Auth (backend + frontend)

- [ ] Backend: `app/auth/dependencies.py` — verify `Authorization: Bearer <supabase_jwt>`, expose `get_current_user`
- [ ] Backend: reject unauthenticated requests before retrieval/LLM work
- [ ] Frontend scaffold (guide: [frontend-setup.md](guides/frontend-setup.md)):
  - Vite + React + TypeScript + Tailwind + shadcn/ui + React Router
  - `src/lib/env.ts`, `supabase.ts`, `http.ts`, `api.ts`
- [ ] Sign-up / sign-in / sign-out pages (email only)
- [ ] Protected chat route; unauthenticated users redirect to login

---

## Phase 8 — Chat vertical slice (stub → real)

Prove browser → FastAPI → Supabase wiring before the full agent.

- [ ] Backend: thread CRUD (`chat_threads`, `chat_messages`) scoped to `user_id`
- [ ] Backend: `POST /chat/stream` — stub assistant response, AI SDK-compatible stream format
- [ ] Frontend: `useChat` + `DefaultChatTransport` pointed at FastAPI with bearer token
- [ ] Frontend: thread list + message history load via `api.ts`
- [ ] End-to-end: sign in → new thread → send message → see streamed stub reply → refresh → history persists

---

## Phase 9 — LLM + grounding (backend)

The trust contract from the client brief.

- [ ] PydanticAI agent (`app/assistant/`): typed deps, `GroundedAnswer`, `Citation`, `SourcePassage`
- [ ] Agent instructions: answer only from retrieved passages; cite every claim; refuse when evidence is insufficient; no stock picks
- [ ] Bounded tools: `search_filings`, `read_chunk`, `read_surrounding_chunks`
- [ ] `app/grounding/validator.py`: every citation maps to a retrieved passage; fail closed on validation errors
- [ ] Wire orchestrator: retrieve → agent → validate → stream → persist messages + `message_citations`
- [ ] Unit tests: citation extraction, grounding enforcement, "insufficient evidence" path

---

## Phase 10 — Chat UI (frontend)

What analysts actually see.

- [ ] Message list with streaming status and errors
- [ ] Citation chips linking to source filing + page/section
- [ ] Expandable source passage panel (verify in one click)
- [ ] Empty state when corpus has no matches
- [ ] Past conversations sidebar / thread picker

---

## Phase 11 — Validate against client brief

Run the [example analyst questions](client-brief.md#example-analyst-questions) manually:

- [ ] Q1–5: cross-year / cross-segment questions return cited answers with passages
- [ ] Q6–9: comparative and risk-factor questions cite specific filings
- [ ] Q10: bot refuses to infer beyond filings when evidence is thin
- [ ] No hallucinated facts; wrong-but-confident answers are treated as bugs
- [ ] Each answer shows underlying passage for one-click verification

---

## Phase 12 — Deploy + pilot readiness

- [ ] Railway: backend service (Uvicorn) + frontend service (Vite static build)
- [ ] Production env vars on both services; `ALLOWED_ORIGINS` includes frontend URL
- [ ] Re-run ingestion against production Supabase (or promote from dev)
- [ ] Smoke test: sign in on deployed URL → ask one client-brief question → citations work
- [ ] Pilot: 5 senior analysts, one week — target ≥3 hours saved per analyst per week ([definition of done](client-brief.md#definition-of-done))

---

## Quick reference — current repo state

| Area | Status |
| ---- | ------ |
| Docs (brief, architecture, setup guides) | Done |
| `data/download.py` | Done |
| Backend deps + Alembic skeleton | Done |
| `backend/app/` (FastAPI service) | Not started |
| DB migrations / models | Not started |
| Frontend app (`src/`, `package.json`) | Not started |
| Ingestion, retrieval, agent, chat UI | Not started |
