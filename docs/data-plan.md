# Data plan — verified against live sources (2026-10-05)

> **Status: source verification complete (2026-10-05 evening), supersedes plan §2 /
> §13 where they differ.** Every load-bearing claim below was checked against the
> vendor's own page or a live API call at retrieval time. Items marked
> *(third-party)* were not verifiable at the primary source.

This is the data layer of the RAG-first roadmap: **pipelines before apps**. The
LLM cannot pick or defend equities without (a) bars, (b) statements, (c)
ratings/estimates context, (d) a text corpus, (e) macro regime context. Sources
are picked so the `MarketDataProvider` / corpus adapters keep a paid-swap path
without pipeline rewrites.

---

## 1. Findings vs the plan as written (2026-10-05)

| # | Finding (verified live) | Impact | Decision |
|---|---|---|---|
| F1 | **Stooq denies scripted downloads.** `/q/d/l/?s=aapl.us&i=d` → "Access denied"; the 515 MB bundle (`db/d/?b=d_us_txt`) → "Unauthorized", even after solving the site's JS SHA-256 proof-of-work. The flow for bulk files = PoW **+ 4-char CAPTCHA + download ticket** (`cpt_g` → `cpt_a` → POST answer to `/q/l/s/`). Site browsing works; automated pulls do not. | Seed + tail-refresh backbone (plan §2/§13) is not scriptable anymore | `make seed` consumes a **bundle you download once in your browser** (browser solves its own CAPTCHA); **yfinance (verified live: 2,512 rows of 10y daily bars for AAPL)** is the automated, labeled ToS-gray fallback. No scraping code against Stooq. |
| F2 | **Finnhub free includes no ratings layer.** Finnhub's own pricing page: Free = company profile, 1-yr company news, 4 EPS-surprise quarters, 1-mo earnings calendar. Recommendation Trends / Price Target / EPS & revenue estimates / Upgrade-Downgrade / transcripts are All-In-One ($3,500/mo). | "Free ratings" assumption dead for official APIs | Statement facts from **SEC XBRL** (fully free); street-consensus via **Yahoo snapshot fields** (labeled gray), history table built only if a paid vendor is added (designed, no source). Upgrade/downgrade history = structural gap, disclosed. |
| F3 | **SEC XBRL is the only license-clean primary for statements** — and it is excellent: `frames` = one call returns a concept across every filer (2,206 filers on one quarterly revenue frame; 6,382 on an annual net-income frame); `companyfacts` = one call = a company's full statement history (Acme 2.5 MB, MSFT 4.9 MB). `company_tickers.json` maps all **10,434** tickers → CIK. No key; ~10 req/s fair-use ceiling. | The statements pillar costs ~$0 and is *official* | Frames bulk-backfill the whole universe's fundamentals; `companyfacts` refreshes portfolio members nightly; filing text via the **verified chain**: `submissions` → accession → `Archives/edgar/data/.../<doc>` (2.4 MB 10-K fetched, HTTP 200). |
| F4 | **EDGAR has per-CIK quirks.** Apple's CIK (320193) 404s on `companyfacts`/`companyconcept`/`submissions` while MSFT and every other tested CIK return 200; frames still cover Apple's facts. | One flaky issuer must not sink a run | Fetcher rule: on 404 → log, fall back to frames-derived facts, move on. Never hard-fail a job on one ticker. |
| F5 | **Alpha Vantage free is a top-up, not a pillar** — real key limits ≈ 25 calls/day *(third-party-attested)*; demo key serves NEWS_SENTIMENT but gates statements behind key registration. | Ratings/news-vendor candidate only for tiny spot refreshes | Keep as NEWS_SENTIMENT top-up on the active subset only; optional. |
| F6 | **Massive free re-verified at massive.com/pricing**: $0/mo, all US tickers, 5 calls/min, 2 yrs history, EOD, reference data, corporate actions, technical indicators. Financials & Ratios = $29/mo (paid). Legacy `api.polygon.io` still answers in parallel. | Nightly incremental stays as planned | Unchanged (plan §11): active subset nightly; pin new code to `api.massive.com`. |
| F7 | **Motley Fool transcript archive is alive and rich** — full 2026 call transcripts with prepared remarks + Q&A render as plain text pages. Stretch goal stays feasible (ToS-gray, personal-use label). | Corpus depth | Unchanged: phase-2 stretch (existing plan). |
| F8 | **NasdaqTrader symbol directory live** (nasdaqlisted 5,638 lines, otherlisted 7,654 — pre-filter counts incl. ETFs/test issues). | Universe truth unchanged | Plan §13 stands. |
| F9 | **Yahoo chart API works from a residential IP** and rate-limits datacenter IPs (429 on this box). yfinance also exposes fundamentals + analyst fields (`trailingPE`, `returnOnEquity`, `recommendationMean`, `targetMeanPrice`, …). | $0 seed + tiny snapshot refreshes are real | Seed fallback (F1) + weekly analyst-snapshot refresh on the active subset, labeled ToS-gray. |

