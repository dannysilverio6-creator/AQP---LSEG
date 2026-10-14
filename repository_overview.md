# Repository Overview — `quantpipe`

**Python implementation of the Multi-Market Quantitative Investment Pipeline**

| | |
|---|---|
| Methodology | `docs/quant_pipeline.md` (version 2, static universe) |
| Language | Python 3.12 only |
| Storage | Parquet files + DuckDB catalogue on local disk |
| Sleeves | ASX interday · US interday · ASX intraday · US intraday |
| Output | Position and leverage report + order tickets for manual placement |

> **How this maps to the methodology.** Each package under `src/quantpipe/` corresponds to one stage of `quant_pipeline.md`, and keeps its stage number (`s0_` … `s7_`). Section references in brackets (e.g. [§4.6]) point to the methodology document.

---

## 1. Repository Tree

```text
quantpipe/
├── pyproject.toml                 # dependencies, pinned with uv.lock
├── uv.lock
├── README.md
├── docs/
│   ├── quant_pipeline.md          # methodology (source of truth for the maths)
│   └── repository_overview.md     # this file
├── configs/                       # every threshold lives here, none in code
│   ├── base.yaml                  # paths, calendars, quantile grid, seeds
│   ├── sleeves/
│   │   ├── asx_interday.yaml
│   │   ├── us_interday.yaml
│   │   ├── asx_intraday.yaml
│   │   └── us_intraday.yaml
│   ├── universe.yaml              # formation rules + safety exits [§4]
│   ├── models.yaml                # model families and hyperparameter grids
│   ├── validation.yaml            # folds, purge/embargo, G3 criteria [§10]
│   ├── risk.yaml                  # covariance, ES engines, stress library, G4 [§11]
│   ├── leverage.yaml              # Kelly fraction, vol target, DD_max, G5 [§12]
│   ├── execution.yaml             # cost model, participation caps, hard limits [§13]
│   └── data_sources.yaml          # provider credentials *references* (never secrets)
├── data/                          # NOT committed (in .gitignore)
│   ├── raw/                       # append-only provider responses
│   ├── pit/                       # bitemporal facts
│   ├── market/                    # daily and intraday bars, ticks
│   ├── knowledge/                 # stickers, labels, peers, graph (versioned)
│   ├── universe/                  # frozen universe versions U_{T0}, U_{T1}, …
│   ├── derived/                   # features, forecasts, signals
│   ├── runs/{run_id}/             # outputs, manifest, gate results per run
│   └── catalog.duckdb
├── src/quantpipe/
│   ├── core/                      # shared infrastructure
│   ├── s0_data/                   # Stage 0: point-in-time data
│   ├── s0_knowledge/              # Stage 0: company knowledge layer
│   ├── s1_universe/               # Stage 1: archetype builder (static)
│   ├── s2_models/                 # Stage 2: four forecast layers
│   ├── s2e_combine/               # Stage 2E: validation, eligibility, combination
│   ├── s3_risk/                   # Stage 3: risk model
│   ├── s4_portfolio/              # Stage 4: optimiser + sleeve allocator
│   ├── s5_leverage/               # Stage 5: leverage, decisions, report
│   ├── s6_execution/              # Stage 6: cost model, tickets, hard limits
│   ├── s7_monitoring/             # Stage 7: drift, recalibration, attribution
│   ├── backtest/                  # event-driven simulator
│   └── cli.py                     # command-line entry points
├── tests/
│   ├── unit/
│   ├── leakage/                   # look-ahead and survivorship tests
│   ├── integration/               # end-to-end on synthetic data
│   └── fixtures/                  # synthetic data generators only
└── notebooks/                     # research only; never imported by src/
```

---

## 2. Package-by-Package Description

### 2.1 `core/` — shared infrastructure

