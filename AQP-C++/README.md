# Repository Overview — `quantpipe-cpp`

**C++20 implementation of the Multi-Market Quantitative Investment Pipeline**

| | |
|---|---|
| Methodology | `docs/quant_pipeline.md` (version 2, static universe) |
| Language | C++20 (CMake, vcpkg) |
| Storage | Parquet files (Apache Arrow C++) + DuckDB catalogue on local disk |
| Sleeves | ASX interday · US interday · ASX intraday · US intraday |
| Output | Position and leverage report + order tickets for manual placement |

> **How this maps to the methodology.** Each library under `src/` corresponds to one stage of `quant_pipeline.md` and keeps its stage number (`s0_` … `s7_`). Section references in brackets (e.g. [§4.6]) point to the methodology document. The structure deliberately mirrors the Python overview, so the two can be compared module by module.

> **What changes compared with Python.** C++ has no direct equivalents of `statsmodels`, `arch`, `linearmodels` or `doubleml`. Markov switching, dynamic factor models, GARCH estimation, local projections, panel regressions and the statistical tests are therefore **implemented in this repository** on top of Eigen. That is the main cost of a C++ build: more code to write and verify, in exchange for speed, determinism and a single compiled binary. Section 9 sets out how to verify these implementations.

---

## 1. Repository Tree

```text
quantpipe-cpp/
├── CMakeLists.txt                 # top-level build
├── CMakePresets.json              # debug / release / sanitiser presets
├── vcpkg.json                     # pinned third-party dependencies
├── README.md
├── docs/
│   ├── quant_pipeline.md          # methodology (source of truth for the maths)
│   └── repository_overview_cpp.md # this file
├── configs/                       # every threshold lives here, none in code
│   ├── base.yaml
│   ├── sleeves/{asx,us}_{interday,intraday}.yaml
│   ├── universe.yaml              # formation rules + safety exits [§4]
│   ├── models.yaml
│   ├── validation.yaml            # folds, purge/embargo, G3 criteria [§10]
│   ├── risk.yaml                  # covariance, ES engines, stress, G4 [§11]
│   ├── leverage.yaml              # Kelly fraction, vol target, DD_max, G5 [§12]
│   ├── execution.yaml             # cost model, participation caps, hard limits [§13]
│   └── data_sources.yaml          # endpoints and credential *references* (never secrets)
├── data/                          # NOT committed (in .gitignore)
│   ├── raw/  pit/  market/  knowledge/
│   ├── universe/                  # frozen universe versions U_{T0}, U_{T1}, …
│   ├── derived/  runs/{run_id}/
│   └── catalog.duckdb
├── include/quantpipe/             # public headers, one folder per stage
│   ├── core/  s0_data/  s0_knowledge/  s1_universe/
│   ├── s2_models/  s2e_combine/  s3_risk/  s4_portfolio/
│   ├── s5_leverage/  s6_execution/  s7_monitoring/  backtest/
│   └── math/                      # shared numerical building blocks
├── src/                           # implementations, same layout as include/
│   └── … one static library per stage (qp_core, qp_s0_data, …)
├── apps/
│   └── quantpipe_cli.cpp          # command-line entry point
├── tests/
│   ├── unit/                      # GoogleTest
│   ├── reference/                 # golden values for parity checks (Section 9)
│   ├── leakage/                   # look-ahead and survivorship tests
│   ├── integration/               # end-to-end on synthetic data
│   └── fixtures/                  # synthetic data generators only
└── bench/                         # Google Benchmark micro-benchmarks
```

Each stage is built as its own static library. Dependencies only point **downstream** (e.g. `qp_s2_models` may link `qp_core` and `qp_math` but never `qp_s1_universe`), and CMake enforces this.

---

## 2. Library-by-Library Description

### 2.1 `math/` — numerical building blocks (shared by every stage)

Because there is no C++ `statsmodels`, the reusable numerics live here and are tested independently.

