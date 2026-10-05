# Database schemas — preliminary draft

> **Status: preliminary draft (2026-10-05).** Source of truth at build time is
> `api/migrations/0001_init.sql` (plan §5). Aligns with plan §5 (core schema), §12
> (auth columns), §13 (universe columns), §11/§7 (macro + metrics loaders).

Per the decided architecture (plan §14), everything lives in **one Aurora
Serverless v2 PostgreSQL cluster (Postgres 16, pgvector 0.8.x)** — one database.
The four "databases" below are **PostgreSQL schemas** (namespaces) inside it:
`securities`, `accounts`, `oms`, `portfolios`. Schemas give clean domain
boundaries (and would be the split lines if a physical separation were ever
needed); they do not change the deploy↔teardown or cost model.

Conventions:
- UUIDs: `gen_random_uuid()`. Timestamps: `TIMESTAMPTZ DEFAULT now()`.
- Money/quantities: `NUMERIC` only — never floats.
- All DDL ships in idempotent sequential migrations; `make initdb` applies them.
- Cross-schema foreign keys (e.g. `oms → accounts.users`) are legal within the
  single cluster and are used where they make sense.

```
accounts.users ──◄ owner_user_id / initiated_by / acting_user_id
                     │
     portfolios.portfolios ◄─── portfolio_runs
                │
        (order intents, SQS)
                ▼
     oms.orders ── executions        oms.positions
                ▲
     securities.instruments ── daily_bars ── symbol_metrics
                └─ corpus_chunks ── macro_series / macro_observations
```

---

## 1. Securities database (`securities`)

Market data, reference data, RAG corpus, macro series, derived metrics.
Expected scale: `instruments` ≈ 9.2k rows (§13 universe); `daily_bars` ≈ 23M rows
(~2.3 GB, 10-year seed); `corpus_chunks` 10⁴–10⁵ rows.

```sql
CREATE TABLE securities.instruments (
  symbol           TEXT PRIMARY KEY,
  name             TEXT NOT NULL,
  sector           TEXT,
  exchange         TEXT,                      -- NASDAQ | NYSE (NasdaqTrader truth)
  etf              BOOLEAN NOT NULL DEFAULT false,
  active           BOOLEAN NOT NULL DEFAULT true,
  last_ingested_at TIMESTAMPTZ
);

CREATE TABLE securities.daily_bars (
  symbol     TEXT NOT NULL,                   -- references instruments(symbol) logically;
  trade_date DATE NOT NULL,                    -- no FK: bulk seed of ~23M rows, integrity
  open       NUMERIC(12,4),                    --   enforced by loader upsert order instead
  high       NUMERIC(12,4),
  low        NUMERIC(12,4),
  close      NUMERIC(12,4) NOT NULL,
  volume     BIGINT,
  source     TEXT NOT NULL,                    -- 'stooq' | 'massive'
  PRIMARY KEY (symbol, trade_date)
);
CREATE INDEX daily_bars_by_date ON securities.daily_bars (trade_date);

CREATE TABLE securities.symbol_metrics (        -- derived per-symbol metrics (plan §7)
  symbol              TEXT PRIMARY KEY,
  metric_date         DATE NOT NULL,
  ret_12m             NUMERIC(10,6),
  drift_3m            NUMERIC(10,6),
  vol_20d             NUMERIC(10,6),
  eps_surprise_latest NUMERIC(10,6),
  computed_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX symbol_metrics_by_date ON securities.symbol_metrics (metric_date);

CREATE TABLE securities.corpus_chunks (         -- filings + news + transcripts (RAG)
  id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  source       TEXT NOT NULL,                   -- 'edgar_10k'|'edgar_10q'|'finnhub_news'|'transcript'
  symbol       TEXT,
  published_at DATE,
  url          TEXT,
  section      TEXT,                            -- 'risk_factors' | 'mdna' | 'qa' | ...
  content      TEXT NOT NULL,
  embedding    vector(1024) NOT NULL            -- titan-embed-text-v2:0
);
CREATE INDEX corpus_hnsw ON securities.corpus_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX corpus_chunks_symbol ON securities.corpus_chunks (symbol);

CREATE TABLE securities.macro_series (
  series_id TEXT PRIMARY KEY,                  -- FRED id, e.g. 'DGS10'
  name      TEXT NOT NULL,
  frequency TEXT,
  source    TEXT NOT NULL DEFAULT 'fred'
);

CREATE TABLE securities.macro_observations (
  series_id TEXT NOT NULL REFERENCES securities.macro_series(series_id),
  obs_date  DATE NOT NULL,
  value     NUMERIC(18,6) NOT NULL,
  PRIMARY KEY (series_id, obs_date)
);
```

