# Fintech LLM RAG Portfolio Demo

Demo: a website where a user picks an investment theme, an LLM builds and periodically
rebalances an equity portfolio from free daily market data (called **through FastAPI** —
the browser never touches a model provider), and a Java OMS applies mock fills. All
market data is EOD/daily-candle; the horizon is medium/long-term, not intraday.

**Status: planning complete, not yet implemented.** The implementation plan lives at
[`.hermes/plans/2026-10-05_114800-themed-llm-rag-portfolio-demo.md`](.hermes/plans/2026-10-05_114800-themed-llm-rag-portfolio-demo.md)
(28 numbered tasks across 6 phases, all open decisions recorded with their rationale).

## Architecture (decided)

| Concern | Choice |
|---|---|
| Frontend | React + Vite + TypeScript, static on S3/CloudFront |
| API / LLM boundary | FastAPI; all model calls server-side |
| Chat LLM | OpenRouter — `z-ai/glm-5.3-flash` ($0.0352/$0.50 per 1M tok) |
| Embeddings | AWS Bedrock Titan Embeddings v2 (1024-d), Bedrock's only role |
| Vector store | pgvector (HNSW, cosine) inside the main Postgres |
| Database | **Aurora Serverless v2 PostgreSQL** (min 0 / max 2-4 ACUs), in-VPC |
| Order flow | FastAPI → SQS (`order-intent-v1`, signed actor claims) → Java OMS → Postgres truth |
| OMS | Java 21 + Spring Boot 3, SQS poller, mock fills at daily close |
| Sessions | ElastiCache Redis only (ephemeral, JWT denylist, run-status) |
| Deploy | CDK, one AWS account; deploy ↔ teardown per demo window (~$40-60/mo deployed, ≈$0 down) |

## Data plan (all free tiers, verified Oct 2026)

- **Seed:** Stooq daily bundle — one ~515 MB US zip; 4,661 NASDAQ + 4,532 NYSE stocks ≈ **9,193 tickers with 10-year daily bars**
- **Nightly incremental:** Massive (ex-Polygon) free tier for active subset (portfolio members + top-50), 5 calls/min throttled
- **Weekly tail refresh:** Stooq bundle re-pull (T+2-3 session freshness for non-portfolio names)
- **RAG corpus:** SEC EDGAR (10-K/10-Q, XBRL), Finnhub company news (free, 1 yr); transcripts via Motley Fool archive
- **Macro:** FRED (rates, CPI) for regime context
- Universes/symbols truth: NasdaqTrader symbol directory (anonymous FTP)

## Repo layout (planned)

`shared/contracts/` (order-intent schema) · `ingestion/` (python) · `api/` (FastAPI) ·
`oms-java/` (Spring Boot) · `frontend/` (React) · `infra/` (CDK) · `docs/`.

## Disclosures

Paper-trading simulation (fills at daily close, not a broker). Market-data sources are
free/personal-use tiers, disclosed in-app. Mock fills; LLM content may be wrong —
insights carry citations.