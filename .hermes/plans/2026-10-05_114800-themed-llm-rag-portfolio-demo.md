# Themed LLM Portfolio Demo (RAG + Data Pipeline + Java OMS) — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** A themed-equity-portfolio demo: user picks an investment theme on a website, an LLM (called through FastAPI) builds and periodically rebalances the portfolio from free daily market data, a Java OMS applies fills, everything runs on AWS in a single account.

**Architecture:** Batch, daily-candle shaped — no streaming market data anywhere. One Postgres (Aurora Serverless v2 PostgreSQL in-VPC, pgvector 0.8.x; 9,193-symbol universe seeded with 10-year bars) holds market data, rag corpus, and portfolio state. One SQS queue carries order intents from FastAPI to the Java OMS. ElastiCache Redis holds session/ephemeral state only. React+Vite static site on S3/CloudFront talks only to FastAPI; LLM credentials never leave the API tier.

**Tech stack:** Python 3.12 (FastAPI, boto3, asyncpg, httpx), Java 21 + Spring Boot 3 (AWS SDK v2), React + Vite + TypeScript + recharts, Postgres 16 + pgvector, Redis, SQS, EventBridge Scheduler, CDK (Python), docker-compose for local dev parity.

---

## 1. Requirements as understood

1. Website: create a portfolio from an investment theme; view holdings, P&L, transaction history.
2. LLM builds the portfolio from market data; **LLM calls go through FastAPI** (frontend never touches a provider).
3. Java OMS updates holdings for the portfolio account from orders.
4. LLM updates periodically (scheduled) and on forced trigger; updates may **substitute** tickers or **rebalance** shares.
5. Market data in Postgres; session data in ElastiCache.
6. Single AWS account.
7. (added 2026-10-05) User table + token-based auth for signing into the site; the token is verified at the OMS and LLM endpoints too (carry-through — §12 defines exactly what that means per hop).

Requirement-7 note: "OMS endpoint" = the signed order intent (the OMS is SQS-only per §3); "LLM endpoints" = the FastAPI admin/update routes that trigger LLM runs — the token itself never crosses to Bedrock, which authenticates via IAM.

Demoted by scope decision (medium/long-term horizon, daily candles): all sub-daily data, intraday fills, streaming ingestion. **Fills are mock fills at the daily close price** — disclosed in UI ("paper trading simulation").

## 2. Data plan (all free tiers, verified Oct 2026)

| Source | Use | Cadence | Limit |
|---|---|---|---|
| Stooq CSV (`stooq.com/q/d/l/?s=aapl.us&i=d`) | whole-universe seed backfill, 30+ yrs daily bars | one-time | none documented; be polite (1 req/s, UA header) |
| Massive (ex-Polygon) Stocks Basic | nightly EOD incremental | nightly | 5 calls/min free — fine for 10-20 tickers |
| Finnhub free | company news, EPS surprises | nightly | 60 calls/min, 1 yr news history |
| SEC EDGAR | 10-K/10-Q text chunks (RAG corpus), XBRL facts | nightly | fair-access ~10 req/min w/ UA |
| FRED | macro regime series (DGS10, CPI, etc.) | weekly | free key, trivial |
| Motley Fool archive | earnings-call transcripts (RAG corpus) | stretch/phase 2 | public pages, personal-use label in UI |

Licenses: most free tiers are personal/non-commercial — the demo runs behind one login, footer discloses data sources and "paper trading simulation". `MarketDataProvider` protocol so free Massive can swap for paid Massive/Bloomberg later without touching pipelines (that swap story is part of the demo narrative).

## 3. Decisions already made (in conversation)