| Header | Contents |
|---|---|
| `linalg.hpp` | Eigen wrappers: robust Cholesky, pseudo-inverse, nearest positive-definite matrix |
| `optimize.hpp` | L-BFGS-B (bounded MLE) via LBFGSpp; Nelder–Mead fallback; numerical-gradient checker |
| `distributions.hpp` | Normal, standardised Student-$t$, GPD, gamma; log-densities, CDFs, quantiles (Boost.Math) |
| `regression.hpp` | OLS, WLS, ridge, quantile regression (iteratively reweighted / LP); HAC (Newey–West) and cluster-robust standard errors |
| `kalman.hpp` | Generic linear-Gaussian Kalman filter with missing observations (ragged edge); **filter only, no smoother in the production path** |
| `bootstrap.hpp` | Stationary bootstrap indices; one RNG stream per replicate |
| `robust.hpp` | Median, MAD, winsorising, rank-Gauss transform |
| `rng.hpp` | xoshiro256** streams seeded from the run manifest |
| `parallel.hpp` | `parallel_for` over independent work items (TBB or OpenMP); results independent of thread count |

### 2.2 `core/` — shared infrastructure

| Module | Responsibility |
|---|---|
| `config` | Loads and validates YAML (yaml-cpp) into typed structs; merges base and sleeve files |
| `types` | `ForecastDist`, `UniverseVersion`, `GateResult`, `Position`, `Ticket` (Section 3) |
| `pit` | The **only** read path for model code: `pit::read(dataset, as_of)` [§3.1] |
| `calendar` | ASX and US trading calendars, sessions, holidays; Sydney ↔ UTC (`std::chrono` time zones) |
| `store` | Arrow/Parquet I/O, DuckDB views, atomic writes (temp file + rename), xxHash checksums |
| `gates` | Common gate interface and reporting for G1–G7 [§2.2] |
| `manifest` | Run ID, config hash, input hashes, gate outcomes → `runs/{run_id}/manifest.json` |
| `registry` | Model registry and **experiment trial counter** for the deflated Sharpe ratio [§10.2] |
| `log` | Structured logging (spdlog, JSON lines) |

### 2.3 `s0_data/` — point-in-time data [§3]

| Module | Responsibility |
|---|---|
| `sources/data_source.hpp` | `DataSource` interface; every provider implements it |
| `sources/lseg_rest` | LSEG Data Platform REST adapter (HTTP via libcurl, JSON via nlohmann/json); rate limiter and request ledger |
| `sources/parquet_drop` | Reads Parquet files produced by an external ingest tool (see note below) |
| `sources/us_market` | US provider adapter (provider still to be chosen) |
| `sources/macro` | Macro series with vintages (e.g. ALFRED, RBA, ABS) over REST |
| `sources/news` | Wire and exchange announcements with publication timestamps |
| `ingest` | Incremental loads with watermarks; append-only `raw/` |
| `pit_store` | Bitemporal merge: closes old versions, opens new ones, never updates |
| `knowledge_time` | Knowledge-time rules per data type |
| `corporate_actions` | Adjustment factors, total-return series, tradability flags [§3.3] |
| `quality` | **◆ G2** firewall: BLOCK / QUARANTINE / WARN [§3.4] |

> **Data-access note.** LSEG's Workspace **Data Library** is published for Python, .NET and TypeScript rather than C++ (check LSEG's current developer documentation). A C++ build therefore has two options: call the LSEG Data Platform **REST** endpoints directly with a platform session (if your licence includes one), or let a small external tool write Parquet files into `data/raw/` that the `parquet_drop` adapter reads. Both feed the same PIT store, so nothing downstream changes.

### 2.4 `s0_knowledge/` — company knowledge layer [§3.5]

| Module | Responsibility |
|---|---|
| `stickers` | Versioned revenue and cost exposure vectors $\mathbf e^{\mathrm{rev}}_i$, $\mathbf e^{\mathrm{cost}}_i$ |
| `labels` | Market, industry, venue, currency labels |
| `peers` | Peer clusters from return correlations (hierarchical clustering) and text embeddings |
| `graph` | Relationship graph $W$ as an Eigen sparse matrix; spectral-radius check (power iteration) |
| `extract_relations` | LLM-assisted relationship extraction over HTTP, validated against a JSON schema |

### 2.5 `s1_universe/` — archetype builder, static universe [§4]

| Module | Responsibility |
|---|---|
| `survival` | Merton DD (2-equation solve), Altman Z, cover, leverage, runway, F-score [§4.2] |
| `liquidity` | MDTV, Amihud, Roll spread, zero-volume ratio, quoted spread [§4.3] |
| `archetypes` | Archetype fit tests; HAR forecastability; recursive-residual CUSUM [§4.4] |
| `scoring` | Robust sector-neutral z-scores, composite score [§4.5] |
| `formation` | **◆ G1 at formation**: the only code that constructs a `UniverseVersion` |
| `safety_exits` | **◆ G1 after formation**: `EXIT-DIST`, `EXIT-DLST`, `EXIT-SUSP`, `EXIT-TKO`, `EXIT-LIQ` only [§4.6] |
| `tiers` | One-off data-depth tier assignment [§4.7] |

