# ADVANCE QUANTITATIVE PIPELINE OVERVIEW

**Version 9: concise architecture, stage workflows and control specification**

> **Status:** Research and system-design specification, not evidence of profitability or production readiness. Model thresholds are provisional. Live use requires tested implementation, validated data and broker interfaces, approved risk limits, operational testing and explicit sign-off.

| Scope | Design |
|---|---|
| Markets | Any (ASX & US) |
| Sleeves | ASX interday, US interday, ASX intraday, US intraday |
| Forecast layers | Financial, macroeconomic, microeconomic and news-derived |
| Main flow | Data → fixed research universe → forecasts → validation → risk/optimisation → leverage/pre-trade gates → execution → monitoring/reporting |
| Out of scope | Full source code, provider API instructions and production runbooks |

## Executive Summary

This is a **gated research-to-execution system**, not a single predictive model. Its governing rules are:

1. Use point-in-time data and preserve historical vintages; do not leak future information.
2. Separate company-universe selection from return forecasting.
3. Standardise forecasts as predictive quantiles with metadata and lineage.
4. Validate models out of sample and after costs before giving them weight.
5. Keep portfolio risk, leverage, order limits and kill switches independent of forecasts.
6. Generate reproducible run records, plots and a narrative report from measured outputs.
7. Promote challengers only after validation, shadow testing, approval and a rollback plan.

A sophisticated design does not establish an economic edge. Test the complete system on chronological out-of-sample data and realistic costs before paper trading or live deployment.

## Contents