- **Frontend:** React + Vite static site on S3 + CloudFront (user-selected).
- **API→OMS:** SQS, one direction only (user-selected). OMS writes truth to Postgres; API reads fills/status from Postgres. No reply queue.
- **LLM:** default **AWS Bedrock** (Converse API) — matches single-account constraint, IAM auth, bank-governance talking point; behind `LlmClient` interface so OpenAI/Ollama are config swaps. *Open question 1 below.*
- **Vector store:** pgvector in the same Postgres (not OpenSearch) — market data and embeddings live together per requirement 5; OpenSearch would add an extra stateful system for zero demo value.
- **Scheduled LLM updates:** EventBridge Scheduler → HTTPS call to FastAPI internal endpoint (API key in Secrets Manager), not a Lambda re-implementing logic. Same code path for scheduled + forced.
- **Java OMS:** Spring Boot 3 + AWS SDK v2 polling loop (receive_message, 10-parallelism via `ThreadPoolTaskSubscriber`-style executor), no spring-cloud-aws dependency (fewer version landmines).

## 4. Repo layout (exact files)

```
fintech-llm-rag-demo/
├── docker-compose.yml            # postgres16+pgvector, redis7, localstack (SQS), frontend:5173
├── Makefile                      # make up, seed, nightly, test-all, deploy, destroy
├── shared/contracts/
│   ├── order-intent-v1.json      # single source of truth for both producers/consumers
│   └── README.md
├── ingestion/                    # python package "ingestion"
│   ├── pyproject.toml
│   ├── src/ingestion/
│   │   ├── fetchers/{stooq.py,massive.py,finnhub_news.py,edgar.py,fred.py}
│   │   ├── loaders/{bars.py,news.py,filings.py,macro.py}
│   │   ├── embeddings.py         # bedrock titan-embed-text-v2 → filings_chunks/news_chunks
│   │   └── jobs/{backfill_seed.py,nightly_incremental.py,embed_refresh.py}
│   └── tests/{test_parse_stooq.py,test_parse_massive.py,fixtures/...}
├── api/                          # python package "api" (FastAPI)
│   ├── pyproject.toml
│   ├── src/api/
│   │   ├── main.py               # app factory, CORS, /health
│   │   ├── settings.py           # pydantic-settings, env-driven
│   │   ├── db.py, models.py      # SQLModel/sqlalchemy core, DDL in migrations/0001_init.sql
│   │   ├── repos/{portfolios.py,orders.py,bars.py,corpus.py}   # all SQL lives here
│   │   ├── llm/{base.py,bedrock.py,fake.py}                    # FakeLlmClient for tests
│   │   ├── services/{portfolio_builder.py,pnl.py,update_scheduler.py,retriever.py,publisher.py}
│   │   └── routers/{portfolios.py,insights.py,admin.py}        # admin = scheduled+force endpoint
│   └── tests/{test_create_portfolio.py,test_force_update.py,test_pnl.py,conftest.py(fakes)}
├── oms-java/
│   ├── pom.xml
│   ├── src/main/java/demo/oms/
│   │   ├── OmsApplication.java
│   │   ├── sqs/OrderIntentConsumer.java        # SDK v2 poll loop, validates contract JSON
│   │   ├── core/OrderService.java              # the only business logic
│   │   ├── persistence/{OrderRepo,PositionRepo,ExecRepo}.java
│   │   └── health/HealthController.java
│   ├── src/main/resources/application.yml
│   └── src/test/java/demo/oms/{OrderServiceTest.java,ContractValidationTest.java}
├── frontend/
│   ├── package.json, vite.config.ts, index.html
│   └── src/{App.tsx,api/client.ts,pages/{CreatePortfolioPage.tsx,PortfolioDetailPage.tsx},
│           components/{HoldingsTable,PnlChart,TransactionHistory,UpdateRunLog,InsightsPanel}.tsx}
├── infra/                        # CDK python app
│   ├── app.py, cdk.json
│   └── stacks/{network_stack.py,data_stack.py,service_stack.py}
└── docs/{runbook.md,demo-script.md,disclosures.md}
```

## 5. Core schema (Postgres 16 + pgvector)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE instruments (        -- reference data
  symbol TEXT PRIMARY KEY, name TEXT NOT NULL, sector TEXT, active BOOL DEFAULT true);

CREATE TABLE daily_bars (         -- THE market-data table (daily candles only)
  symbol TEXT NOT NULL, trade_date DATE NOT NULL,
  open NUMERIC(12,4), high NUMERIC(12,4), low NUMERIC(12,4), close NUMERIC(12,4) NOT NULL,
  volume BIGINT, source TEXT NOT NULL,              -- 'stooq' | 'massive'
  PRIMARY KEY (symbol, trade_date));