**Static-universe rules enforced by the type system and build:**

- `UniverseVersion` has a private constructor; `formation` is declared its only `friend`.
- `safety_exits` returns a new `UniverseVersion` with members removed; there is no member function that adds a name.
- `qp_s1_universe` exposes only `UniverseVersion` and `safety_exits` to downstream libraries; `scoring` and `formation` headers are private to the target, so stages 2–7 cannot include them.
- Re-formation writes a new versioned file and is triggered only by the `form-universe` command.

### 2.6 `s2_models/` — forecast layers [§5–9]

```text
s2_models/
├── forecast_model.hpp     # ForecastModel concept, quantile grid, pinball/CRPS
├── targets                # excess-return targets with label_end [§5.2]
├── features/              # market, fundamental, in-fold transforms
├── financial/             # §6
│   ├── baselines          # random walk, historical mean, factor model, EWMA
│   ├── elastic_net        # coordinate descent with warm starts
│   ├── gbm_quantile       # LightGBM C API, quantile objective, monotone constraints
│   ├── quantile_forest    # in-house quantile random forest
│   ├── garch              # GJR-GARCH-t and EGARCH: NLL + analytic gradient, L-BFGS-B
│   ├── har                # HAR-RV(-J) by OLS
│   ├── hazard             # discrete-time logistic hazard → p_dist
│   └── state_space        # Kalman time-varying betas (filtered only)
├── macro/                 # §7
│   ├── dfm                # dynamic factor model, EM + Kalman, ragged edge
│   ├── markov_switching   # Hamilton filter, EM estimation → π_t(k)
│   ├── midas              # exponential Almon weights, NLS
│   ├── bvar               # Minnesota-prior BVAR (closed-form posterior)
│   ├── exposure           # sticker-weighted company macro factor
│   ├── sensitivities      # empirical-Bayes shrinkage toward sticker prior
│   ├── events             # standardised release surprises
│   ├── local_projections  # per-horizon OLS with Newey–West bands
│   └── fx                 # net FX exposure from stickers
├── micro/                 # §8
│   ├── panel_fe           # two-way FE by alternating demeaning, clustered SEs
│   ├── hierarchical       # partial pooling by market label
│   ├── dml                # cross-fitted partially linear model
│   ├── earnings_response  # CAR on SUE by group
│   ├── realised           # trade cleaning, RV, BV, realised kernel, jump test
│   └── microstructure     # Hawkes (O(n) NLL + gradient), ACD, OFI, Kyle λ
└── news/                  # §9
    ├── ingest             # dedupe, entity linking, credibility
    ├── llm_client         # HTTP client to the chosen LLM API; pinned model + prompt
    ├── schema             # JSON schema for extracted events; strict validation
    ├── novelty            # embedding cosine novelty, priced-in check
    ├── propagate          # sparse solve (I − δW) s = s⁰, time decay
    └── shock_model        # calibrated logistic + quantile model
```

**LLM and embeddings in C++.** Extraction calls an LLM over HTTPS (libcurl) and validates responses against a JSON schema; the model name and prompt version are written to the run manifest. Embeddings come either from the same API or from a local ONNX model through ONNX Runtime's C++ API.

### 2.7 `s2e_combine/` — validation, eligibility, combination [§10]

| Module | Responsibility |
|---|---|
| `splits` | Purged, embargoed walk-forward folds with nested inner splits |
| `stat_tests` | Diebold–Mariano (HLN), Giacomini–White, Hansen SPA (parallel stationary bootstrap), MCS, Romano–Wolf |
| `overfitting` | Deflated Sharpe ratio (trial count from `core::registry`), PBO via CSCV |
| `eligibility` | **◆ G3** criteria [§10.3] |
| `combine` | Inverse-CRPS weights, stacking (small QP), regime weights, shrinkage to equal |
| `conformal` | CQR and adaptive conformal |
| `alpha_decay` | IC term structure, half-life, horizon choice |

### 2.8 `s3_risk/` — risk model [§11.1–11.3]