Notes / deltas vs plan §5:
- `instruments` + `exchange`/`etf`/`last_ingested_at` (§13), `symbol_metrics` and
  the two `macro_*` tables — §5 had no tables for the macro loader (§11) or the
  metrics table (§7); drafted here.
- `daily_bars` keeps `(symbol, trade_date)` PK with **no FK to instruments**
  (bulk upserts at seed; the loaders create instruments first). Cross-sectional
  scans (metrics, equity curves) get the `trade_date` index.

## 2. Accounts database (`accounts`)

Auth identities only. Everything ephemeral (jti denylist, login lockout,
run-status) lives in ElastiCache Redis per §12 — none of it is durable state.

```sql
CREATE TABLE accounts.users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,                 -- argon2id
  display_name  TEXT,
  role          TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user','admin')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Notes:
- RS256 private key: Secrets Manager; public key to OMS via SSM (plan §12) —
  not stored here.
- Seed migration idempotently creates the demo `admin` user with a generated
  password printed once to the deployer (`make` target) — no hardcoded creds in
  git.

## 3. OMS database (`oms`)

Order truth written only by the Java OMS (SQS poller). `client_order_id` is the
idempotency key: conflict = skip. Fill price = that symbol's last
`securities.daily_bars.close` for the trade date; no bar → REJECTED.

```sql
CREATE TABLE oms.orders (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_order_id TEXT NOT NULL UNIQUE,        -- intent UUID — idempotency
  portfolio_id    UUID NOT NULL,               -- no FK (matches §5): OMS-only write path
  symbol          TEXT NOT NULL,
  side            TEXT NOT NULL CHECK (side IN ('BUY','SELL')),
  qty             NUMERIC(14,4) NOT NULL,
  reason          TEXT NOT NULL,               -- INITIAL_BUILD | REBALANCE | SUBSTITUTION
  status          TEXT NOT NULL DEFAULT 'NEW' CHECK (status IN ('NEW','FILLED','REJECTED')),
  reject_reason   TEXT,
  acting_user_id  UUID REFERENCES accounts.users(id),   -- from signed actor claim (§12)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX orders_by_portfolio ON oms.orders (portfolio_id, created_at);

CREATE TABLE oms.executions (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id    UUID NOT NULL REFERENCES oms.orders(id),
  symbol      TEXT,
  qty         NUMERIC(14,4),
  price       NUMERIC(12,4),
  executed_at TIMESTAMPTZ                      -- mock fill at daily close (disclosed)
);

CREATE TABLE oms.positions (
  portfolio_id UUID NOT NULL,
  symbol       TEXT NOT NULL,
  qty          NUMERIC(14,4) NOT NULL DEFAULT 0,
  avg_cost     NUMERIC(12,4) NOT NULL DEFAULT 0,   -- WAC math on position upsert
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (portfolio_id, symbol)
);
```

Notes / deltas vs plan §5: adds `reject_reason` (§6 says REJECTED carries a
reason), `orders.updated_at`, `positions.updated_at`, CHECK constraints, and the
portfolio index. Transaction-history UI = `executions JOIN orders`; P&L
invariant (§10): `realized + unrealized + cash = initial cash ± rounding`.

## 4. Portfolios database (`portfolios`)

Portfolio state and the record of every LLM run (the "why" audit trail).

```sql
CREATE TABLE portfolios.portfolios (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  theme         TEXT NOT NULL,
  cash          NUMERIC(14,2) NOT NULL DEFAULT 100000,
  status        TEXT NOT NULL DEFAULT 'PENDING_BUILD' CHECK (status IN ('PENDING_BUILD','ACTIVE')),
  owner_user_id UUID NOT NULL REFERENCES accounts.users(id),   -- §12
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE portfolios.portfolio_runs (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id UUID NOT NULL REFERENCES portfolios.portfolios(id),
  trigger_type TEXT NOT NULL CHECK (trigger_type IN ('initial','scheduled','forced')),
  rationale    JSONB NOT NULL,                 -- reasoning, narratives, substitutions
  proposed     JSONB NOT NULL,                 -- target weights before delta orders
  status       TEXT NOT NULL CHECK (status IN ('PROPOSED','EXECUTING','DONE','FAILED')),
  initiated_by UUID REFERENCES accounts.users(id),   -- NULL = scheduler (§12)
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Notes / deltas vs plan §5: owner column is a real FK here; `trigger` renamed
`trigger_type` (§5's name reads as a SQL trigger keyword and is ambiguous);
`created_at` added to runs. Builder service emits **delta** orders from
`proposed` − current `oms.positions` (net-effect only, §6).

## 5. v2 draft additions — RAG metrics layer (2026-10-05 evening)

**Companion doc: [`docs/data-plan.md`](data-plan.md) (all sources verified live
that day).** Statement facts come from SEC XBRL (`frames`/`companyfacts` — no
key, license-clean), analyst consensus only exists as **snapshots** on free
tiers, and every derived number carries the source it was computed from. Per
`data-plan.md` §2: no free consensus/upgrade *history* exists anywhere — these
tables hold snapshots with explicit `as_of`, and the join to daily bars is how
the UI reconstructs a timeline honestly.

```sql
CREATE TABLE securities.fundamentals (          -- statement facts, point-in-time
  symbol    TEXT NOT NULL,                      -- instruments(symbol), logical
  concept   TEXT NOT NULL,                      -- us-gaap tag: 'Revenues', 'NetIncomeLoss', 'EarningsPerShareDiluted', ...
  period_start DATE NOT NULL,
  period_end   DATE NOT NULL,
  val       NUMERIC(20,4) NOT NULL,             -- as-reported USD
  filed_at  DATE NOT NULL,                      -- XBRL 'filed'; amended filings ADD rows, never rewrite
  accession TEXT NOT NULL,                      -- EDGAR accession — the citation string
  source    TEXT NOT NULL DEFAULT 'sec_xbrl',
  PRIMARY KEY (symbol, concept, period_end, filed_at)
);
CREATE INDEX fundamentals_symbol_period ON securities.fundamentals (symbol, concept, period_end DESC);

CREATE TABLE securities.analyst_ratings_daily ( -- consensus snapshot, not history
  symbol        TEXT NOT NULL,
  as_of         DATE NOT NULL,                  -- snapshot date
  strong_buy    INTEGER, buy INTEGER, hold INTEGER, sell INTEGER, strong_sell INTEGER,
  target_mean   NUMERIC(12,4), target_high NUMERIC(12,4), target_low NUMERIC(12,4),
  eps_surprise_latest NUMERIC(10,6),            -- joins Finnhub free 4-quarter surprises
  source        TEXT NOT NULL,                  -- 'yahoo_snapshot' | 'finnhub'
  fetched_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (symbol, as_of, source)
);

CREATE TABLE securities.source_registry (       -- what every feed is allowed to be
  source       TEXT PRIMARY KEY,                -- 'stooq' | 'massive' | 'finnhub' | 'sec_xbrl' | 'yahoo' | 'fred' | ...
  role         TEXT NOT NULL,                   -- 'seed' | 'nightly_bars' | 'statements' | 'corpus' | 'macro' | ...
  license_class TEXT NOT NULL,                  -- 'official' | 'personal_use' | 'tos_gray'
  tier         TEXT,                            -- free-tier / plan name at verification time
  verify_url   TEXT NOT NULL,                   -- vendor pricing/API page used for verification
  last_verified DATE NOT NULL,                  -- re-check quarterly (data-plan §5)
  notes        TEXT
);

ALTER TABLE securities.symbol_metrics
  ADD COLUMN net_margin      NUMERIC(10,6),     -- from fundamentals
  ADD COLUMN revenue_cagr_3y NUMERIC(10,6),
  ADD COLUMN drawdown_12m    NUMERIC(10,6);
```

Notes:
- `fundamentals` PK includes `filed_at`: restatements/amendments are appended as
  new point-in-time rows; queries default to "latest filed ≤ as-of date".
- `corpus_chunks.source` enum expands beyond §1's comment:
  `edgar_10k|edgar_10q|finnhub_news|transcript|fool_transcript` — section labels
  gain `prepared_remarks` alongside `qa`.
- `source_registry` feeds the in-app data-sources footer and the quarterly
  re-verification routine (`data-plan.md` §5) — disclosure as a table, not a
  one-off sentence.
- Ratings-history vendors (Finnhub premium, Massive Financials $29/mo) plug in
  behind the same `MarketDataProvider` adapter; schema needs no rework if a paid
  tier is ever added.

---

## Cross-schema reference map

| Reference | FK in draft | Why |
|---|---|---|
| `oms.executions.order_id → oms.orders.id` | yes | §5 |
| `oms.orders.acting_user_id → accounts.users.id` | yes | signed actor claim (§12) |
| `oms.orders/positions.portfolio_id → portfolios.portfolios.id` | no | §5 parity; OMS-only write path |
| `portfolios.portfolios.owner_user_id → accounts.users.id` | yes | ownership filter (§12) |
| `portfolios.portfolio_runs.initiated_by → accounts.users.id` | yes, nullable | NULL = scheduler |
| `securities.macro_observations → macro_series` | yes | §5 |

Ownership/authorization (owner-or-admin, foreign portfolios → 404) is enforced
in the API layer, not by row-level security, matching plan §12.