CREATE TABLE corpus_chunks (      -- filings + news + transcripts for RAG
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source TEXT NOT NULL,           -- 'edgar_10k' | 'edgar_10q' | 'finnhub_news' | 'transcript'
  symbol TEXT, published_at DATE, url TEXT,
  section TEXT,                   -- e.g. 'risk_factors', 'mdna', 'qa'
  content TEXT NOT NULL,
  embedding vector(1024) NOT NULL);                -- titan-embed-text-v2:0
CREATE INDEX corpus_hnsw ON corpus_chunks USING hnsw (embedding vector_cosine_ops);

CREATE TABLE portfolios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  theme TEXT NOT NULL, created_at TIMESTAMPTZ DEFAULT now(),
  cash NUMERIC(14,2) NOT NULL DEFAULT 100000,
  status TEXT NOT NULL DEFAULT 'PENDING_BUILD');   -- PENDING_BUILD|ACTIVE

CREATE TABLE portfolio_runs (     -- every LLM run, initial/scheduled/forced
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id UUID NOT NULL REFERENCES portfolios(id),
  trigger TEXT NOT NULL,          -- 'initial' | 'scheduled' | 'forced'
  rationale JSONB NOT NULL,       -- LLM output: reasoning, per-symbol narrative, substitutions
  proposed JSONB NOT NULL,        -- target weights/holdings before orders
  status TEXT NOT NULL);          -- PROPOSED|EXECUTING|DONE|FAILED

CREATE TABLE positions (
  portfolio_id UUID NOT NULL, symbol TEXT NOT NULL,
  qty NUMERIC(14,4) NOT NULL DEFAULT 0, avg_cost NUMERIC(12,4) NOT NULL DEFAULT 0,
  PRIMARY KEY (portfolio_id, symbol));

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_order_id TEXT NOT NULL UNIQUE,           -- idempotency key = intent UUID
  portfolio_id UUID NOT NULL, symbol TEXT NOT NULL,
  side TEXT NOT NULL CHECK (side IN ('BUY','SELL')),
  qty NUMERIC(14,4) NOT NULL, reason TEXT NOT NULL, -- INITIAL_BUILD|REBALANCE|SUBSTITUTION
  status TEXT NOT NULL DEFAULT 'NEW',               -- NEW|FILLED|REJECTED
  created_at TIMESTAMPTZ DEFAULT now());

CREATE TABLE executions (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id),
  symbol TEXT, qty NUMERIC(14,4), price NUMERIC(12,4),
  executed_at TIMESTAMPTZ);       -- mock fill price = that symbol's last daily_bars.close