| Module | Responsibility |
|---|---|
| `covariance` | Ledoit–Wolf, factor model, DCC-GARCH (two-step MLE), regime mixture, cross-market lead–lag |
| `tail` | Filtered historical simulation, GJR-t Monte Carlo with t-copula, EVT (GPD MLE); Euler ES contributions |
| `risk_backtest` | Kupiec, Christoffersen, Acerbi–Székely |
| `stress` | Stress library incl. sticker-driven and graph-contagion scenarios |

### 2.9 `s4_portfolio/` — optimiser and allocator [§11.4–11.5]

| Module | Responsibility |
|---|---|
| `black_litterman` | Posterior expected returns from forecasts as views |
| `optimiser` | Conic program: quadratic risk, linear + power-cone (3/2) costs, CVaR constraints. Solver: **Clarabel** (C/C++ interface) or **MOSEK**; OSQP if the 3/2 term is approximated piecewise-linearly |
| `risk_limits` | **◆ G4** post-optimisation checks |
| `allocator` | Risk budgeting across sleeves (Newton iteration on equal/target risk contributions) |

### 2.10 `s5_leverage/` — leverage, decisions, report [§12]

| Module | Responsibility |
|---|---|
| `kelly` | Uncertainty-adjusted fractional Kelly cap |
| `leverage` | Vol target, Kelly, policy cap, margin headroom, drawdown multiplier; **◆ G5** |
| `decision` | State machine, no-trade band, invalidation conditions |
| `governance` | **◆ G6** NO-TRADE rule sets |
| `report` | Report in Markdown/HTML (inja templates), CSV and JSON; XLSX via libxlsxwriter |

### 2.11 `s6_execution/` — execution modelling [§13.1–13.2]

| Module | Responsibility |
|---|---|
| `cost_model` | Spread + square-root impact + commission; recalibration from fills |
| `schedule` | Almgren–Chriss trajectory; auction vs VWAP |
| `tickets` | Order tickets for **manual** placement |
| `fills` | Fill import; implementation shortfall |
| `hard_limits` | Loss, drawdown, divergence, caps — linked against `qp_core` only, never against model libraries |

### 2.12 `s7_monitoring/` and `backtest/`

| Module | Responsibility |
|---|---|
| `drift` | PSI, KS, Page–Hinkley, coverage, alpha-decay alarms |
| `recalibration` | Trigger → action policy |
| `lifecycle` | Candidate → shadow → challenger → champion; **◆ G7** |
| `attribution` | Stage, layer, Brinson–Fachler, factor, execution |
| `backtest/engine` | Event-driven simulator calling the same `s4`–`s6` code; next-session fills, participation caps, lots, delisting haircut; universe formed at the backtest start date |

---

## 3. Key Interfaces

### 3.1 Forecast distribution

```cpp
namespace qp {

inline constexpr std::array<double, 11> kTauGrid{
    0.01, 0.025, 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95, 0.975, 0.99};

struct ForecastDist {
    std::string model_id;                 // "financial/gbm_quantile@1.2.0"
    std::string sleeve;                   // "asx_interday"
    Timestamp   as_of;                    // decision time t
    int         horizon;                  // h, sessions
    std::vector<SecId> sec_ids;           // n
    Eigen::Matrix<double, Eigen::Dynamic, kTauGrid.size(),
                  Eigen::RowMajor> q;     // n × |T|, monotone per row
    Timestamp   label_end;                // used for purging
};

}  // namespace qp
```

### 3.2 Forecast model concept

```cpp
template <class M>
concept ForecastModel = requires(M m, const FeaturePanel& x,
                                 const TargetPanel& y, Timestamp t, int h) {
    { m.id() }                  -> std::convertible_to<std::string_view>;
    { m.layer() }               -> std::same_as<Layer>;      // Financial, Macro, Micro, News
    { m.fit(x, y) }             -> std::same_as<FitReport>;
    { m.predict(x, t, h) }      -> std::same_as<ForecastDist>;
    { m.diagnostics() }         -> std::same_as<Diagnostics>; // convergence, stability → G3
};
```

At runtime, models are held in a type-erased `AnyForecastModel` wrapper so the registry can store all four layers in one container.

### 3.3 Frozen universe