| Module | Responsibility |
|---|---|
| `config.py` | Loads and validates YAML into typed objects (pydantic); merges `base.yaml` with a sleeve file |
| `types.py` | Shared data structures: `ForecastDist`, `UniverseVersion`, `GateResult`, `Position`, `Ticket` (Section 3) |
| `pit.py` | The **only** read path for model code: `pit.read(dataset, as_of, obs=None)` [§3.1] |
| `calendar.py` | ASX and US trading calendars, session times, holidays; Sydney ↔ UTC conversion |
| `store.py` | Atomic Parquet writes (temp file + rename), DuckDB views, checksums |
| `gates.py` | `Gate` base class and the G1–G7 implementations' common reporting format [§2.2] |
| `manifest.py` | Run ID, config hash, input hashes, gate outcomes written to `runs/{run_id}/manifest.json` |
| `registry.py` | Model registry and **experiment trial counter** used by the deflated Sharpe ratio [§10.2] |
| `rng.py` | Seeded random streams so every run is reproducible |

### 2.2 `s0_data/` — point-in-time data [§3]

| Module | Responsibility |
|---|---|
| `sources/lseg.py` | ASX market data, fundamentals, estimates, news via the LSEG Data Library; rate limiting and request ledger |
| `sources/us_market.py` | US market and fundamentals provider adapter (provider still to be chosen) |
| `sources/macro.py` | Macro series **with vintages** per economy (RBA, ABS, FRED/ALFRED, China, etc.) |
| `sources/news.py` | Wire and exchange announcements with publication timestamps |
| `sources/base.py` | `DataSource` protocol all adapters implement, so providers can be swapped |
| `ingest.py` | Incremental loads with watermarks; append-only to `raw/` |
| `pit_store.py` | Bitemporal merge: closes `known_to` on old versions, opens new ones; never updates |
| `knowledge_time.py` | Knowledge-time rules per data type (announcement time, release time, first-seen) |
| `corporate_actions.py` | Adjustment factors, total-return series, tradability flags [§3.3] |
| `quality.py` | **◆ G2** data-integrity firewall: BLOCK / QUARANTINE / WARN checks [§3.4] |

### 2.3 `s0_knowledge/` — company knowledge layer [§3.5]

| Module | Responsibility |
|---|---|
| `stickers.py` | Builds versioned revenue and cost exposure vectors $\mathbf e^{\mathrm{rev}}_i$, $\mathbf e^{\mathrm{cost}}_i$ from segment disclosures |
| `labels.py` | Market, industry, venue and currency labels |
| `peers.py` | Data-driven peer clusters from return correlations and business-description embeddings |
| `graph.py` | Relationship graph $W$ (supplier, customer, competitor); spectral-radius check |
| `extract_relations.py` | LLM-assisted extraction of relationships from filings, written to the graph with source IDs |

### 2.4 `s1_universe/` — archetype builder, static universe [§4]

| Module | Responsibility |
|---|---|
| `survival.py` | Merton DD solver, Altman Z, interest cover, leverage, runway, F-score [§4.2] |
| `liquidity.py` | MDTV, Amihud, Roll spread, zero-volume ratio, quoted spread [§4.3] |
| `archetypes.py` | Fit tests per archetype, HAR forecastability test, beta CUSUM stability [§4.4] |
| `scoring.py` | Robust sector-neutral z-scores and composite score $q_i$ [§4.5] |
| `formation.py` | **◆ G1 at formation.** Runs the full screen once at $T_0$, writes a frozen `UniverseVersion` with reason codes |
| `safety_exits.py` | **◆ G1 after formation.** Checks only `EXIT-DIST`, `EXIT-DLST`, `EXIT-SUSP`, `EXIT-TKO`, `EXIT-LIQ` [§4.6] |
| `tiers.py` | One-off data-depth tier assignment at formation [§4.7] |

**Static-universe rules enforced in code:**

- `UniverseVersion` is immutable. `formation.py` is the only code that creates one.
- `safety_exits.py` can only *remove* members; there is no function that adds a member to an existing version.
- No module in `s2_` to `s7_` may import from `s1_universe.scoring` or `formation` — enforced by an import-linter rule in the tests.
- A re-formation creates a new version file (`universe/asx_interday/U_2026-10-14.parquet`) and is triggered only by the `form-universe` command.