## 2. Source registry (post-verification)

| Source (role) | Free shape | Cadence | Limits / license | Verified how |
|---|---|---|---|---|
| **SEC EDGAR XBRL** — statements, primary | `frames` (whole-universe cross-sections), `companyfacts`, `companyconcept`, full-text search (FTS), real filing documents | backfill + nightly (members) | no key; ~10 req/s fair-use; public-domain-ish federal data; proper UA mandatory | 200s: frames ×3, companyfacts (Acme 2.5 MB, MSFT 4.9 MB), companyconcept, submissions, Archives 2.4 MB 10-K; 404s per F4 tolerated |
| **EDGAR FTS + documents** — filing text corpus | 20M+ filings since 2001; doc HTML fetchable | backfill + nightly | free, official | 200 (FTS hit, index page) |
| **Finnhub free tier** — news, EPS surprises, profile, earnings calendar | `/company-news` (1 yr), `/stock/metric?metric=all` basic financials incl. EPS surprises, `/stock/profile2`, earnings calendar | nightly | 60 calls/min; personal-use | endpoints confirmed on finnhub.io/docs + pricing page; ratings/estimates confirmed PAID (F2) |
| **Massive (ex-Polygon) Stocks Basic** — nightly EOD bars, active subset | EOD aggregates 2y, all US tickers | nightly | 5 calls/min; personal-use | pricing page re-verified (F6); API base answers (401-no-key) |
| **Yahoo Finance via yfinance** — seed fallback + analyst snapshot | 10y daily bars per ticker; `.info` ratings/fundamentals snapshot | one-time seed + weekly snapshot | unofficial, ToS-gray; breaks without notice; works on residential IPs | live: 2,512-row 10y AAPL history; ratings fields present |
| **Alpha Vantage** — news sentiment top-up | NEWS_SENTIMENT w/ per-ticker relevance scores, backfill ≈2022 | optional top-up | real key ≈25/day *(third-party)*; lifetime free key | demo-key 200 on NEWS_SENTIMENT; statements gated |
| **FRED** — macro regime | DGS10, CPI, payrolls, full history | Sat reconcile | free key | key-gated 400 on fake key (alive); trivial |
| **Stooq** — bulk bundle (manual) | 515 MB US zip (per-ticker multi-decade CSVs) | monthly, manual browser download; `make seed` ingests the zip | web-download terms; no API contract; CAPTCHA+ticket flow verified in page JS | F1 (PoW solved; ticket gate confirmed) |
| **NasdaqTrader** — universe truth | nasdaqlisted.txt + otherlisted.txt, pipe-delimited | daily file, pull Sat | free attribution-friendly | 200 live (F8) |
| **Motley Fool transcripts** — corpus depth (stretch) | full prepared remarks + Q&A | phase 2 | scrape-derived; personal-use label | live 2026 transcripts (F7) |

**Structural gaps (disclose in-app, never workaround silently):** no free
consensus-estimate *history* (4 quarters of EPS surprises only); no analyst
upgrade/downgrade *history*; no real-time tape/L2 (out of scope anyway); ratings
snapshots are spot-values, not history.