```cpp
class UniverseVersion {
public:
    const std::string& sleeve() const noexcept;
    Timestamp formation_date() const noexcept;
    std::span<const SecId> members() const noexcept;
    std::vector<SecId> active(Timestamp as_of) const;      // never adds names
    std::optional<ExitRecord> exit_of(const SecId&) const;

private:
    UniverseVersion(/* built only by formation */);
    friend class Formation;                                  // sole creator
    friend UniverseVersion apply_safety_exits(const UniverseVersion&,
                                              const SafetyExitInputs&, Timestamp);
    // members, archetypes, tiers, rejection reasons, exits (all immutable)
};
```

### 3.4 Point-in-time read

```cpp
auto table = pit::read("fundamentals", {.as_of = t,
                                        .fields = {"eps", "ebit"},
                                        .sec_ids = universe.active(t)});
```

Raw tables are not exposed by any public header; a leakage test inserts "future-poison" rows and fails if any reach a model input.

---

## 4. Configuration

The YAML files are **identical in schema** to the Python version, so a configuration can be shared between the two implementations. The key file:

```yaml
# configs/universe.yaml
universe:
  mode: static
  formation:
    admission_rank_min: 0.50
    min_size: {asx_interday: 60, us_interday: 80, asx_intraday: 20, us_intraday: 30}
    max_size: {asx_interday: 200, us_interday: 300, asx_intraday: 40, us_intraday: 60}
    history_min_days: 252
  survival: {distress_prob_max: 0.05, icr_min: 2.0, nd_ebitda_max: 4.0,
             runway_min_quarters: 4, altman_z_min: 1.8}
  liquidity: {mdtv_min: {AUD: 5.0e5, USD: 2.0e6}, zero_volume_max: 0.05,
              intraday_spread_max_bps: 10}
  safety_exits:
    distress_prob_max: 0.05
    suspension_max_sessions: 5
    liquidity_collapse: {mdtv_fraction_of_floor: 0.5, sessions: 60}
    replace_exited: false
  reformation: {trigger: manual, advisory_min_fraction: 0.70}
```

---

## 5. Dependencies (managed by vcpkg)