1. [Pipeline at a Glance](#1-pipeline-at-a-glance)
2. [Design Principles and Gates](#2-design-principles-and-gates)
3. [Stage 0: Point-in-Time Data and Company Knowledge](#3-stage-0-point-in-time-data-and-the-company-knowledge-layer)
4. [Stage 1: Universe and Archetype Selection](#4-stage-1-archetype-builder)
5. [Stage 2: Common Forecast Contract](#5-stage-2-the-common-forecast-representation)
6. [Stage 2A: Financial Models](#6-stage-2a-financial-model)
7. [Stage 2B: Macro Models](#7-stage-2b-macro-model-economy-stickers)
8. [Stage 2C: Micro Models](#8-stage-2c-micro-model-market-labels)
9. [Stage 2D: News Models](#9-stage-2d-news-model-llm-extraction)
10. [Stage 2E: Validation and Combination](#10-stage-2e-validation-eligibility-and-signal-combination)
11. [Stages 3–4: Risk, Optimisation and Sleeve Allocation](#11-stages-34-risk-model-optimiser-and-sleeve-allocator)
12. [Stage 5: Leverage and Position Report](#12-stage-5-leverage-and-position-report)
13. [Stages 6–7: Execution, Controls and Monitoring](#13-stages-67-execution-hard-limits-monitoring-and-learning)
14. [Core Design Decisions](#14-core-design-decisions)
15. [Notation and Open Decisions](#15-notation-glossary-and-open-questions)
16. [Run Contract and Outputs](#16-operating-contract-and-run-outputs)
17. [Implementation Roadmap](#17-implementation-roadmap-and-readiness-gates)

## 1. Pipeline at a Glance

Use this as the system map. The stage diagrams in Sections 3–13 show the detailed process and hand-off.

```mermaid
flowchart TB
    START([RUN CONFIGURATION<br/>Choose date and sleeve<br/>ASX / US · Interday / Intraday])

    DATA["0 · DATA FOUNDATION<br/>Point-in-time inputs · lineage · vintage history<br/>Company knowledge layer · data-integrity firewall G2"]
    UNIVERSE["1 · COMPANY SELECTION<br/>Survival · liquidity · history · archetype fit · G1<br/>Versioned universe; safety exits only"]

    subgraph MODELS["2 · FORECAST IN PARALLEL"]
        direction LR
        FIN["FINANCIAL<br/>Returns · volatility · distress"]
        MACRO["MACROECONOMIC<br/>Regimes · economy stickers"]
        MICRO["MICROECONOMIC<br/>Peers · order flow · spreads"]
        NEWS["NEWS<br/>Events · direction · decay"]
    end

    CONTRACT["COMMON FORECAST FORMAT<br/>Horizon · timestamp · quantiles<br/>Uncertainty · model version · data lineage"]
    VALIDATE["3 · VALIDATE AND COMBINE · G3<br/>Walk-forward tests · calibration · overfit controls<br/>Combine eligible forecasts only"]
    PORTFOLIO["4 · PORTFOLIO CONSTRUCTION · G4<br/>Risk model + cost-aware optimiser<br/>Liquidity · concentration · stress limits"]
    PRETRADE["5 · POSITION AND PRE-TRADE CONTROLS · G5 / G6<br/>Position sizing · leverage · margin headroom<br/>Final data, model, risk and kill-switch check"]
    EXECUTE["6 · EXECUTION<br/>Order sizing · routing · fills · slippage<br/>Independent order and position limits remain active"]
    REPORT["7 · MONITOR AND REPORT<br/>P&L · forecast drift · costs · incidents<br/>Plots · structured run record · narrative report"]
    END([END OF RUN])

    START --> DATA --> UNIVERSE --> MODELS
    FIN --> CONTRACT
    MACRO --> CONTRACT
    MICRO --> CONTRACT
    NEWS --> CONTRACT
    CONTRACT --> VALIDATE --> PORTFOLIO --> PRETRADE --> EXECUTE --> REPORT --> END

    classDef terminal fill:#f3f4f6,stroke:#6b7280,color:#202938,stroke-width:1px;
    classDef stage fill:#eaf2f8,stroke:#477b9e,color:#19364a,stroke-width:1px;
    classDef forecast fill:#edf7ed,stroke:#54845e,color:#1d4026,stroke-width:1px;
    classDef control fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;

    class START,END terminal;
    class DATA,UNIVERSE,CONTRACT,VALIDATE,PORTFOLIO,EXECUTE,REPORT stage;
    class FIN,MACRO,MICRO,NEWS forecast;
    class PRETRADE control;
```

### Model improvement loop

Monitoring may trigger a challenger review, but a challenger never replaces the production model automatically.

```mermaid
flowchart LR
    MON["MONITOR<br/>Detect drift or degradation"] --> TEST["TEST CHALLENGER<br/>Out-of-sample + shadow run"]
    TEST --> G7{"G7<br/>Approved?"}
    G7 -->|Yes| RELEASE["VERSIONED RELEASE<br/>Approval + rollback plan"]
    G7 -->|No / unclear| KEEP["KEEP CURRENT MODEL<br/>Continue monitoring"]

    classDef review fill:#eaf2f8,stroke:#477b9e,color:#19364a,stroke-width:1px;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef safe fill:#edf7ed,stroke:#54845e,color:#1d4026,stroke-width:1px;
    classDef retain fill:#f3f4f6,stroke:#6b7280,color:#202938,stroke-width:1px;
    class MON,TEST review;
    class G7 gate;
    class RELEASE safe;
    class KEEP retain;
```

### Sleeves

| Sleeve | Market / currency | Key concern |
|---|---|---|
| ASX interday | ASX / AUD | Closing-auction execution and limited short borrow |
| US interday | US / USD | FX exposure and overnight gap risk |
| ASX intraday | ASX / AUD | Spread, tick size and available depth |
| US intraday | US / USD | Execution costs and competition for short-horizon signals |

Sleeves share infrastructure but keep separate calendars, cost assumptions, data depth and risk budgets.

| Forecast layer | Interday role | Intraday role |
|---|---|---|
| Financial | Return, volatility and distress | Distress controls and volatility |
| Macro | Regime and factor exposures | Scheduled-release risk and event windows |
| Micro | Industry economics and peer relationships | Order flow, spread, intensity and impact |
| News | Multi-day information drift | Fast-decaying, latency-sensitive event features |

## 2. Design Principles and Gates

| Principle | Implementation |
|---|---|
| Point-in-time | Input knowledge time must be no later than decision time. Filtered states only; smoothing is for retrospective analysis. |
| Common forecast format | Each model produces a predictive distribution and lineage metadata. |
| Validate before weighting | Fit success is not proof of out-of-sample skill or economic value. |
| Conservative credible risk | Use the most conservative risk estimate that passes its own backtests. |
| Default NO-TRADE | Trade only when required checks pass and expected benefit exceeds expected cost. |
| Separate sizing and leverage | Optimiser chooses relative weights; a separate rule controls total exposure. |
| Charge for search | Record trials and account for multiple testing and backtest overfitting. |
| Static universe | Formation runs once per universe version; only specified safety exits may remove names. |

| Gate | Pass condition | Failure action |
|---|---|---|
| **G1 Universe** | Formation filters pass; existing members meet safety rules | Reject at formation or log a safety exit |
| **G2 Data integrity** | No look-ahead, schema, unit, timezone or freshness breach | Quarantine affected name or block affected branch/run |
| **G3 Model eligibility** | Stable, calibrated, out-of-sample skill and positive net-of-cost evidence | Exclude model; no eligible signal means NO-TRADE |
| **G4 Portfolio risk** | Volatility, expected shortfall, concentration, liquidity and stress limits pass | Re-optimise within approved relaxations or scale toward cash |
| **G5 Leverage/margin** | Exposure, financing and stressed margin stay within approved limits | Reduce gross exposure or block new risk |
| **G6 Final no-trade** | Data, model, risk, liquidity, market-state and kill-switch checks pass | Block the order; risk-reducing actions only if policy permits |
| **G7 Promotion** | Challenger passes independent validation and shadow period | Keep current champion |

**Run status:** `PASS` means checks passed; `DEGRADED` means an approved fallback is active; `BLOCKED` means a critical check failed. Every failure receives a machine-readable reason code. If no approved fallback exists, stop the affected run/sleeve.

## 3. Stage 0: Point-in-Time Data and the Company Knowledge Layer

**Purpose:** Build one reproducible, timestamped view of every company and relevant market input before calculating features or forecasts.

```mermaid
flowchart TB
    A([Run configuration and decision cut-off]) --> B[Load market, fundamentals, estimates, macro, news, membership, borrow and margin data]
    B --> C[Standardise identifiers, calendars, currencies, units and time zones]
    C --> D[Store valid time and knowledge time; retain historical vintages]
    D --> E[Apply as-of query and corporate-action rules]
    E --> F{G2 data-integrity checks pass?}
    F -->|Block-level failure| G[Block affected branch or run; record reason code]
    F -->|Name-specific failure| H[Quarantine affected name and retain audit trail]
    F -->|Pass| I[Build versioned Company Knowledge Layer]
    I --> J[Economy stickers, market labels, peer clusters, relationship graph and membership history]
    J --> K([Validated point-in-time data and features with lineage])

    classDef input fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef process fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A,B input;
    class C,D,E,I,J,K process;
    class F gate;
    class G,H stop;
```

### Data timing and integrity

Store each fact with **valid time** (period described) and **knowledge time** (when it became available). A query as of time `t` must return only versions with knowledge time `k ≤ t`; restatements create new vintages rather than overwriting history.

| Source | Timing rule |
|---|---|
| Daily/intraday prices and quotes | Bar end or session close plus publication lag; include delisted names in historical tests |
| Fundamentals and guidance | Announcement/publication timestamp; use a conservative fallback if unavailable |
| Estimates and consensus | First observed timestamp and subsequent revisions |
| Macro data | Official release timestamp and each historical vintage |
| News | Publication time plus processing latency |
| Index membership | Announcement date, not only effective date |
| Borrow and margin | Observation time |

Raw prices are preserved. Adjustment factors for splits, rights and dividends are applied on read; level filters use the unadjusted price actually available at the time. Halted, suspended or zero-volume names are not tradable.

| G2 severity | Examples | Action |
|---|---|---|
| Block | Look-ahead, schema/type mismatch, bad units/timezone, duplicate keys | Stop affected branch/run |
| Quarantine | Stale or incomplete history, unexplained extreme return, unreconciled corporate action | Exclude affected name for this run |
| Warn | Repeated closes, revision bursts | Record warning; do not silently suppress it |

### Company Knowledge Layer

| Component | Contents | Downstream use |
|---|---|---|
| Economy stickers | Dated revenue and cost exposure vectors by economy | Macro models, FX and stress tests |
| Market labels | Industry, product market, listing venue and currency | Universe selection, peer models and constraints |
| Peer clusters | Label- and data-based peer groups | Pooling and relative valuation |
| Relationship graph | Directed customer, supplier, competitor and shared-input links | News propagation, macro transmission and contagion tests |
| Membership history | Listing, delisting and identity changes | Survivorship-safe universes |

Keep revenue and cost stickers separately: a company may sell in one economy and incur costs in another. Version all components point-in-time.

## 4. Stage 1: Archetype Builder

**Purpose:** Select companies suitable for analysis, not forecast returns. Historical formation must include later-delisted companies to avoid survivorship bias.

```mermaid
flowchart TB
    A([Formation date and market scope]) --> B[Build point-in-time candidate list, including later-delisted firms in historical tests]
    B --> C[Apply hard survival and distress filters]
    C --> D{Pass hard survival rules?}
    D -->|No| X[Reject with reason code: DIST, HIST or other rule]
    D -->|Yes| E[Apply sleeve-specific liquidity and tradability screens]
    E --> F{Tradable at required horizon and size?}
    F -->|No| Y[Reject with reason code: LIQ or TRADABILITY]
    F -->|Yes| G[Test data depth, forecastability and stability]
    G --> H[Assign one or more strategy archetypes]
    H --> I[Standardise variables within appropriate peer or sector groups]
    I --> J[Combine category scores and rank eligible names]
    J --> K{Pass admission threshold and sleeve capacity rules?}
    K -->|No| Z[Reject with formation scores and reason code]
    K -->|Yes| L[Assign fixed data tier and create versioned universe]
    L --> M([Frozen research universe])
    M --> N[Monitor safety exits only: distress, delisting, suspension, takeover or liquidity collapse]
    N --> O{Safety exit triggered?}
    O -->|Yes| P[Log exit and retain history; do not replace or re-enter]
    O -->|No| M

    classDef process fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef decision fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef reject fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    classDef output fill:#edf7ed,stroke:#54845e,color:#1d4026;
    class A,B,C,E,G,H,I,J,N process;
    class D,F,K,O decision;
    class X,Y,Z,P reject;
    class L,M output;
```

### Hard filters and score

The order is: **survival/distress → liquidity/tradability → data depth and stability → archetype fit → category score → admission**. Rejection reasons are recorded (for example `DIST`, `LIQ`, `HIST`, `DATA`). Banks and insurers require sector-appropriate capital/funding metrics rather than generic debt/EBITDA rules.

| Category | Example measures |
|---|---|
| Survival | 12-month hazard-model distress probability, distance to default, interest cover, refinancing wall, cash runway |
| Liquidity | Median daily traded value (MDTV), Amihud illiquidity, spread, zero-volume ratio, order-book depth |
| Stability | Beta-break tests, volatility forecastability and usable history |
| Business/market suitability | Profitability/cash conversion, valuation, trend, peer relationships and event coverage as relevant to the archetype |
| Operational/data coverage | Identifier quality, publication times, corporate actions, borrow data and intraday quote quality |

Metrics are robustly standardised within sector/peer group, direction-aligned and combined at category level so a category with more variables does not dominate by count:

$$
z_{ij}=\frac{x_{ij}-\operatorname{median}_{s(i)}(x_j)}{1.4826\,\operatorname{MAD}_{s(i)}(x_j)},\qquad q_i=\sum_j\omega_j\Phi(z_{ij}),\quad \sum_j\omega_j=1.
$$

A company is admitted only if it passes all hard gates and clears the configured formation threshold/capacity for that sleeve. Scores and membership are then frozen for that universe version.

| Archetype | Typical fit |
|---|---|
| Momentum/trend | Persistent 12–1 month trend, adequate liquidity, manageable idiosyncratic noise |
| Quality-value | Profitability, cash conversion, low accruals and sector-relative valuation |
| Mean-reversion/pairs | Stable peer relationship and spread half-life; borrow/costs permit trading |
| Event-driven | Reliable event stream, measurable announcement responses and sufficient coverage |
| Intraday liquid large-cap | Tight spreads, depth, activity and potentially forecastable short-horizon volatility |

### Static membership and safety exits

No score-based removal, replacement or re-entry is allowed inside a universe version. A deliberate re-formation creates a new version and requires validation.

| Exit code | Trigger |
|---|---|
| `EXIT-DIST` | Distress probability exceeds approved threshold or insolvency event |
| `EXIT-DLST` / `EXIT-SUSP` | Delisting or suspension beyond policy tolerance |
| `EXIT-TKO` | Approved takeover/scheme changes the security's status |
| `EXIT-LIQ` | Liquidity remains below the formation floor for the configured period |

Exited names stay in historical data and attribution. Positions are closed through normal execution controls.

### Data-depth tiers

| Tier | Typical allocation | Data use |
|---|---|---|
| T1 | Top ~40 per market | Trades/quotes for intraday and realised measures |
| T2 | Next ~120 | One-minute bars for intraday volatility/costs |
| T3 | Remaining names | Daily data; tick data selectively around events |

Sizes are budget examples, not fixed requirements. Allocate expensive data by expected analytical value, signal need, storage and provider cost; tier membership stays fixed within the universe version.

## 5. Stage 2: The Common Forecast Representation

All model families must use one adapter so forecasts can be compared, combined, calibrated and passed into risk calculations consistently.

```mermaid
flowchart TB
    A[Financial model output] --> F[Forecast adapter]
    B[Macro model output] --> F
    C[Micro model output] --> F
    D[News model output] --> F
    F --> G[Attach security, decision time, horizon, model version and data lineage]
    G --> H[Map outputs to common excess-return target and quantile grid]
    H --> I{Schema, timestamps and target definition valid?}
    I -->|No| X[Reject output; log contract failure]
    I -->|Yes| J[Check quantile ordering and repair crossings by monotone rearrangement]
    J --> K[Store label end date for purging and embargo]
    K --> L([Comparable forecast distributions for Stage 2E])

    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef process fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A,B,C,D model;
    class F,G,H,J,K,L process;
    class I gate;
    class X stop;
```

For security `i`, decision time `t` and horizon `h`, emit quantiles on a shared grid:

$$
\mathcal T=\{0.01,0.025,0.05,0.10,0.25,0.50,0.75,0.90,0.95,0.975,0.99\}.
$$

The return target is benchmark-relative excess return from the next executable session:

$$
y^{(h)}_{i,t}=\sum_{s=2}^{h+1}(r_{i,t+s}-r_{b,t+s}).
$$

Each forecast includes security, decision timestamp, horizon, target definition, quantile values, model/version, training window, feature version, data lineage and label end date. Reject invalid timestamps/schema or crossing quantiles; record the label end date for purging.

Use average quantile pinball loss as the primary distribution score; also track interval coverage, sharpness and the probability integral transform. More elaborate models are not admitted solely because they fit in sample.

## 6. Stage 2A: Financial Model

**Purpose:** Estimate return distributions, volatility, distress and changing factor exposures. Stage 2E—not this stage—decides model eligibility.

```mermaid
flowchart TB
    A[Point-in-time prices, returns, fundamentals and benchmark data] --> B[Create leakage-safe features and target labels]
    B --> C[Fit simple benchmarks first]
    C --> D[Estimate return quantile candidates: regularised, tree-based or other tested models]
    C --> E[Estimate volatility: GJR-GARCH, EGARCH and HAR when realised data exist]
    C --> F[Estimate distress probability with firm-year hazard model]
    C --> G[Estimate changing factor exposures with filtered state-space model]
    D --> H[Check fit status, parameter stability and out-of-sample forecasts]
    E --> H
    F --> H
    G --> H
    H --> I[Check distress estimates and create survival-risk feature]
    H --> J[Select candidate return and volatility outputs for common forecast adapter]
    I --> K([Distress probability for risk and safety logic])
    J --> L([Return quantiles and volatility forecasts to Stage 2])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef output fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    class A,B,C data;
    class D,E,F,G,H,J model;
    class I,K,L output;
```

| Task | Candidate methods | Baseline / control |
|---|---|---|
| Return quantiles | Elastic net, quantile regression, gradient boosting (LightGBM/XGBoost), quantile random forest | Zero excess return, rolling mean and factor model |
| Distress | Discrete-time firm-year hazard model | Compare with simple threshold/logistic baseline |
| Daily volatility | GJR-GARCH or EGARCH with Student-t innovations | EWMA variance |
| Intraday realised volatility | HAR model on realised variance/jumps | Daily-only volatility baseline |
| Dynamic exposures | Filtered state-space/Kalman model | Static rolling beta |

**Distress hazard:**

$$
P(\text{default in }(t,t+1]\mid x_{i,t})=\Lambda(\alpha(t)+\beta^\top x_{i,t}),
$$

with distance to default, profitability, leverage, cash runway, returns and volatility as inputs. Use the output for survival gates, optimiser penalties and position reporting.

**GJR-GARCH:**

$$
r_t=\sigma_tz_t,\qquad \sigma_t^2=\omega+(\alpha+\gamma\mathbf1\{r_{t-1}<0\})r_{t-1}^2+\beta\sigma_{t-1}^2.
$$

Student-t innovations capture fat tails; `γ` captures asymmetric volatility response. For realised data, HAR uses daily, weekly and monthly realised-variance averages plus a jump component. Filter changing factor exposures using data available up to `t`; never use smoothed future-informed estimates for live signals.

**Feature groups:** earnings revisions/surprises, profitability and cash conversion, accruals, margins, investment and asset growth, leverage, valuation, momentum/reversal, idiosyncratic volatility and distress. Standardise features inside each training fold.

## 7. Stage 2B: Macro Model (Economy Stickers)

**Purpose:** Combine economy-level states with each firm's dated revenue/cost exposures and estimated sensitivities.

```mermaid
flowchart TB
    A[Point-in-time macro releases, market variables and consensus data] --> B[Standardise by economy and retain release vintages]
    B --> C[Estimate economy states with dynamic factor models]
    B --> D[Estimate regime probabilities and mixed-frequency nowcasts]
    C --> E[Combine state estimates with each firm's dated revenue and cost stickers]
    D --> E
    E --> F[Estimate company sensitivities to macro factors]
    F --> G[Shrink noisy sensitivities toward sticker-based priors]
    E --> H[Translate scheduled release surprises into event effects]
    G --> I[Combine states, regimes, sensitivities and event effects]
    H --> I
    I --> J[Propagate scenario shocks through sector and relationship channels]
    J --> K[Include explicit FX and input-cost exposure]
    K --> L([Macro contribution and predictive quantiles for Stage 2])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef output fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    class A,B data;
    class C,D,E,F,G,H,I,J,K model;
    class L output;
```

| Component | Method / output |
|---|---|
| Economy states | Dynamic factor model for activity, inflation, rates, labour, credit and financial conditions; use the Kalman filter for ragged-edge releases |
| Regimes | Markov-switching model with filtered probabilities for expansion, slowdown and stress |
| Nowcasts | MIDAS for mixed-frequency targets such as GDP; Bayesian VAR as a challenger |
| Company exposure | Sticker-weighted state: $f^{mac}_{i,t}=\sum_c e_{i,c}f_{c,t}$ |
| Sensitivities | Estimate firm responses to macro factors; shrink noisy coefficients toward sticker/industry priors |
| Scheduled events | Standardised actual-minus-consensus surprise, with release-time volatility controls for intraday sleeves |
| Transmission/stress | Local projections and the shared relationship graph to convert macro scenarios into firm shocks |
| FX | Explicit revenue/cost currency exposure and portfolio-level USD translation exposure |

Shrink noisy firm sensitivities toward prior estimates:

$$
\hat b^{post}_{ij}=\lambda_{ij}\hat b^{OLS}_{ij}+(1-\lambda_{ij})b^{prior}_{ij},\qquad
\lambda_{ij}=\frac{\tau_j^2}{\tau_j^2+\mathrm{se}(\hat b_{ij})^2}.
$$

Standardised release surprise: $S_{c,t}=(\mathrm{actual}-\mathrm{consensus})/\mathrm{sd}(\mathrm{surprise})$. Estimate horizon-specific shock responses with local projections; include sector, sticker, input-cost and FX channels. Use filtered, point-in-time states only.

## 8. Stage 2C: Micro Model (Market Labels)

Interday micro models describe product-market economics. Intraday micro models describe price formation and execution.

```mermaid
flowchart TB
    A[Point-in-time company, peer, market and trading data] --> B{Which sleeve is running?}
    B -->|Interday| C[Build company-market panel: prices, demand, input costs, margins and competition]
    C --> D[Fit panel and hierarchical models; use cross-fitted DML where justified]
    D --> E[Estimate peer and earnings-surprise responses]
    B -->|Intraday| F[Clean trades, quotes and order-book observations]
    F --> G[Estimate trade intensity, durations, order-flow imbalance and price impact]
    G --> H[Compute realised variance, jumps, kernels and semivariances]
    E --> I[Combine economic response, market labels and forecast horizon]
    H --> J[Produce liquidity, execution-cost and short-horizon market-state features]
    J --> I
    I --> K([Micro contribution and predictive quantiles for Stage 2])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef decision fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef output fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    class A data;
    class B decision;
    class C,D,E,F,G,H,I,J model;
    class K output;
```

| Branch | Main methods and features | Output |
|---|---|---|
| Interday | Firm/market panel and hierarchical models; demand, input costs, prices, margins, market share and competitive intensity; cross-fitted DML only where justified | Peer/earnings response and firm-level micro contribution |
| Intraday | Trade/quote cleaning; trade intensity and durations; order-flow imbalance; spread, depth and price impact | Short-horizon features, liquidity state and execution-cost estimates |

Two-way panel baseline:

$$
r_{it}=\alpha_i+\lambda_t+\beta^\top x_{it}+\epsilon_{it},
$$

with firm- and date-clustered uncertainty. Hierarchical estimates shrink low-history firms toward market-label/peer-group estimates.

Realised measures from intraday data include realised variance, bipower variation, jumps, semivariance and realised kernels. Use them only when quote/trade timestamps and market sessions are reliable; otherwise fall back to validated daily estimates.

## 9. Stage 2D: News Model (LLM Extraction)

Use an LLM to extract structured facts from timestamped text—not to issue orders or directly declare a profitable trade.

```mermaid
flowchart TB
    A[News stories, transcripts and exchange announcements] --> B[Check source, publication time, processing latency, prompt and model version]
    B --> C{Point-in-time and model-version rules valid?}
    C -->|No| X[Block use in the affected historical test or run]
    C -->|Yes| D[Link entities and de-duplicate stories into event records]
    D --> E[LLM extracts fixed-schema fields from untrusted text]
    E --> F{Schema and timestamp checks pass?}
    F -->|No| X
    F -->|Yes| G[Measure novelty and compare with price move since publication]
    G --> H[Propagate effects across the relationship graph with decay]
    H --> I[Feed structured features to calibrated model trained on realised returns]
    I --> J[Estimate shock probability and return quantiles]
    J --> K([News contribution to common forecast adapter])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A,B,D data;
    class E,G,H,I,J,K model;
    class C,F gate;
    class X stop;
```

### Processing and event schema

1. Check source, publication timestamp, processing latency and model/prompt version.
2. Deduplicate stories, link entities, compare with prior disclosures and estimate novelty/priced-in status.
3. Extract structured fields; pass them to calibrated statistical models.

| Field | Meaning |
|---|---|
| Entity / source / time | Company identifier, source credibility, publication time and processing latency |
| Event | Type, channel, novelty and whether expected/already priced |
| Impact | Direction, magnitude, confidence and persistence/decay horizon |
| Evidence | Supporting text span or source reference for audit |

Propagate material effects through the shared relationship graph. For edge weight $W_{ij}$ and decay $\delta$, a first-order propagation is proportional to $W_{ij}\delta$; enforce stability of the propagation network (spectral radius below one) to avoid exploding effects.

A calibrated logistic model estimates shock probability, for example:

$$
P(|AR_{i,t}|>2\hat\sigma_i\mid z_d)=\Lambda(\theta^\top z_d),
$$

where features include novelty, direction, magnitude, source credibility, residual surprise and propagated effects. Validate with Brier scores and reliability curves.

**Horizon policy:** intraday news usually widens volatility or reduces exposure unless latency tests show a directional edge; interday models may test slower diffusion/drift.

**Leakage controls:** freeze the LLM/prompt during evaluation, obey publication/training cut-offs, use only text available at decision time, and validate prospectively through paper/live shadow runs.

## 10. Stage 2E: Validation, Eligibility and Signal Combination

Validation occurs before a candidate model receives any combination weight.

```mermaid
flowchart TB
    A[Archived candidate forecasts and matched realised outcomes] --> B[Create chronological walk-forward folds]
    B --> C[Purge overlapping labels and apply embargo]
    C --> D[For each candidate, test benchmarks, calibration, stability, net-of-cost skill and overfitting]
    D --> E{Candidate passes G3 eligibility?}
    E -->|Yes| F[Add candidate to eligible model set]
    E -->|No| X[Exclude candidate and record reason code]
    F --> G[Finish eligibility assessment for all candidates]
    X --> G
    G --> H{Eligible model set non-empty?}
    H -->|No| N([NO-TRADE for affected name or sleeve])
    H -->|Yes| I[Estimate inverse-loss, stacking or regime-conditional weights]
    I --> J[Combine predictive quantiles; shrink weights when evidence is weak]
    J --> K[Apply conformal calibration and choose horizon by net information ratio]
    K --> L([Combined calibrated forecast distributions for risk and optimisation])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A,B,C data;
    class D,F,G,I,J,K,L model;
    class E,H gate;
    class X,N stop;
```

### Validation protocol

- Use chronological walk-forward folds; train only on prior data.
- Purge training labels overlapping each test window and embargo at least the forecast horizon.
- Tune hyperparameters inside training folds; use outer test blocks once.
- Compare with relevant baselines and deduct realistic spread, commission, borrow and impact costs.

| Test / metric | Question |
|---|---|
| Pinball loss / CRPS and coverage | Is the predictive distribution accurate and calibrated? |
| Diebold–Mariano / Giacomini–White | Is forecast skill better, possibly conditional on regime? |
| Hansen SPA / Model Confidence Set | Does a candidate beat the benchmark after model search? |
| Deflated Sharpe / PBO | Could apparent trading performance be due to repeated trials/overfitting? |
| Stability and residual checks | Do parameters, errors and signs remain credible across folds? |

### G3 eligibility defaults

| Criterion | Provisional research rule |
|---|---|
| Fit | Converged; acceptable diagnostics; no unexplained boundary estimates |
| Stability | Coefficients/signs reasonably stable across folds |
| Calibration | 90% interval coverage near target (e.g. within ±3 percentage points) |
| Skill | Out-of-sample improvement against suitable baseline/model set |
| Economic value | Positive net-of-cost value; costs and turnover included |
| Overfitting | Record all trials; apply PBO/DSR or comparable controls |

Thresholds must be calibrated per sleeve and approved; they are not universal constants. If the eligible set is empty, output `NO-TRADE` for the affected name/sleeve.

### Combination and calibration

Candidate approaches include inverse-loss weights, stacking and regime-conditional weights. Shrink estimated weights toward equal weights when evidence is weak. Quantile averaging:

$$
\hat q_\tau=\sum_kw_k\hat q_{k,\tau},\qquad w_k\ge0,\quad\sum_kw_k=1.
$$

Apply conformal calibration on held-out data and choose the horizon with the best net information ratio after costs, capacity and portfolio constraints. Track signal decay and re-evaluate the sleeve assignment if the estimated half-life changes materially.

## 11. Stages 3–4: Risk Model, Optimiser and Sleeve Allocator

The optimiser uses only risk estimates that passed their own checks. Liquidity and tail-risk limits are not relaxed merely to produce a feasible solution.

```mermaid
flowchart TB
    A[Combined eligible forecasts, current positions, holdings and cost estimates] --> B[Estimate covariance using shrinkage, factor, dynamic-correlation and regime models]
    B --> C[Calculate volatility, expected shortfall and stress losses]
    A --> D[Convert forecasts and uncertainty into expected-return views]
    D --> E[Build cost-aware portfolio objective with distress penalties]
    C --> F[Supply checked risk estimates and binding risk limits]
    E --> G[Optimise each sleeve with position, turnover, sector, country, FX, liquidity and tail-risk constraints]
    F --> G
    G --> H{Solver successful and G4 limits satisfied?}
    H -->|Yes| I[Accept target weights for the sleeve]
    H -->|No| J[Relax only approved constraints and rerun optimiser: turnover, then sector, then tracking error]
    J --> K{Feasible after permitted relaxation?}
    K -->|No| L[Hold current weights and flag the sleeve; do not manufacture a solution]
    K -->|Yes| I
    I --> M[Allocate capital across sleeves by risk budget and evidence, within sleeve caps]
    M --> N([Target weights and risk diagnostics for Stage 5])

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef model fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A data;
    class B,C,D,E,F,G,J,M model;
    class H,K gate;
    class I,N output;
    class L stop;
```

### Risk estimation

| Method | Role |
|---|---|
| Ledoit–Wolf shrinkage | Stabilise sample covariance |
| Factor covariance | Explain market, sector, style, macro/sticker and FX exposures |
| DCC-GARCH | Allow correlations to vary over time |
| Regime mixture | Blend calm/stress covariance using filtered regime probabilities |
| Filtered historical simulation / Monte Carlo / EVT | Independent tail-risk views; the most conservative credible estimate binds |

$$
\mathrm{VaR}_\alpha=-\inf\{x:P(R_p\le x)>1-\alpha\},\qquad
\mathrm{ES}_\alpha=-E[R_p\mid R_p\le-\mathrm{VaR}_\alpha].
$$

Backtest VaR coverage/independence and ES. Demote an engine that fails. Cross-market covariance must account for asynchronous ASX and US sessions via multi-day returns or lead–lag terms.

### Stress library

Include global crisis/recession/rate-shock history, China activity shocks, AUD moves, US overnight gaps, liquidity drought (lower volume/wider spreads) and relationship-graph contagion. Apply shocks through company-specific stickers and sector exposures.

### Portfolio optimisation

Use Black–Litterman to blend equilibrium returns with confidence-weighted forecast views:

$$
\mu_{BL}=\left[(\tau\Sigma)^{-1}+P^\top\Omega^{-1}P\right]^{-1}
\left[(\tau\Sigma)^{-1}\pi+P^\top\Omega^{-1}q\right].
$$

Cost-aware objective:

$$
\max_w\ \mu_{BL}^\top w-\frac{\gamma}{2}w^\top\hat\Sigma w
-\sum_i(c_i|\Delta w_i|+\eta_i|\Delta w_i|^{3/2})
-\kappa\sum_i p_i^{dist}|w_i|.
$$

Constraints cover position/liquidity capacity, turnover, sector/country/sticker, factor/FX exposures, borrow and stress/CVaR. Check solver status and residuals. If infeasible, relax only approved constraints (e.g. turnover, then sector, then tracking error); never relax liquidity or tail-risk limits. If still infeasible, hold current positions and flag the sleeve.

**Provisional examples only:** volatility no more than 1.1× target; single name ≤8%; sector ≤30%; exit within three days at 20% ADV. Configure and validate limits separately for each sleeve/instrument.

### Sleeve allocation

Allocate capital across sleeves by risk budget and out-of-sample evidence, with bounds and a hard cap on intraday risk. Budget USD/FX exposure explicitly or hedge it. Do not allow intraday drawdown to silently consume interday capacity.

## 12. Stage 5: Leverage and Position Report

Optimisation determines relative weights; this stage determines permitted total exposure and whether each proposed trade should be submitted.

```mermaid
flowchart TB
    A[Target weights plus current positions and portfolio NAV] --> B[Estimate uncertainty-adjusted fractional Kelly cap]
    B --> C[Calculate leverage limits from target volatility, policy maximum and margin headroom]
    C --> D[Apply drawdown control and financing or borrow costs]
    D --> E{G5 leverage, margin and financing checks pass?}
    E -->|No| F[Reduce gross exposure and recheck]
    F --> G{Within limits after reduction?}
    G -->|No| X[Block new risk; retain only risk-reducing actions allowed by policy]
    G -->|Yes| H[Convert target weights into position changes]
    E -->|Yes| H
    H --> I[Compare expected benefit with spread, commission and market-impact cost]
    I --> J[Suppress changes inside the no-trade band]
    J --> K{G6 data, model, risk, liquidity, market-state and kill-switch checks pass?}
    K -->|No| Y[NO-TRADE or documented risk-reducing action]
    K -->|Yes| L[Create approved order intents]
    X --> M[Generate position and sleeve report with reason codes]
    Y --> M
    L --> M
    M --> N([Report and order intents for Stage 6])

    classDef process fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    classDef output fill:#edf7ed,stroke:#54845e,color:#1d4026;
    class A input;
    class B,C,D,H,I,J process;
    class E,G,K gate;
    class F,X,Y stop;
    class L,M,N output;
```

### Kelly cap and leverage rule

Full Kelly $f^*=\mu/\sigma^2$ is too sensitive to estimation error to use as a direct target. Apply fractional Kelly and account for uncertainty in expected return:

$$
f=c\frac{\mu_{BL}}{\sigma^2+\mathrm{Var}(\hat\mu)},\qquad c\in[0.25,0.5]\ \text{(research range)}.
$$

Overall leverage is the lowest permitted bound, then reduced by drawdown:

$$
L_t=\min\left\{\frac{\sigma^*}{\hat\sigma_{p,t}},\ f_t,\ L_{max},\ L_t^{margin}\right\}
\max\left\{0,1-\frac{DD_t}{DD_{max}}\right\}.
$$

Overnight sizing must include gap risk; US positions held during Sydney night hours are a key case. Financing, borrow costs and stressed margin headroom are included in expected net returns and G5.

### Decision and position report

Trade only if expected benefit over the chosen horizon exceeds spread, commission and market-impact cost by the configured margin. Suppress changes within the no-trade band. Each position records invalidation conditions (forecast reversal, tail-quantile breach, stress-regime change, adverse news or rising distress probability).

Every report—trade or `NO-TRADE`—contains:

- Current/target weight, action and reason code.
- Median and interval forecasts, model-layer contributions and model agreement.
- Volatility, expected-shortfall contribution and main stress scenarios.
- Distress probability, sticker/sector/factor/FX exposures and expected costs.
- Gross/net exposure, leverage components, margin headroom, drawdown state and G5/G6 results.

## 13. Stages 6–7: Execution, Hard Limits, Monitoring and Learning

Execution uses approved order intents only. Hard limits are independent of forecast, optimisation and monitoring models.

```mermaid
flowchart TB
    A[Approved order intents from Stage 5] --> B[Refresh market state and verify order, security, venue and position limits]
    B --> C{Independent hard limits and kill switch permit order?}
    C -->|No| X[Reject or cancel order; halt new risk if required]
    C -->|Yes| D[Choose execution schedule using urgency, liquidity and cost estimates]
    D --> E[Interday: closing auction or VWAP; intraday: limit orders and participation caps]
    E --> F[Submit through broker adapter and capture acknowledgements]
    F --> G[Monitor fills, rejects, cancellations and partial execution]
    G --> H[Reconcile fills, cash and positions against broker records]
    H --> I{Reconciliation complete and consistent?}
    I -->|No| Y[Block further orders and raise operational incident]
    I -->|Yes| J[Calculate fees, slippage and implementation shortfall]
    J --> K([Confirmed positions and execution records for monitoring])

    classDef input fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef process fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A input;
    class B,D,E,F,G,H,J,K process;
    class C,I gate;
    class X,Y stop;
```

### Independent safety controls

```mermaid
flowchart LR
    A[Proposed or live order] --> B[Per-name, gross, leverage and participation limits]
    B --> C[Daily loss and drawdown checks]
    C --> D[Model-reality divergence and market-state checks]
    D --> E{Any hard limit breached?}
    E -->|No| F[Permit activity within approved limits]
    E -->|Yes| G[Block new risk, cancel orders or flatten as policy requires]
    G --> H[Log incident and require manual review before restart]

    classDef check fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef safe fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A,B,C,D check;
    class E gate;
    class F safe;
    class G,H stop;
```

### Hard limits and monitoring

| Control | Illustrative behaviour |
|---|---|
| Name/gross leverage/participation | Reject orders beyond approved limits |
| Kill switch | Cancel open orders and block new risk on breach, stale data or operational incident |
| Reconciliation | Block further orders if broker fills, cash or positions do not reconcile |
| Drawdown | Reduce risk according to the approved drawdown policy |
| Monitoring | Track P&L, forecast calibration, feature drift, risk backtests, execution error, slippage and data quality |

Example alerts (calibrate per sleeve): 90% interval coverage outside 87–93% for 20 sessions; alpha half-life more than 50% shorter than at admission; VaR backtest p-value below 0.01; realised execution-cost error above 30%. Alerts trigger the configured response, not automatic model promotion.

### Monitoring and controlled learning

```mermaid
flowchart TB
    A[Run records, forecasts, realised returns, fills, costs and positions] --> B[Calculate P&L attribution, calibration, feature drift, risk backtests and execution error]
    B --> C[Generate per-security and per-sleeve plots from versioned run data]
    B --> D[Create narrative report from structured results and verified metrics only]
    B --> E{Monitoring threshold or material degradation triggered?}
    E -->|No| F[Continue monitoring and retain current champion]
    E -->|Yes| G[Apply approved safe response: recalibrate, reduce exposure or switch risk engine]
    G --> H[Train and evaluate a challenger in walk-forward tests]
    H --> I[Run challenger in shadow without order authority]
    I --> J{G7 approved and calibration intact?}
    J -->|Yes| K[Versioned promotion for future runs with rollback plan]
    J -->|No or unclear| L[Retain champion; document failure and continue monitoring]
    F --> M([Auditable run package])
    D --> M
    C --> M
    K --> M
    L --> M
    M --> N[Review attribution and update research hypotheses]
    N --> O[Universe membership unchanged unless deliberate re-formation is approved]

    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef process fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef safe fill:#f3f4f6,stroke:#6b7280,color:#202938;
    class A data;
    class B,C,D,G,H,I,N,O process;
    class E,J gate;
    class F,K,L,M safe;
```

A model may move through `candidate → G3 eligible → shadow → challenger → G7 approved champion → retired`. Promotion requires independent review, a pre-agreed shadow period (e.g. at least 60 sessions), intact calibration, evidence against the champion and a rollback path. A challenger never replaces the champion automatically.

Performance attribution should separate selection, forecasting, portfolio allocation, factor/sector/country effects and execution shortfall. Findings can inform later research but must not alter a frozen universe version silently.

## 14. Core Design Decisions

- Quantile distributions and metadata provide a common interface for all forecast layers.
- Validation precedes eligibility and combination; trial counts and net-of-cost results are recorded.
- Economy stickers are versioned revenue/cost exposures; one relationship graph supports macro transmission, news propagation and contagion stress.
- The universe is static per version; safety exits are logged and do not trigger replacement.
- Data depth follows model need and provider/storage cost.
- Relative weights, total leverage, and order authorisation are separate decisions.
- Hard limits and kill switches remain independent of model outputs.
- AI-generated narrative reports explain structured results only; they do not invent metrics or issue orders.

## 15. Notation, Glossary and Open Questions

| Symbol/term | Meaning |
|---|---|
| $t$, $k$, $	au$ | Decision time, knowledge time and valid time |
| $h$, $r_{i,t}$ | Forecast horizon and security return |
| $\mathbf e_i$, $f_{c,t}$ | Economy sticker vector and economy state factor |
| $\hat q_	au$, CRPS | Forecast quantile and distribution scoring rule |
| $p_i^{dist}$ | Estimated distress probability |
| $w$, $\Sigma$, $L_t$ | Portfolio weights, covariance and leverage |
| $DD_t$, $b_i$ | Drawdown and no-trade band |
| Purging/embargo | Removing overlapping labels/adjacent observations during validation |
| DSR/PBO/SPA/MCS | Multiple-testing and model-comparison diagnostics |

### Decisions required before implementation

1. Instruments and capital structure: cash equities on margin, CFDs or futures; available capital and financing arrangements.
2. Per-sleeve target volatility, maximum drawdown, per-name/sector/gross limits and margin buffer.
3. US-market broker and provider; licensed data coverage and intraday data cost/depth.
4. News sources and latency requirements; whether intraday news can be used directionally.
5. Long-only versus long-short and permitted borrow, especially for ASX names.
6. Whether universe re-formation is only by deliberate decision or on a defined schedule.
7. Whether model families are fixed or open to a challenger process governed by G3/G7.

## 16. Operating Contract and Run Outputs

Every run creates a unique `run_id` and records:

| Record | Required fields |
|---|---|
| Run/config | Start/end timestamps, code revision, config version, market, sleeve, cutoff, calendar and horizons |
| Data lineage | Provider/dataset, extraction and observation timestamps, knowledge-time rule, row counts and schema version |
| Universe | Universe version, eligible names, exits and reason codes |
| Models/validation | Model/feature versions, training window, fit status, fold/purge/embargo settings, benchmark/calibration/net-cost results and trial count |
| Portfolio/decision | Forecasts, current/target weights, costs, risk contributions, binding constraints, `TRADE`/`REDUCE`/`HOLD`/`NO-TRADE` and reason codes |
| Execution | Order intents, submissions/fills, timestamps, fees, cancellations/rejections and implementation shortfall |
| Outputs | Per-security and per-sleeve plots, diagnostics, warnings and narrative report |

**Terminal status:**
- `PASS`: required data, models, risk and execution checks passed.
- `DEGRADED`: a non-critical component failed and an approved fallback is used; trading remains subject to every hard gate.
- `BLOCKED`: a critical check failed; do not initiate new risk.

The narrative report must be generated from structured outputs. AI/ML may summarise measured results but may not invent numbers, override a gate or issue an order. Plot/report failures are logged and never conceal safety failures. Fallbacks must be tested and cannot bypass data integrity, margin, hard exposure limits, kill switches or reconciliation.

## 17. Implementation Roadmap and Readiness Gates

| Phase | Deliverable | Exit criterion |
|---|---|---|
| 1. Data foundation | Point-in-time schemas, calendars, identifiers, corporate actions and lineage | As-of queries and schema/timestamp/unit tests pass |
| 2. Baseline research | Benchmarks, costs and frozen-universe builder | Survivorship/look-ahead tests pass; clean runs reproduce results |
| 3. Validation | Walk-forward folds, purging/embargo and trial registry | Leakage tests work; trials and fold boundaries logged |
| 4. Forecast layers | Financial, macro, micro and news modules under common contract | Baselines, calibration and model failure states are measured |
| 5. Risk/optimisation | Covariance/ES, stress library, optimiser and sleeve allocator | Constraint tests pass; infeasibility cannot create tradable weights |
| 6. Execution simulation | Scheduling, fill/cost model and shortfall attribution | Costs/fills are stress-tested and documented |
| 7. Paper trading | Scheduled end-to-end runs without capital at risk | Reconciliation, stale-data, restart, latency and kill-switch tests pass |
| 8. Controlled deployment | Small approved allocation with monitoring and broker permissions | Written sign-off, limits, rollback and incident response tested |
| 9. Challenger lifecycle | Shadow tests and champion/challenger governance | Out-of-sample criteria met; versioned rollback available |

A stage is complete only when its input/output schema, unit/failure tests, point-in-time handling, versioned configuration, logs/reason codes, reproducible outputs, benchmark/invariant and downstream failure action are documented.

**Final rule:** research models propose forecasts; the validated pipeline proposes target positions; only independent risk and execution controls authorise orders.