## 3. Revised pipeline architecture (jobs keep the §11 names)

The §11 decision stands untouched: **clock for data, events for orders**;
EventBridge Scheduler → ECS `RunTask`; lookback windows, not cursors; no
queue/streaming in the data path. What changes is job *contents*:

| Job | Cadence | Contents (revised) |
|---|---|---|
| `frames_backfill` (NEW) | one-time | ~8 concepts × ~26 periods ≈ **200-300 SEC frames calls** for a whole-universe statements base → `securities.fundamentals`. Minutes of runtime, zero keys. |
| `ingestion seed` | manual | Ingest one manually-downloaded Stooq bundle zip (`make seed seedfile=~/Downloads/d_us_txt.zip`); **fallback**: `make seed-yahoo` from the Mac — ~9k tickers × ~1.5 s ≈ 4 h, labeled ToS-gray. Bars tagged `source` ∈ {stooq, yahoo}. |
| `ingestion nightly` | 20:00 ET Mon–Fri | Unchanged: Massive bars (active subset, 13 s throttle), Finnhub news+profile+EPS surprises, EDGAR filings for members → corpus + embeddings. **Added:** `companyfacts`/`companyconcept` statement facts for members → `fundamentals` (statements move quarterly; the nightly pass only re-checks). |
| `ingestion metrics` (SHARPENED) | nightly, deeper Sat | Derived metrics from bars for the active subset (plan §7 metrics) **+ fundamentals-derived ratio columns (net margin, revenue CAGR) + weekly analystSnapshot refresh (Yahoo .info gray-labeled)** → `symbol_metrics` / `analyst_ratings_daily`. |
| `ingestion reconcile` | Sat 03:00 ET | Tail refresh: yfinance tail sweep for non-member names that have moved (cheap), or consume a fresh manual bundle; FRED; universe diff vs NasdaqTrader. Stooq re-pull (plan §11) demoted to manual-bundle consumption per F1. |

Failure policy: single-ticker EDGAR 404 (F4) or Yahoo 429 → log + skip + retry
next window; jobs never hard-fail on one issuer or one rate-limited source.

## 4. How the RAG uses this (the point of the exercise)

Structured arm (exact rows, deterministic → strong grounding for numbers):
- `fundamentals` — revenue/profits/EPS trajectory as XBRL facts, point-in-time via `filed_at` (amended filings never rewrite history).
- `symbol_metrics` — technical state from bars (returns, drift, vol, drawdown) + fundamentals-derived ratios; the LLM sees computed numbers, never does arithmetic.
- `analyst_ratings_daily` — street-consensus snapshot with an explicit `as_of` + source label; UI shows "snapshot, not history".

Unstructured arm (the corpus that earns the "RAG" name):
- `corpus_chunks` — EDGAR 10-K/10-Q sections, Finnhub news, Fool transcripts (stretch): embeddings joined with citations (source/symbol/date) in the UI.

Retrieval pattern for portfolio build / rebalance / insights (plan §7):
theme → candidate set from metrics+fundamentals screen → corpus retrieval per
candidate → cited narrative → proposed portfolio. Macro gating (FRED) on sector
tilts rounds it out. Every insight row traces to: bars source, XBRL accession,
news URL, or transcript page — that traceability *is* the governance story, and
it is why all primary feeds below are license-clean: federal filings (official),
NasdaqTrader (official), FRED (official), Massive/Finnhub (personal-use, disclosed).

## 5. Licensing posture

- Public-domain-grade: SEC EDGAR (X+BRL + text), NasdaqTrader, FRED — the audit-heavy core. **This demo can claim source-truth licensing only because statements came from SEC, not from a vendor's restated feed.**
- Personal-use-license tiers (disclosed in footer): Massive, Finnhub, Alpha Vantage.
- ToS-gray (labeled, fallback/small-snapshot only): Yahoo via yfinance; Motley Fool archives. Never the stated backbone of a feed; swap-ready.
- `source_registry` table (schema doc §1.1) records per-source license class,
  tier, last-verified date + URL — the quarterly re-verify hook that makes the
  disclosure durable instead of decorative.