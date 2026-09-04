Users provide amounts of money they are willing to invest each month.

The app

- ask questions related for them

- suggests best suitable funds for them to invest.

- show history and future growth of their investment portfolios with the provided answers.

Scope: educational/informational tool only, not real financial advice. Simulation only — no broker integration, no real trading, no money movement. No login; the questionnaire, portfolio, and chat all work anonymously within a browser session.

## Personalization

A fixed, static multiple-choice risk questionnaire (5-8 questions), not free text and not LLM-adaptive, covering:

- time horizon

- loss tolerance

- financial capacity (income stability, emergency fund)

- investment goal

- experience level

- ESG preference (optional)

Answers produce a numeric risk score (Conservative -> Aggressive), mapped to a target asset allocation (e.g. equity vs. bonds %). Fund selection fills that allocation from the seeded fund categories via deterministic, rule-based matching — not ML — so every recommendation is fully explainable ("here's exactly why user X got fund Y").

## Fund data

No free, reliable API exists for real Norwegian verdipapirfond NAV
history (they're not exchange-traded). Instead: real historical prices for globally-listed ETFs that act as proxies for common fund categories available to Norwegian investors (global equity index, Nordic/Oslo Børs index, emerging markets index, bond index), ingested via a `yfinance` script and seeded into Postgres. Labeled clearly in the UI everywhere as "representative of typical fund categories," not the real NAV of a specific Norwegian fund.

## Growth visualization

- History: the real ingested NAV series.

- Future: a Monte Carlo simulation from each category's historical mean return/volatility, shown as percentile bands (not a single promised line), with an explicit disclaimer next to the chart.

## RAG / LLM

Knowledge base: investing glossary, fund fact sheets, and Norwegian tax/account basics (ASK, aksjefond, fondskonto) at a conceptual level only, with a disclaimer pointing to Skatteetaten / a real advisor for exact figures. Chunked and embedded (local, open-source embedding model) into Postgres via the `pgvector` extension.

Two consumers share the same retrieval pipeline:

1. **"Why this fund" explanations** — after rule-based scoring picks candidate funds, retrieve the relevant fund fact-sheet chunks and feed them plus the user's profile to the LLM to generate the plain-language explanation shown next to each recommendation.

2. **Standalone Q&A chat widget** — free-text investing questions answered via retrieval over the same knowledge base.

Both are grounded generation with a "don't know" fallback when retrieval finds nothing sufficiently relevant, to limit hallucination.

The pipeline is architected the same way it would be for confidential documents (e.g. a user's own brokerage statements, a firm's proprietary research) — that's the standard industry reason RAG exists, since no amount of LLM web browsing reaches private data. The MVP itself only ingests public content; the architecture is simply ready to extend to private data later.

## Tech stack

- Frontend
  - React, Typescript
  - Recharts (or visx) for portfolio history / Monte Carlo growth charts
  - TanStack Query for server state / API data fetching
  - Typed API client generated from FastAPI's OpenAPI schema (openapi-typescript-codegen) for end-to-end type safety

- Database
  - PostgreSQL
  - pgvector extension (embeddings for RAG)
  - TimescaleDB extension, optional (fund NAV history time series)
  - SQLAlchemy + Alembic for ORM and migrations

- ML
  - LLM generation: a cheap hosted API model (e.g. Claude Haiku / GPT-4o-mini / Gemini Flash tier) — exact provider/model still to be decided, optimized for cost
  - Embeddings: a local, free, open-source embedding model (e.g. `sentence-transformers`) run on the same box — no per-call API cost
  - RAG via pgvector similarity search

- Backend
  - FastAPI
  - Gunicorn
  - Nginx

- Testing
  - pytest (backend)
  - Vitest + React Testing Library (frontend)
  - Playwright for one end-to-end golden-path test (questionnaire -> recommendation -> growth chart)

- DevOps
  - Docker (Docker Compose, single box)
  - Github Actions
  - AWS EC2 (not ECS/Fargate — simpler and cheaper for a demo)

## Public-deployment safeguards

Since the app is deployed with no login, LLM-backed endpoints (chat,
explanations) need:

- per-session / per-IP rate limiting
- a hard monthly spend cap/alert on the LLM provider dashboard
- a `max_tokens` cap per response

## Build order (MVP)

Vertical feature slices (DB -> backend -> frontend per feature), so there's always a working demo at each step, rather than building an entire layer at a time:

1. **Skeleton** — Docker Compose (FE + BE + Postgres), GitHub Actions running tests, one deploy to EC2, even with near-empty pages.

2. **Fund data** — yfinance ingestion script, SQLAlchemy models + Alembic migrations, seed fund categories into Postgres.

3. **Questionnaire -> recommendation** — multiple-choice form, scoring endpoint, rule-based allocation, recommended funds returned. Already a working core demo on its own.

4. **Charts** — historical NAV line + Monte Carlo future-growth bands.

5. **RAG explanations** ("why this fund"), using pgvector + knowledge base.

6. **RAG chat widget**, reusing the same pipeline as step 5.

7. **Polish** — rate limiting/spend caps, remaining tests, demo README/script for recruiters.

Steps 1-3 are deliberately "CRUD-ish" (standard data in/out plumbing, not yet ML-flavored): if time runs out, what exists is a complete, demoable app rather than several half-finished, disconnected pieces. The RAG/Monte Carlo differentiation is layered on afterward, once the foundation works.
