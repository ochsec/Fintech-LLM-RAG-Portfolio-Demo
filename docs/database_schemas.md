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