### 2.5 `s2_models/` — forecast layers [§5–9]

```text
s2_models/
├── base.py              # ForecastModel protocol, quantile grid, pinball/CRPS
├── targets.py           # excess-return targets with label_end for purging [§5.2]
├── features/
│   ├── market.py        # momentum, reversal, volatility, illiquidity
│   ├── fundamental.py   # value, profitability, accruals, revisions, SUE
│   └── transform.py     # in-fold robust z / rank-Gauss (never full-sample)
├── financial/           # §6
│   ├── baselines.py     # random walk, historical mean, factor model, EWMA vol
│   ├── returns.py       # elastic net, LightGBM quantile, quantile forest
│   ├── volatility.py    # GJR-GARCH-t, EGARCH (arch), HAR-RV(-J)
│   ├── distress.py      # discrete-time hazard model → p_dist
│   └── state_space.py   # Kalman time-varying betas (filtered only)
├── macro/               # §7
│   ├── dfm.py           # dynamic factor model per economy
│   ├── regimes.py       # Markov-switching, Hamilton filter → π_t(k)
│   ├── nowcast.py       # MIDAS, Minnesota-prior BVAR
│   ├── exposure.py      # sticker-weighted company macro factor
│   ├── sensitivities.py # empirical-Bayes shrinkage toward sticker prior
│   ├── events.py        # standardised release surprises
│   ├── transmission.py  # local projections → sector IRFs → firm shocks
│   └── fx.py            # net FX exposure from revenue/cost stickers
├── micro/               # §8
│   ├── panel.py         # two-way FE with clustered errors (linearmodels)
│   ├── hierarchical.py  # partial pooling by market label
│   ├── dml.py           # double machine learning (cross-fitted)
│   ├── earnings.py      # earnings-response coefficients by group
│   ├── realised.py      # trade cleaning, RV, BV, realised kernel, jumps
│   └── microstructure.py# Hawkes, ACD, order-flow imbalance, Kyle λ
└── news/                # §9
    ├── ingest.py        # dedupe, entity linking, source credibility
    ├── extract.py       # LLM structured extraction (fixed schema, pinned model)
    ├── schema.py        # pydantic schema for extracted events
    ├── novelty.py       # embedding novelty, priced-in check
    ├── propagate.py     # (I − δW)⁻¹ s⁰ propagation with time decay
    └── model.py         # calibrated shock probability + quantile model
```

Every model class implements the same interface (Section 3.2), so Stage 2E treats all four layers identically.

### 2.6 `s2e_combine/` — validation, eligibility, combination [§10]

| Module | Responsibility |
|---|---|
| `splits.py` | Purged, embargoed walk-forward folds with nested inner splits [§10.1] |
| `stat_tests.py` | Diebold–Mariano (HLN), Giacomini–White, Hansen SPA, MCS, Romano–Wolf |
| `overfitting.py` | Deflated Sharpe ratio (trial count from `core.registry`), PBO via CSCV |
| `eligibility.py` | **◆ G3**: convergence, stability, calibration, OOS skill, net-of-cost value, overfitting [§10.3] |
| `combine.py` | Inverse-CRPS weights, stacking, regime weights, shrinkage to equal weights [§10.4] |
| `conformal.py` | CQR and adaptive conformal calibration [§10.5] |
| `alpha_decay.py` | IC term structure, half-life, horizon choice per sleeve [§10.6] |

### 2.7 `s3_risk/` — risk model [§11.1–11.3]

| Module | Responsibility |
|---|---|
| `covariance.py` | Ledoit–Wolf, factor model, DCC-GARCH, regime mixture; cross-market lead–lag treatment |
| `tail.py` | Filtered historical simulation, GJR-t Monte Carlo with t-copula, EVT; Euler ES contributions |
| `risk_backtest.py` | Kupiec, Christoffersen, Acerbi–Székely; engine demotion |
| `stress.py` | Stress library including sticker-driven and graph-contagion scenarios |