```

Transaction history UI = `executions JOIN orders` (buys, sells, avg-cost P&L from `positions`). Session cache (Redis): UI session state + last-run status (`run:{portfolio_id}` → run id/status with 60s TTL) so the UI can poll "is the LLM working".

## 6. Order contract — `shared/contracts/order-intent-v1.json`

Validation schemas ship in this one file, referenced by both python producer and Java consumer:

```json
{
  "schema_version": 1,
  "client_order_id": "uuid",
  "portfolio_id": "uuid",
  "symbol": "AAPL",
  "side": "BUY",
  "qty": 12,
  "order_type": "MARKET_EOD_CLOSE",
  "reason": "REBALANCE",
  "llm_run_id": "uuid-or-null",
  "requested_at": "2026-10-05T21:00:00Z"
}
```

OMS behavior per message: validate → upsert order row `NEW` (conflict on client_order_id = skip, idempotent) → fill at last `daily_bars.close` for the trade date (no bar → REJECTED with reason) → insert execution → upsert position (qty/avg_cost, WAC) → set order `FILLED`. Net-effect only: the builder service emits *delta* orders from proposed−current positions.

## 7. LLM boundary (through FastAPI, per requirement 2)

- `LlmClient` (interface): `embed(texts) -> vectors`, `chat(system, user, schema) -> json`.
  - `BedrockLlmClient`: `bedrock-runtime` Converse API + Titan Embeddings v2; model ids in settings.
  - `FakeLlmClient`: deterministic fixture output — all tests run against it, zero cost, no flaky CI.
- **Portfolio build** (`POST /portfolios {theme, cash}`): retrieve top-k corpus chunks by theme-similarity + a metrics table (12-month return, 3-month drift, vol, latest EPS surprise from bars+corpus) → LLM → pydantic-validated `ProposedPortfolio{holdings:[{symbol, weight, narrative}], substitutions_policy}` → net-order deltas → publish intents → 202 + run id.
- **Periodic update** (`POST /admin/run-updates?portfolio_id=…`, EventBridge weekly + user "Force update"): same retrieval, prompt compares current holdings vs. theme, may return `SUBSTITUTION` (sell A, buy B) or `REBALANCE` (weight drift > 5pp triggers trades). Rationale stored in `portfolio_runs.rationale` — the UI shows "why the LLM changed things", which is the demo's money shot.
- **Insights RAG** (`GET /portfolios/{id}/insights` or chat panel): pgvector retrieval over corpus_chunks joined with holdings → answer with citations (source, symbol, date) rendered in UI.
- Rate-limit reality built into settings: Massive 5/min (`THROTTLE_SECONDS=13`), handled in the massive fetcher, not ad hoc.
- **Auth boundary (§12):** all routers except `/auth/*` and `/health` require Bearer JWT (RS256, `users` table, owner-or-admin); `/admin/run-updates` accepts a user JWT **or** the scheduler internal key; order intents carry signed actor claims the OMS verifies before applying; Bedrock is IAM-authenticated by the task role — user tokens stop at the API tier.

## 8. Implementation phases & tasks

Each task: write failing test → minimal impl → pass → commit. Commands from repo root. Python tests: `cd api && pytest -x -q` / `cd ingestion && pytest -x -q`; Java: `cd oms-java && mvn -q test`; frontend: `cd frontend && npm test`.

### Phase 0 — Skeleton (2-3 tasks)
1. Repo init, git, `Makefile`, `docker-compose.yml` (postgres+pgvector via `ankane/pgvector`, redis, localstack SQS), env templates `.env.example`. Verify: `docker compose up -d && docker compose exec postgres pg_isready`.
2. DDL `api/migrations/0001_init.sql` (schema above) + `make initdb`. Verify: psql `\d daily_bars`.
3. Contract file `shared/contracts/order-intent-v1.json` + JSON Schema validation copies for py (import) and java (Everit/json-utils).

### Phase 1 — Ingestion (5-7 tasks)
4. `fetchers/stooq.py` + parse test against `tests/fixtures/aapl_us.csv` (no network in CI). Verify live once: `curl 'https://stooq.com/q/d/l/?s=aapl.us&i=d'` with UA header.
5. `loaders/bars.py` (upsert on (symbol,trade_date), source tag) + test.
6. `jobs/backfill_seed.py`: seed 10-20 tickers (universe list in `ingestion/universe.yaml`), 1 req/s. Verify: row counts ≥ 500 days for AAPL in local PG.
7. `fetchers/massive.py` nightly incremental (env key, 5/min throttle) + parse test on fixture JSON. Register: free account at massive.com for key.
8. `fetchers/finnhub_news.py` + `loaders/news.py` (dedupe on url) + tests. Register: free Finnhub key.
9. `fetchers/edgar.py` via `edgartools` (10-K/10-Q → section chunks) + fixture test. `fetchers/fred.py` (DGS10, CPALTT01) + test.
10. `embeddings.py` + `jobs/embed_refresh.py` (titan embed → corpus_chunks) wired behind `LlmClient.embed`; test with fake embedder (deterministic hash vectors).

### Phase 2 — FastAPI core (6-8 tasks)
11. `settings.py`, `db.py`, repos, `main.py` with `/health`, CORS. Test: httpx TestClient `/health`.
12. `repos/bars.py` price queries + `services/pnl.py` (unrealized = close−avg_cost×qty; realized on sells) + unit tests on fixture bars.
13. `llm/base.py` + `fake.py` (fixture `ProposedPortfolio` JSONs) `bedrock.py` behind env flag. Test: fake proposes portfolio for theme "AI infrastructure".
14. `routers/portfolios.py` create-portfolio flow w/ mocked `publisher.py` (fake SQS) → asserts 202, run row PROPOSED→EXECUTING, N intents sent. TDD here first, logic after.
15. `services/portfolio_builder.py`: retrieval → metrics → prompt → validate → delta-orders. Prompt stored as `api/src/api/prompts/build_portfolio.md` (versioned, reviewable — talking point).
16. Force-update + scheduled endpoints (`admin.py`), shared `update_scheduler.py`; Redis run-status caching. Tests with fakeredis + fake SQS.
17. `routers/insights.py` RAG query path (pgvector KNN + join holdings) — test with fake embeddings, fixture corpus.
18. `docs/` API contract for the frontend (openapi.json exported).
18a. **JWT auth (§12):** `users` DDL + ALTERs, argon2id hashing, `/auth/register|login|logout|me`, RS256 issue/verify `kid=v1` (private key Secrets Manager, public key SSM→OMS), `get_current_user` dependency, ownership enforcement (404-not-403), Redis jti denylist + login lockout. Tests: round-trip, lockout, owner-only, denylist, admin dual-auth paths.

### Phase 3 — Java OMS (4-5 tasks)
19. Spring Boot skeleton, `application.yml` (SQS queue URL, PG DSN), `/health`. Verify: `mvn test` + boot against compose.
20. `ContractValidationTest`: order-intent-v1 JSON fixtures valid/invalid.
21. `OrderService` TDD: idempotent order insert, close-price fill (test PG via testcontainers-postgres), position WAC math, REJECTED-on-no-bar path.
22. `OrderIntentConsumer` poll loop (SDK v2), delete-on-success, retry-without-delete on transient DB error, dead-letter on poison JSON. Unit test consumer with mocked `SqsClient`.
23. Compose-level smoke: seed bars → publish intent via redis-cli? no — via api `/portfolios` create with FakeLlm → watch orders become FILLED → positions updated. This is "the demo works locally" gate.
22a. **Signed-intent verification (§12):** nimbus-jose-jwt verify of `actor` sig (api public key via SSM) in `OrderIntentConsumer` before apply; sig/parse failure → explicit DLQ copy + delete + alarm; `orders.acting_user_id` recorded. Tests: valid sig passes, tampered payload → DLQ path.

### Phase 4 — Frontend (4-5 tasks)
24. Vite scaffold + `api/client.ts` typed from openapi.json; vitest on client.
25. `CreatePortfolioPage` (theme textarea, cash, submit → run poll).
24a. **Login flow (§12):** `LoginPage` + register; fetch wrapper auto-Bearer, 401 → redirect; route guards on both portfolio pages; token in localStorage (demo tradeoff documented in talking-points).
26. `PortfolioDetailPage`: HoldingsTable, PnlChart (recharts, equity curve from daily_bars + positions), TransactionHistory, UpdateRunLog (rationale JSON → human cards), "Force update" button, InsightsPanel with citations. Poll run status every 5s while EXECUTING.
27. Empty/error states (PENDING_BUILD spinner, REJECTED orders shown in red with reason).

### Phase 5 — AWS single-account deploy (5-6 tasks, CDK)
28. `network_stack.py`: VPC 2 AZs, public subnets (ALB), private (ECS+Aurora+Redis), SGs (ALB→api:8000, api→oms:8080, api/oms→pg:5432 restored, api/oms→redis:6379; only ALB public). **OpenRouter is the one external SaaS** (egress to api.openrouter.ai; Bedrock/Aurora/SQS all in-account).
29. `data_stack.py`: **Aurora Serverless v2 PostgreSQL cluster (Aurora Standard, us-east-1, user decision — §14)** — NOT Neon/RDS-provisioned; ServerlessV2ScalingConfig min 0 / max 2-4 ACUs; pgvector enabled at init + job running 0001_init.sql (10-year seed default); ElastiCache Redis (cache.t3.micro or serverless small), SQS standard queue + DLQ, Secrets Manager (Aurora creds, OpenRouter key, Finnhub/Massive tokens), SSM parameter for embed model id.
30. `service_stack.py`: 2× ECR repos; ECS Fargate api + oms tasks (0.25 vCPU/0.5 GB), ALB → api only; IAM: api task role (sqs:SendMessage, bedrock:Invoke*, secrets get), oms task role (sqs:Receive+Delete, pg via SG), EventBridge Scheduler rule → managed HTTPS target hitting `POST /admin/run-updates` w/ API key from Secrets Manager; S3 + CloudFront (OAC) for frontend build output.
31. CDK deploy + initdb on Aurora + ECR image pushes (Make targets `deploy`, `migrate`, `seed-aws`). Verify checklist in `docs/runbook.md`: health via ALB endpoint, create portfolio end-to-end in prod, EventBridge fires (set 1-off schedule, watch CloudWatch logs).
32. `docs/demo-script.md` + `disclosures.md`: paper-trading note, free-tier data sources + personal-use labels, mock-fill note. Teardown: `make destroy` (single stack delete + ECR/S3 cleanup; Aurora final snapshot retained ~$0). Keep the account bill at demo-only level (**~$40-60/mo including Aurora while deployed, ≈$0 idle-state after teardown**; verify at deploy).

### Phase 6 — Hardening (stretch, post-demo-v1)
- Production-path auth swap to Cognito (JWT §12 → Cognito mapping documented), MF transcripts ingestion, FRED regime gating in prompt, universe config UI, paid-Massive swap runbook page, always-on DB revisit (Aurora always-on is a different cost conversation than demo-window).

## 9. Risks / tradeoffs / open questions

| # | Item | Position |
|---|---|---|
| R1 | ~~LLM provider unconfirmed~~ **Decided → superseded by §14** | Chat: OpenRouter `z-ai/glm-5.3-flash` ($0.0352/$0.50 per 1M [28]); Bedrock retains embeddings only; DB: **Aurora Serverless v2 in-VPC** (user decision, demo-window cadence ≈$2/mo, note 09 v1.2) |
| R2 | Mock fills ≠ real prices intraday | Disclosed; fills at close only. Not a broker |
| R3 | Massive 5/min free ceiling | 13 s throttle; nightly = active subset only (portfolio members + top-50); full-universe tail weekly via Stooq bundle re-pull (§13, note 08) |
| R4 | Data licenses are personal-use; MF/yfinance scraping gray zone | Footer disclosures; internal demo only; adapter layer ready for licensed swap |
| R5 | Free Finnhub news = 1 yr history only | Corpus depth from EDGAR (2001→) + transcripts stretch compensates |
| R6 | pgvector scale | Fine at demo corpus (~10⁴–10⁵ chunks, HNSW); note in docs |
| R7 | Single account blast radius | All resources in one stack; teardown documented; SGs private-first |
| Q1 | **Decided 2026-10-05:** Bedrock default = `zai.glm-4.7-flash` (Bedrock catalog has no GLM-5.3-Flash — only GLM 4.7 / 4.7 Flash / GLM 5 [22]); `ZaiLlmClient` direct-API fallback for GLM-5.3-Flash at $0.15/$0.50 per 1M [23]; details §13 |
| Q2 | **Decided 2026-10-05:** all free NASDAQ+NYSE listings ≈ 9,193 common stocks (NasdaqTrader symbol files × Stooq daily bundle [24][25]); tiered freshness — portfolio members + top-50 nightly, tail weekly via Stooq re-pull; details §13 |
| Q3 | **Decided 2026-10-05:** DIY JWT (RS256) + `users` table, signed actor intents verified by OMS; Cognito = production swap (§12) |
| Q4 | **Decided 2026-10-05:** us-east-1 (GLM 4.7-Flash and GLM 5 model cards both In-Region available; near Philly) |

## 10. Tests / verification summary

- **Python:** pytest unit + TestClient flows (all LLM/SQS faked); parse tests run on fixtures, never network.
- **Live smoke gates (manual, once per phase):** Stooq curl, one Massive call, one Finnhub call, one Bedrock embed+converse call, one EDGAR filing pull — each a `make verify-*` one-liner in the runbook.
- **Java:** mvn unit (Service, contract validation) + testcontainers PG for the fill/WAC math; consumer unit-tested with mocked SqsClient.
- **End-to-end gates:** (a) compose: create "AI infrastructure"/"$100k" portfolio → holdings appear after fake-LLM build + OMS fill; (b) force update yields substitution or rebalance with visible rationale; (c) insights answer cites at least one corpus chunk; (d) deployed: same (a-c) against ALB/CloudFront, EventBridge fired once at a set time.
- **P&L correctness:** property test — sum(realized P&L + unrealized P&L + cash) equals initial cash ± rounding, invariant from fixture data.

## 11. Data pipeline scheduling (decision 2026-10-05)

**Schedule-based end to end; events reserved for order intents (SQS).** Rationale: all chosen free feeds are poll-only with fixed publication cadence (no push on free tiers — Massive WS needs paid Starter; Finnhub free WS is intraday/out-of-scope); workload is uniform, idempotent, hours-tolerant; EventBridge already owns LLM-update scheduling, so one timing mechanism total. *"Clock for data, events for orders."*

| Job | Cadence (ET) | Contents |
|---|---|---|
| `ingestion seed` | manual | Stooq whole-universe backfill (≤1 req/s) |
| `ingestion nightly` | 20:00 Mon–Fri (EventBridge Scheduler → ECS RunTask) | Massive bars for **active subset** (portfolio members + top-50 liquid), lookback 5 sessions, 13 s throttle → `daily_bars` upsert; Finnhub news; EDGAR filings for portfolio members → corpus; stage 2 embeddings in same run |
| `ingestion reconcile` | Sat 03:00 | **Stooq daily-bundle re-pull for the full-universe tail** (~9k tickers, T+2-3 sessions freshness; corporate actions/restatements absorbed), FRED releases, derived metrics |

Robustness: **lookback windows, not cursors** — every run re-pulls a trailing window and upserts, so failures self-heal next run with no high-water-mark state. Reversal triggers (documented, falsifiable): intraday scope gains a push source; a second consumer needs same-landing fan-out; SLA < 15 min. Full design: Trilium note 05 (child of YNPTpI1SaQas).

## 12. Authentication & token carry-through (added 2026-10-05; settles Q3)

**DIY JWT (RS256) + `users` table; Cognito = production swap.** Access token 12 h, `kid=v1`, jti; private key in Secrets Manager, public key to OMS via SSM (JWKS rotation deferred). Passwords argon2id; lockout + jti denylist in Redis (`deny:jti:{jti}` — OMS doesn't consult Redis; documented revocation lag).

New tables/columns: `users(id,email UNIQUE,password_hash argon2id,display_name,role 'user'|'admin',created_at)`; `portfolios.owner_user_id`, `portfolio_runs.initiated_by` (NULL = scheduler), `orders.acting_user_id` (from signed intent).

Verification matrix:

| Hop | Verification |
|---|---|
| Browser → api | JWT: RS256 sig + exp/aud/iss + Redis denylist + fresh user-role load |
| api → OMS (SQS) | intent gains `actor {sub, role, token_kid}` + `sig` (RS256 over canonical JSON minus sig); OMS: parse → verify (nimbus-jose-jwt) → contract-validate → idempotent apply; failure = explicit DLQ copy, delete, alarm — never in-place retry |
| admin / LLM-trigger routes | user JWT (owner/admin) OR scheduler `X-Internal-API-Key`; audited which was used |
| api → Bedrock | IAM task role; **no user token crosses to the LLM provider** |

Ownership = authorization: portfolio routes filter `owner_user_id == current_user` or admin; foreign portfolios → 404 (no existence leak). Frontend: LoginPage + register, auto-Bearer wrapper, 401→redirect, route guards; localStorage token for demo (XSS tradeoff documented; httpOnly cookie/Cognito = production alternatives). Demo seed: migration idempotently creates an `admin` user with a generated password printed once to the deployer (make target) — no hardcoded creds in git.

Full design: Trilium child note 06 (`d1eGZpF8NURY`, under YNPTpI1SaQas); schema additions appended to note 04, signed-intent amendment appended to note 02.

## 13. Provider & universe decisions (added 2026-10-05; settles Q1, Q2, Q4)

**LLM provider.** Bedrock's Z.AI catalog carries **GLM 4.7, GLM 4.7 Flash, GLM 5 — GLM-5.3-Flash is not offered on Bedrock** [22]. Since the requested model isn't there, `LlmClient` gets two concrete clients (provider = config, both in demo): **`BedrockLlmClient` → `zai.glm-4.7-flash`** (default; IAM auth, one-account, Flash = cheap tier, us-east-1 In-Region verified) and **`ZaiLlmClient` → GLM-5.3-Flash via Z.ai direct API at $0.15/$0.50 per 1M tokens, cached input $0.03** [23] (dev/iteration + fallback). Embeddings stay Titan Embeddings v2 via Bedrock in both paths. Demo inference cost ≈ cents per full build/update/insights cycle.

**Ticker universe = all free NASDAQ+NYSE listings.** Symbol truth from NasdaqTrader `nasdaqlisted.txt`/`otherlisted.txt` (anonymous FTP, pipe-delimited, updated daily) [25]; daily bars from the **Stooq daily bundle** — one ~515 MB US zip, per-ticker full-history CSVs: **nasdaq stocks 4,661 + nyse stocks 4,532 ≈ 9,193 common stocks** seeded in a single download (bundle updated nightly server-side) [24]. Nightly incremental via Massive (5/min) covers the **active subset** (portfolio members + top-50 liquid); the tail refreshes weekly via Stooq bundle re-pull — T+2-3 session freshness for non-portfolio names, disclosed in the data-freshness footer. `instruments` gains `etf`, `exchange`, `last_ingested_at`. ETFs (~3,812) = phase-3 stretch; OTC/pink = out of scope (not free).

Full designs: Trilium notes 07 (`wwpC2p5chbJ4`) and 08 (`JxYZv3kHsI99`), children of YNPTpI1SaQas.

## 14. External accounts: OpenRouter inference + Neon Postgres (added 2026-10-05 evening)

**LLM = OpenRouter (user's account), model `z-ai/glm-5.3-flash` — price verified [28]: $0.0352 in / $0.50 out per 1M tokens, cache read $0.0232, 1.0M context (cheapest routed provider).** Supersedes §13's model split: `LlmClient` → **`OpenRouterLlmClient`** (OpenAI-compatible, `https://openrouter.ai/api/v1`, key in Secrets Manager) is the single chat path; Bedrock now serves exactly one role — Titan Embeddings v2 for the corpus (pay-per-token cents; zero-Bedrock fallback = local BGE-small flag). Demo inference cost ≈ well under $0.05/mo. Requirement 2 intact: browser never touches OpenRouter.

**Postgres = Aurora Serverless v2 (user's account, in-VPC; supersedes §14's Neon paragraph — decision history in note 09).** Aurora Standard $0.12/ACU-hr with min 0 ACUs (pause) / max 2-4; ≈ **$2/mo at 4 demo windows/mo + tear-down between** (idle storage $0.23, ~$0.46/demo, backups free) [26][27]. 10-year seed default restored (`SEED_HISTORY_YEARS=10`; 23M rows ≈ 2.3 GB). pgvector verified: Aurora PG 16.8+/17.4+, ext 0.8.2, HNSW (m=16, ef_construction=128) + iterative scans [31][32]. DB inside the VPC again: SG rules api/oms→pg:5432 restored; connect-timeout >15 s + retry (resume <15 s) [27]. Neon recorded as the evaluated alternative (free $0, but in-VPC/single-vendor wins under the user's deploy-for-demo model).

**Requirement-6 reading (updated twice, current):** one AWS account — app tier, SQS, ElastiCache, **and Aurora DB now in-account**; only OpenRouter is external (inference). The DB bill under deploy↔teardown demo cadence ≈ $2/mo; teardown (`make destroy`) drops the cluster, retention between demos optional via ~$0 snapshots. Revised total AWS bill ≈ **$40-60/mo incl. Aurora** (only while deployed).

Full design: Trilium child note 09; supersedence recorded in notes 07, 04, 05.