| Area | Libraries |
|---|---|
| Linear algebra | **Eigen 3** (dense and sparse) |
| Special functions, distributions | **Boost.Math** |
| Optimisation (MLE) | **LBFGSpp** (header-only L-BFGS-B) |
| Convex optimisation | **Clarabel** (C/C++ interface) or **MOSEK**; **OSQP** as a QP fallback |
| Gradient boosting | **LightGBM** C API |
| Columnar data | **Apache Arrow / Parquet** C++ |
| Query catalogue | **DuckDB** C++ API |
| Parallelism | **oneTBB** (or OpenMP) |
| HTTP and JSON | **libcurl**, **nlohmann/json**, **json-schema-validator** |
| Embeddings (optional, local) | **ONNX Runtime** C++ API |
| Config, CLI, logging | **yaml-cpp**, **CLI11**, **spdlog**, **fmt** |
| Dates and calendars | C++20 `std::chrono` with time zones (or Howard Hinnant's `date` on older toolchains) |
| Hashing | **xxHash** |
| Reporting | **inja** (templates), **libxlsxwriter** |
| Testing, benchmarks | **GoogleTest**, **RapidCheck** (property tests), **Google Benchmark** |

### Build settings for reproducible numerics

```cmake
target_compile_features(qp_core PUBLIC cxx_std_20)
# Deterministic floating point: no fast-math, no fused multiply-add contraction,
# no host-specific instruction sets, so results match across machines.
target_compile_options(qp_core PUBLIC
    $<$<CXX_COMPILER_ID:GNU,Clang>:-O3 -ffp-contract=off -fno-fast-math>
    $<$<CXX_COMPILER_ID:MSVC>:/O2 /fp:precise>)
```

Every random draw comes from a per-work-item RNG stream, so results are bitwise identical regardless of thread count.

---

## 6. Building and Running

```bash
# 1. Configure and build (vcpkg toolchain picked up by the preset)
cmake --preset release
cmake --build --preset release -j

# 2. Run the test suite
ctest --preset release

# 3. Load or refresh data up to a date
./build/release/quantpipe ingest --sleeve asx_interday --as-of 2026-10-14

# 4. Build the knowledge layer
./build/release/quantpipe knowledge --as-of 2026-10-14

# 5. Form the static universe ONCE per sleeve
./build/release/quantpipe form-universe --sleeve asx_interday --as-of 2026-10-14

# 6. Daily run for one sleeve, or all sleeves with the allocator
./build/release/quantpipe run --sleeve asx_interday --as-of 2026-10-14
./build/release/quantpipe run --all --as-of 2026-10-14

# 7. Backtest (forms its own universe at the start date)
./build/release/quantpipe backtest --sleeve asx_interday --start 2016-01-04 --end 2026-09-30
```

All runs are started manually. The order of operations inside `run` is the same as the Python version:

```text
load configs + UniverseVersion
 → s0_data: ingest, PIT merge, ◆ G2
 → s1_universe::apply_safety_exits (removals only)
 → s2_models → ForecastDist per layer
 → s2e_combine: ◆ G3 → combine → conformal → horizon
 → s3_risk → s4_portfolio: Black–Litterman → optimiser → ◆ G4 → allocator
 → s5_leverage: leverage → ◆ G5 → decisions → ◆ G6
 → s6_execution: tickets (manual placement), hard limits
 → s7_monitoring: drift and attribution
 → report + manifest in data/runs/{run_id}/
```

---

## 7. Testing Strategy

| Test type | What it checks |
|---|---|
| Unit | Each formula against hand-worked values (the methodology's worked examples) |
| Gradient checks | Every analytic gradient (GARCH, Hawkes, hazard, DCC) against central finite differences |
| Reference parity | Estimates against golden values (Section 9) |
| Leakage | Future-poison rows, one-day feature shift, restatements invisible before knowledge time |
| Survivorship | Backtest universes include later-delisted names |
| Static universe | Compile-time: downstream targets cannot include `formation`/`scoring`; runtime: no path adds a member |
| Gates | Pass and fail branches of G1–G7 on synthetic inputs |
| Determinism | Identical output hashes across runs and across 1 vs N threads |
| Sanitisers | Full test suite under AddressSanitizer and UndefinedBehaviorSanitizer |
| Property tests | Invariants such as monotone quantiles, weights summing to 1, PSD covariance |

---

## 8. Suggested Build Order

| Phase | Scope | Done when |
|---|---|---|
| 1 | `math`, `core`, Parquet/DuckDB I/O, PIT store, G2 | 20 consecutive days ingested with G2 green and identical hashes on re-run |
| 2 | `s0_knowledge`, `s1_universe`; formation of ASX interday | Formation reproducible; safety exits tested on historical delistings |
| 3 | Baselines, GARCH, HAR, elastic net, LightGBM; `s2e_combine` | Parity tests pass; G3 decisions for at least three families |
| 4 | `s3_risk`, `s4_portfolio`, `s5_leverage`, `s6_execution`, `backtest` | 40 paper sessions with full gate traces |
| 5 | Macro (DFM, Markov switching, local projections) and micro (panel FE, hierarchical, DML) | Parity tests pass; layers pass G3 or documented as non-eligible |
| 6 | US interday, FX channel, allocator | Cross-sleeve covariance validated |
| 7 | News layer (HTTP LLM client, propagation) | Evaluated only on post-cutoff news |
| 8 | Intraday sleeves (realised measures, Hawkes, ACD) | Extended paper trading; simulated vs realised fills |
| 9 | `s7_monitoring` lifecycle; small live capital | Hard limits tested end to end |

Phases 3 and 5 are noticeably larger than in a Python build, because the estimators are written here rather than imported.

---

## 9. Verifying In-House Estimators

Writing econometric estimators from scratch is where a C++ build is most likely to go wrong, so each one has three independent checks:

1. **Simulation recovery.** Simulate data from known parameters (e.g. a GJR-GARCH-$t$ process, a three-state Markov-switching model, a Hawkes process with branching ratio 0.6) and confirm the estimator recovers them within sampling error over many replications.
2. **Golden reference values.** For each estimator, `tests/reference/` stores inputs and expected outputs produced once by an established implementation (e.g. log-likelihoods and parameter estimates from standard statistical packages). C++ results must match within stated tolerances (e.g. $10^{-8}$ for likelihoods at fixed parameters).
3. **Closed-form cases.** Cases with known answers: Kupiec statistics from the methodology's worked example, Black–Litterman with a single view, DSR from the worked example, conformal coverage on exchangeable data.

An estimator is not used by any stage until all three checks pass.

---

## 10. What Is Deliberately Not in the Repository

- Licensed market data, credentials or API keys (kept in the OS keychain and `data/`, both outside version control).
- Scheduling or automation: every run is started manually.
- Automatic order placement: the pipeline produces tickets; orders are placed by hand.