### 2.8 `s4_portfolio/` — optimiser and allocator [§11.4–11.5]

| Module | Responsibility |
|---|---|
| `black_litterman.py` | Equilibrium prior + combined forecasts as views |
| `optimiser.py` | Cost-aware convex problem (cvxpy + Clarabel) with liquidity, sector, country, CVaR constraints; infeasibility protocol |
| `risk_limits.py` | **◆ G4** post-optimisation checks |
| `allocator.py` | Risk budgeting across sleeves; intraday cap; FX budget |

### 2.9 `s5_leverage/` — leverage, decisions, report [§12]

| Module | Responsibility |
|---|---|
| `kelly.py` | Uncertainty-adjusted fractional Kelly cap |
| `leverage.py` | $L_t=\min\{\cdot\}\times$ drawdown multiplier; intraday vs overnight; financing cost; **◆ G5** |
| `decision.py` | State machine, no-trade band, invalidation conditions |
| `governance.py` | **◆ G6** NO-TRADE rule sets |
| `report.py` | Per-company and per-sleeve position and leverage report (Markdown, HTML, XLSX, JSON) |

### 2.10 `s6_execution/` — execution modelling [§13.1–13.2]

| Module | Responsibility |
|---|---|
| `cost_model.py` | Spread + square-root impact + commission; recalibration of $Y$ from fills |
| `schedule.py` | Almgren–Chriss schedule; auction vs VWAP choice |
| `tickets.py` | Order tickets for **manual** placement with the broker |
| `fills.py` | Imports executed fills; implementation shortfall |
| `hard_limits.py` | Daily loss, drawdown halt, P&L divergence, per-name and gross caps — no imports from model packages |

### 2.11 `s7_monitoring/` — monitoring and learning [§13.3–13.5]

| Module | Responsibility |
|---|---|
| `drift.py` | PSI, KS, Page–Hinkley, calibration coverage, alpha-decay alarms |
| `recalibration.py` | Trigger → action policy table |
| `lifecycle.py` | Candidate → shadow → challenger → champion; **◆ G7** |
| `attribution.py` | Stage, layer, Brinson–Fachler, factor and execution attribution |

Monitoring never writes to `data/universe/`; it can only change model weights, model status and sleeve budgets.

### 2.12 `backtest/`

An event-driven simulator that calls the **same** `s4_`–`s6_` functions used for live tickets: decision after the close, fill at the next session, participation caps, lot sizes, delisting haircut. Each backtest forms its universe at its own start date from point-in-time data.

---

## 3. Key Interfaces

### 3.1 Forecast distribution (common language of Stage 2)

```python
@dataclass(frozen=True)
class ForecastDist:
    model_id: str            # "financial/lgbm_quantile@1.2.0"
    sleeve: str              # "asx_interday"
    as_of: pd.Timestamp      # decision time t
    horizon: int             # h, sessions
    sec_ids: np.ndarray      # (n,)
    quantiles: np.ndarray    # (n, len(TAU_GRID)), monotone along axis 1
    label_end: pd.Timestamp  # used for purging

TAU_GRID = (0.01, 0.025, 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95, 0.975, 0.99)
```

### 3.2 Forecast model protocol

```python
class ForecastModel(Protocol):
    id: str
    layer: Literal["financial", "macro", "micro", "news"]
    horizons: tuple[int, ...]

    def fit(self, train: FeaturePanel, targets: TargetPanel) -> FitReport: ...
    def predict(self, panel: FeaturePanel, as_of: pd.Timestamp,
                horizon: int) -> ForecastDist: ...
    def diagnostics(self) -> dict[str, float]: ...   # convergence, stability → G3
```

### 3.3 Frozen universe

```python
@dataclass(frozen=True)
class UniverseVersion:
    sleeve: str
    formation_date: pd.Timestamp
    members: frozenset[str]
    archetypes: Mapping[str, tuple[str, ...]]
    tiers: Mapping[str, Literal["T1", "T2", "T3"]]
    rejections: Mapping[str, str]          # sec_id -> reason code
    exits: Mapping[str, tuple[pd.Timestamp, str]] = field(default_factory=dict)

    def active(self, as_of: pd.Timestamp) -> frozenset[str]:
        """Members not exited on or before as_of. Never adds names."""
```

### 3.4 Point-in-time read

```python
df = pit.read("fundamentals", as_of=t, fields=["eps", "ebit"], sec_ids=universe.active(t))
```

Raw tables are not importable by model code; a leakage test inserts "future-poison" rows and fails if any reach a model input.

---

## 4. Configuration Examples

### 4.1 `configs/universe.yaml`

```yaml
universe:
  mode: static                 # filtration runs once at formation
  formation:
    admission_rank_min: 0.50
    min_size: {asx_interday: 60, us_interday: 80, asx_intraday: 20, us_intraday: 30}
    max_size: {asx_interday: 200, us_interday: 300, asx_intraday: 40, us_intraday: 60}
    history_min_days: 252
  survival:
    distress_prob_max: 0.05    # hazard model, 12-month horizon
    icr_min: 2.0
    nd_ebitda_max: 4.0
    runway_min_quarters: 4
    altman_z_min: 1.8
  liquidity:
    mdtv_min: {AUD: 5.0e5, USD: 2.0e6}
    zero_volume_max: 0.05
    intraday_spread_max_bps: 10
  safety_exits:
    distress_prob_max: 0.05
    suspension_max_sessions: 5
    liquidity_collapse: {mdtv_fraction_of_floor: 0.5, sessions: 60}
    replace_exited: false      # never
  reformation:
    trigger: manual            # deliberate decision only
    advisory_min_fraction: 0.70
```

### 4.2 `configs/sleeves/us_interday.yaml`

```yaml
sleeve: us_interday
market: US
currency: USD
calendar: XNYS
data_source: us_market
horizons: [5, 21]
execution: {default: closing_auction, vwap_if_auction_share_above: 0.05}
risk_budget_initial: 0.30
overnight: true
fx: {base: AUD, hedge: false, budget_vol: 0.03}
```

### 4.3 `configs/leverage.yaml`

```yaml
leverage:
  kelly_fraction: 0.25
  vol_target_annual: 0.10
  max_gross: 1.5
  drawdown_max: 0.20
  margin_buffer: 0.30          # headroom after worst stress
  intraday_max_gross: 2.0
  overnight_uses_gap_stress: true
```

---

## 5. Python Dependencies

| Area | Libraries |
|---|---|
| Data handling | `pandas`, `polars` (large panels), `pyarrow`, `duckdb` |
| Numerics | `numpy`, `scipy`, `numba` (Hawkes, ACD, bootstrap and backtest loops) |
| Econometrics | `statsmodels` (Markov switching, dynamic factor, Kalman, local projections), `arch` (GARCH family, bootstrap), `linearmodels` (panel FE) |
| Machine learning | `scikit-learn`, `lightgbm`, `quantile-forest`, `doubleml` |
| Optimisation | `cvxpy` with `clarabel` |
| Text and LLM | provider SDK for the chosen LLM, `sentence-transformers` (embeddings), `pydantic` (extraction schema) |
| Data providers | `lseg-data` (ASX); US provider package to be chosen |
| Config and CLI | `pydantic`, `pyyaml`, `typer` |
| Reporting | `jinja2`, `matplotlib`, `openpyxl` |
| Quality | `pytest`, `hypothesis`, `ruff`, `mypy`, `import-linter` |

**Performance note.** Python is sufficient for this design. The handful of sequential loops (Hawkes likelihood, SPA bootstrap, event-driven backtest, tick cleaning) are written with `numba`; GARCH uses `arch`; everything else is vectorised NumPy or library code that is already compiled.

---

## 6. Running the Pipeline

All runs are started manually from the command line.

```bash
# 1. Install
uv sync

# 2. Load or refresh data up to a date
quantpipe ingest --sleeve asx_interday --as-of 2026-10-14

# 3. Build the knowledge layer (stickers, labels, peers, graph)
quantpipe knowledge --as-of 2026-10-14

# 4. Form the static universe ONCE per sleeve
quantpipe form-universe --sleeve asx_interday --as-of 2026-10-14

# 5. Daily run: safety exits -> models -> combine -> risk -> optimise -> leverage -> report + tickets
quantpipe run --sleeve asx_interday --as-of 2026-10-14

# 6. All sleeves plus the cross-sleeve allocator
quantpipe run --all --as-of 2026-10-14

# 7. Backtest (forms its own universe at the start date)
quantpipe backtest --sleeve asx_interday --start 2016-01-04 --end 2026-09-30

# 8. Model lifecycle review
quantpipe review-models --sleeve asx_interday
```

### What `quantpipe run` does, in order

```text
load configs + UniverseVersion
 → s0_data: incremental ingest, PIT merge, ◆ G2
 → s1_universe.safety_exits: remove exited names (no additions)
 → s2_models: fit/predict each layer → ForecastDist
 → s2e_combine: ◆ G3 eligibility → combine → conformal → horizon
 → s3_risk: covariance, ES engines, stress
 → s4_portfolio: Black–Litterman → optimiser → ◆ G4 → sleeve allocator
 → s5_leverage: leverage rule → ◆ G5 → decisions → ◆ G6
 → s6_execution: tickets (manual placement), hard-limit check
 → s7_monitoring: drift and attribution on previous fills
 → write report + manifest to data/runs/{run_id}/
```

A failed gate stops only the branch it governs and is recorded in the manifest; it never silently passes downstream.

---

## 7. Testing Strategy

| Test type | What it checks |
|---|---|
| Unit | Each formula against a hand-worked value (e.g. the worked examples in the methodology) |
| Reference parity | GJR likelihood vs `arch`, SPA vs a NumPy reference, Kupiec vs closed form |
| Leakage | Future-poison rows never reach a model; shifting features by one day must change results; restated values invisible before their knowledge time |
| Survivorship | Backtest universes include later-delisted names |
| Static universe | No code path adds a member to an existing `UniverseVersion`; monitoring cannot write to `data/universe/` |
| Gate behaviour | Each gate's pass and fail branches on synthetic inputs |
| Integration | Full `run` on synthetic data for all four sleeves |
| Determinism | Same inputs + config + seed → identical output hashes |

Fixtures are generated synthetically; no licensed data is stored in the repository.

---

## 8. Suggested Build Order

| Phase | Scope | Done when |
|---|---|---|
| 1 | `core`, `s0_data` (ASX daily), PIT store, G2 | 20 consecutive days ingested with G2 green and identical hashes on re-run |
| 2 | `s0_knowledge` (stickers, labels), `s1_universe`, formation of ASX interday universe | Formation reproducible; safety exits tested on historical delistings |
| 3 | `s2_models` baselines + GJR/HAR + elastic net + LightGBM; `s2e_combine` | G3 decisions for at least three families; DSR uses registry trial counts |
| 4 | `s3_risk`, `s4_portfolio`, `s5_leverage`, `s6_execution` tickets; `backtest` | Paper trading for 40 sessions with full gate traces |
| 5 | Macro layer (DFM, regimes, sensitivities, transmission) and micro layer (panel, hierarchical) | Both layers pass G3 or are documented as non-eligible |
| 6 | US interday sleeve, FX channel, sleeve allocator | Cross-sleeve covariance and allocator validated |
| 7 | News layer (LLM extraction, propagation) | Evaluated only on post-cutoff news; forward paper results |
| 8 | Intraday sleeves (realised measures, microstructure) | Extended paper trading; simulated vs realised fills compared |
| 9 | `s7_monitoring` lifecycle and small live capital | Hard limits tested end to end |

---

## 9. What Is Deliberately Not in the Repository

- Licensed market data, credentials or API keys (kept in the OS keychain and `data/`, both outside version control).
- Scheduling or automation: every run is started manually.
- Automatic order placement: the pipeline produces tickets; orders are placed by hand.
