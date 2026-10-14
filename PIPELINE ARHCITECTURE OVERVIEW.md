# Multi-Market Quantitative Investment Pipeline

**Version 2: methodology and model specification**

| | |
|---|---|
| Scope | ASX and US equities · interday and intraday sleeves · live capital |
| Structure | Archetype selection → per-company forecasting (financial, macro, micro, news) → signal combination → risk model → portfolio optimiser → leverage and position report → execution → monitoring |
| Purpose of this document | Explain *what* each stage does, *why* it is designed that way, and *which models* it uses, with the mathematics written out and worked examples |
| Out of scope | Code, repository layout, scheduling and automation |

> **How to read this document.** Each stage opens with its design principle, then a qualitative explanation (what problem the stage solves and why the chosen approach suits it), then the formal model specification, then a worked example where useful. Section 14 lists the improvements made relative to version 1. All numerical examples are illustrative.

---

## Contents

1. [Pipeline at a Glance](#1-pipeline-at-a-glance)
2. [Design Principles and Gates](#2-design-principles-and-gates)
3. [Stage 0: Point-in-Time Data and the Company Knowledge Layer](#3-stage-0-point-in-time-data-and-the-company-knowledge-layer)
4. [Stage 1: Archetype Builder](#4-stage-1-archetype-builder)
5. [Stage 2: The Common Forecast Representation](#5-stage-2-the-common-forecast-representation)
6. [Stage 2A: Financial Model](#6-stage-2a-financial-model)
7. [Stage 2B: Macro Model (Economy Stickers)](#7-stage-2b-macro-model-economy-stickers)
8. [Stage 2C: Micro Model (Market Labels)](#8-stage-2c-micro-model-market-labels)
9. [Stage 2D: News Model (LLM Extraction)](#9-stage-2d-news-model-llm-extraction)
10. [Stage 2E: Validation, Eligibility and Signal Combination](#10-stage-2e-validation-eligibility-and-signal-combination)
11. [Stages 3–4: Risk Model, Optimiser and Sleeve Allocator](#11-stages-34-risk-model-optimiser-and-sleeve-allocator)
12. [Stage 5: Leverage and Position Report](#12-stage-5-leverage-and-position-report)
13. [Stages 6–7: Execution, Hard Limits, Monitoring and Learning](#13-stages-67-execution-hard-limits-monitoring-and-learning)
14. [Improvements Relative to Version 1](#14-improvements-relative-to-version-1)
15. [Notation, Glossary and Open Questions](#15-notation-glossary-and-open-questions)

---

## 1. Pipeline at a Glance

```
Stage 0   Point-in-time data  +  Company Knowledge Layer
              (economy stickers, market labels, peer clusters, relationship graph)
   |
Stage 1   Archetype Builder: survival filter -> liquidity -> archetype fit
              -> scoring with hysteresis -> universe per sleeve      ◆ G1
   |
Stage 2   Forecast models, each emitting a predictive distribution
              2A Financial | 2B Macro | 2C Micro | 2D News (LLM)
   |
Stage 2E  Validate -> eligibility gate ◆ G3 -> combine eligible models
              -> conformal calibration -> alpha decay / horizon choice
   |
Stage 3   Risk model: covariance, VaR/ES, stress (sticker-driven scenarios)
   |
Stage 4   Optimiser per sleeve (cost-aware) + sleeve capital allocator  ◆ G4
   |
Stage 5   Leverage and position report: sizing, leverage, drawdown control
              ◆ G5 leverage/margin, then decision layer ◆ G6 NO-TRADE
   |
Stage 6   Execution + hard limits independent of the models
   |
Stage 7   Monitoring and learning -> reweight / retrain / revise screens  ◆ G7

Validation harness (purging, embargo, multiple-testing control) governs Stages 2–5.
Data integrity firewall ◆ G2 sits between Stage 0 and every consumer.
```

### 1.1 Sleeves

The pipeline runs as four **sleeves** that share one architecture but have separate configuration: universe filters, trading calendar, cost model, data depth and risk budget.

| Sleeve | Calendar and currency | Distinguishing features |
|---|---|---|
| ASX interday | ASX session, AUD | Closing-auction execution, franking, limited short borrow |
| US interday | US session (overnight in Sydney), USD | FX exposure, overnight gap risk while unattended |
| ASX intraday | ASX session with auctions, AUD | Tick size relative to price, thin depth outside large caps |
| US intraday | US session, USD | Deeper liquidity but more competition; costs dominate |

**Why sleeves.** Interday and intraday strategies differ in what information matters, how quickly signals decay and what they cost to trade. Treating them as one book would let one horizon's risk silently consume the other's capacity. Sleeves make each horizon separately measurable and allow the allocator (Section 11.4) to treat them as assets in their own right.

### 1.2 Relative importance of the four model layers by horizon

| Layer | Interday role | Intraday role |
|---|---|---|
| Financial | Core return, volatility and distress driver | Distress exclusion and volatility forecast only |
| Macro | Regime, factor loadings, transmission of shocks | Scheduled releases as volatility and direction events |
| Micro | Industry economics, peer-relative valuation | Market microstructure: order flow, spread, intensity |
| News | Multi-day drift after material news | Fast-decaying, latency-sensitive event signal |

---

## 2. Design Principles and Gates

> **Principle.** The pipeline is a chain of increasingly expensive decisions. Each stage may only use information that was knowable at decision time, and each stage has an explicit, measurable condition under which it stops.

### 2.1 Design principles

| # | Principle | Meaning in practice |
|---|---|---|
| P1 | **Point-in-time** | No datum is used at decision time $t$ unless its knowledge time $k \le t$. |
| P2 | **Filtered, not smoothed** | Latent states (regimes, time-varying betas) are estimated with filters that use data up to $t$ only. Smoothed estimates use the future and are for research plots only. |
| P3 | **One forecast language** | Every model emits a predictive distribution on a common quantile grid, so all models can be scored, combined and risk-checked identically. |
| P4 | **Validate, then admit, then combine** | A model receives combination weight only after passing out-of-sample validation and the eligibility gate. |
| P5 | **Most conservative credible risk binds** | Risk is estimated by several engines; the most conservative estimate that passes its own backtest is the one used. |
| P6 | **Default action is NO-TRADE** | A trade happens only when every rule set passes and expected benefit clears expected cost by a margin. |
| P7 | **Sizing is separate from leverage** | Relative position sizes come from the optimiser; total gross exposure is a separate decision with its own controls. |
| P8 | **Search is charged for** | The number of configurations tried is recorded and enters overfitting statistics (deflated Sharpe, PBO). |

### 2.2 Gates: the only places the pipeline may stop or exclude

| Gate | Location | Passes when | On failure |
|---|---|---|---|
| ◆ **G1** Universe | Stage 1 | Survival, liquidity, archetype and history tests all pass | Name excluded with reason code; history retained |
| ◆ **G2** Data integrity | Stage 0 → all | No look-ahead, schema, unit or freshness breach | Affected names quarantined for this run |
| ◆ **G3** Model eligibility | Stage 2E | Converged, stable, calibrated, skilful out of sample, positive net-of-cost value, low overfitting probability | Model excluded from combination this run |
| ◆ **G4** Risk limits | Stage 4 | Ex-ante volatility, ES, concentration, liquidity and stress limits hold | Re-optimise with tighter bounds, else scale toward cash |
| ◆ **G5** Leverage and margin | Stage 5 | Margin headroom under stress, drawdown rule, financing cost within budget | Reduce gross exposure until satisfied |
| ◆ **G6** NO-TRADE | Stage 5 | Data, model, risk, liquidity, market-state and kill-switch rule sets all pass | No trade for the name or the whole sleeve |
| ◆ **G7** Promotion | Stage 7 | Challenger beats champion over a shadow period with intact calibration | Champion retained |

---

## 3. Stage 0: Point-in-Time Data and the Company Knowledge Layer

> **Principle.** A backtest is only as honest as its timestamps. Every fact records *when it became knowable*, and every read asks "as of when?".

### 3.1 Why point-in-time matters

Most backtest inflation comes not from bad models but from information that would not have been available at the time: restated earnings, revised GDP, today's index members, or prices adjusted with future corporate actions. The pipeline therefore stores data **bitemporally**:

- **Valid time** $\tau$: the period a fact describes (e.g. FY2025 earnings).
- **Knowledge time** $k$: when the fact became public (e.g. the announcement timestamp).

A restatement never overwrites the original; it opens a new version with a later knowledge time. A query "as of $t$" returns the latest version with $k \le t$.

**Example.** FY2025 EPS is reported as 1.10 in August 2025 and restated to 1.04 in February 2026. A decision made on 31 December 2025 must see 1.10; one made on 15 March 2026 sees 1.04. A naive database would show 1.04 for both, leaking the restatement into the earlier decision.

### 3.2 Data inputs and knowledge-time rules

| Source | Content | Knowledge-time rule |
|---|---|---|
| Market data | Daily and intraday bars, trades and quotes, order book; **including delisted names** | Bar end or session close plus publication lag |
| Fundamentals | Statements, quarterly cash-flow reports, segment disclosures | Announcement timestamp; if missing, period end plus statutory deadline (conservative) |
| Estimates | Consensus, revisions, target prices, guidance | First time the value was observed |
| Macro | Activity, inflation, labour, rates, commodities, FX, credit, per economy | Official release time of each **vintage** (first-release values kept) |
| News | Wire stories, exchange announcements, transcripts | Publication time plus processing latency |
| Index membership | Constituents | Announcement date, not effective date |
| Borrow and margin | Short availability, borrow fee, margin rates | Observation time |

### 3.3 Corporate actions and tradability

Raw prices are never overwritten. Adjustment factors are stored and applied on read:

$$
F_t=\prod_{e:\ t<\mathrm{ex}_e\le T} a_e,\qquad \tilde P_t = P_t\,F_t ,
$$

where $a_e = m/n$ for an $n\!:\!m$ split or consolidation and $a_e=\mathrm{TERP}/P_{\mathrm{cum}}$ for a rights issue. Total return accumulates dividends at the ex-date: $\mathrm{TR}_t=\mathrm{TR}_{t-1}(P_t+D_t)/P_{t-1}$ .

Two subtleties matter:

1. **Level filters use unadjusted as-of prices.** A rule such as "price ≥ A\$0.10" must be applied to the price actually quoted at the time, not a back-adjusted one.
2. **Tradability is explicit.** A name that is halted, suspended or has zero volume at $t$ cannot be traded in the backtest.

### 3.4 Data-integrity firewall (◆ G2)

| Severity | Examples | Action |
|---|---|---|
| **Block** | Schema or type mismatch; any input with $k$ after the decision cut-off; timezone or unit error; duplicate keys | Affected branch does not run |
| **Quarantine** | Stale prices, incomplete history, $\lvert r_{i,t}\rvert > 8\hat\sigma_i$ with no corporate action or news, unreconciled split | Name removed from this run |
| **Warn** | Runs of identical closes, revision bursts | Reported, not excluded |

### 3.5 The Company Knowledge Layer

The knowledge layer is the shared description of every company. It is built once, versioned point-in-time, and read by the archetype builder, all four forecast models and the optimiser's exposure constraints. This keeps the layers consistent: the macro model, the stress tests and the sector limits all use the same view of what a company *is*.

| Component | Definition | Source | Used by |
|---|---|---|---|
| **Economy stickers** | Exposure vector $\mathbf e_i=(e_{i,1},\dots,e_{i,C})$, $e_{i,c}\ge0$, $\sum_c e_{i,c}=1$, giving the share of firm $i$'s economic activity tied to economy $c$ | Segment revenue and cost geography; estimated where disclosures are coarse | 2B macro, stress tests, country limits |
| **Market labels** | Industry, product market, listing venue, currency | Classification plus business descriptions | 1, 2C micro, sector limits |
| **Peer clusters** | Data-driven groups from return correlations and text embeddings of business descriptions | Estimated | 2C pooling, relative valuation |
| **Relationship graph** | Weighted directed graph $W$ of supplier, customer, competitor and shared-input links | Filings, transcripts, LLM extraction from text | 2D news propagation, 2B transmission, contagion stress |
| **Membership history** | Listing, delisting and identity changes | Exchange data | Survivorship-safe universes |

**Why stickers are versioned exposures, not tags.** A company listed on the ASX may earn most of its revenue in the US or China, and that mix changes with acquisitions and divestments. Storing $\mathbf e_i$ as a dated vector rather than a single label lets the macro model weight several economies at once and lets backtests use the exposure that was known at the time.

**Revenue versus cost stickers.** A firm can sell in one economy and produce in another (e.g. revenue in the US, costs in Australia). The layer stores two vectors, $\mathbf e^{\mathrm{rev}}_i$ and $\mathbf e^{\mathrm{cost}}_i$. Their difference measures natural FX and input-cost exposure, which a single sticker would hide.

---

## 4. Stage 1: Archetype Builder

> **Principle.** Selection is cheap, explainable and stable. It excludes names that are likely to fail or impossible to trade, and it chooses the strategy archetype for which each surviving name is suited. It does **not** forecast returns.

### 4.1 Why selection is a separate stage

Bankruptcy and liquidity risk are far cheaper to *exclude* than to model or hedge later. Separating selection from forecasting also prevents a common error: if the same model both selects stocks and forecasts their returns, the universe becomes tailored to that model's past successes, and out-of-sample results are biased upward.

### 4.2 Step 1: Hard survival filter (all sleeves)

The aim is to keep the universe "mostly fixed" while removing names with a material probability of distress.

**Merton distance to default.** Treat equity as a call option on firm assets $V$ with debt face value $D$:

$$
\mathrm{DD}_{i,t}=\frac{\ln(V_{i,t}/D_{i,t})+\left(\mu_i-\tfrac12\sigma^2_{V,i}\right)T}{\sigma_{V,i}\sqrt T},
\qquad \mathrm{PD}_{i,t}=\Phi(-\mathrm{DD}_{i,t}).
$$

$V$ and $\sigma_V$ are not observed; they are solved jointly from the observed equity value $E$ and equity volatility $\sigma_E$ using $E=V\Phi(d_1)-De^{-rT}\Phi(d_2)$ and $\sigma_E=\frac{V}{E}\Phi(d_1)\sigma_V$. **Interpretation:** DD is the number of asset-volatility standard deviations by which assets exceed debt; a DD of 1.5 implies roughly a 6.7% default probability over the horizon.

**Accounting and liquidity screens.**

| Metric | Definition | Default rule |
|---|---|---|
| Altman Z (non-financials) | $1.2X_1+1.4X_2+3.3X_3+0.6X_4+1.0X_5$ (working capital, retained earnings, EBIT, market equity / liabilities, sales, each scaled by assets) | Exclude $Z<1.8$ unless DD is high |
| Interest cover | $\mathrm{EBIT}/\mathrm{Interest}$ | $\ge 2$ |
| Leverage | Net debt / EBITDA | $\le 4$ |
| Refinancing wall | Debt due within 18 months / market capitalisation | Flag above 0.5 |
| Piotroski F-score | Sum of nine binary profitability, leverage and efficiency signals | $\ge 4$ |
| Cash runway (pre-revenue firms) | $\mathrm{Cash}_q/\max\{-(\mathrm{OCF}_q+\mathrm{ICF}_q),\varepsilon\}$ quarters | $\ge 4$ quarters |

Banks and insurers use capital and funding ratios instead, because their balance-sheet leverage is structural rather than a distress signal.

**Improvement over a pure threshold.** Rather than combining these rules mechanically, the pipeline fits a **discrete-time hazard model** (Section 6.4) that converts them into a single probability of distress within 12 months, $p^{\mathrm{dist}}_i$. The hard exclusion is $p^{\mathrm{dist}}_i > \bar p$ (e.g. 5%); the individual thresholds are kept as explanatory reason codes.

### 4.3 Step 2: Liquidity screen

| Metric | Definition | Interpretation |
|---|---|---|
| Median daily traded value | $\mathrm{MDTV}_i=\operatorname{med}_{s\in[t-59,t]}P_{i,s}V_{i,s}$ | Capacity; sets maximum position size |
| Amihud illiquidity | $\mathrm{ILLIQ}_i=\frac1{\lvert S\rvert}\sum_{s}\frac{\lvert r_{i,s}\rvert}{P_{i,s}V_{i,s}}$ | Price move per dollar traded |
| Roll spread | $\hat s_i=2\sqrt{\max\{0,-\operatorname{Cov}(\Delta p_s,\Delta p_{s-1})\}}$ | Effective spread estimated from daily prices |
| Zero-volume ratio | Share of sessions with no trades | Stale-price risk |
| Quoted spread (intraday) | Time-weighted $(\mathrm{ask}-\mathrm{bid})/\mathrm{mid}$ in bps | Direct cost of each round trip |

Intraday sleeves apply much tighter limits (e.g. spread below 10 bps and depth sufficient for the intended clip size) because costs are paid many times per day.

### 4.4 Step 3: Strategy archetypes

Each surviving name is tested for fit to one or more archetypes. A name can belong to several; the archetype tag determines which forecast features are emphasised and what turnover is expected.

| Archetype | Holding period | Selection criteria | Why these criteria |
|---|---|---|---|
| Momentum / trend | Weeks–months | Persistent 12–1 month returns, low idiosyncratic noise, adequate liquidity | Trend signals need clean price paths and must be cheap to rebalance |
| Quality-value | Months | Profitability, cash conversion, low accruals, valuation discount | Value without quality often signals distress ("value traps") |
| Mean reversion / pairs | Days–weeks | Cointegrated peer pairs, stable spread half-life, low borrow cost | Requires a stable economic link that the knowledge layer can confirm |
| Event-driven | Days | Regular information flow, analyst coverage, measurable announcement reactions | Needs events that the news and earnings models can price |
| Intraday liquid large-cap | Minutes–hours | Tight spread, deep book, high trade intensity, forecastable volatility | Only liquid names leave any edge after costs |

**Forecastability check.** A cheap test of whether volatility models will add value is the out-of-sample $R^2$ of a HAR model on range-based variance $v=\frac{(\ln H/L)^2}{4\ln2}$ (Parkinson), plus a Ljung–Box test on $\lvert r_t\rvert$. Names with no volatility clustering offer little for volatility-based sizing.

**Stability check.** A CUSUM test on recursive residuals of each name's market and sector beta regression. Names whose betas break frequently are poor candidates for factor-based strategies.

### 4.5 Step 4: Scoring, hysteresis and reconstitution

Metrics are standardised robustly within sector (median and MAD, winsorised at ±3), sign-aligned so higher is better, and combined:

$$
z_{ij}=\frac{x_{ij}-\operatorname{med}_{s(i)}(x_j)}{1.4826\,\operatorname{MAD}_{s(i)}(x_j)},\qquad
q_i=\sum_j\omega_j\,\Phi(z_{ij}),\quad \sum_j\omega_j=1 .
$$

The factor 1.4826 makes MAD consistent with the standard deviation under normality. $\Phi$ maps each score to $[0,1]$ so that no single extreme metric dominates.

**Hysteresis.** To keep the universe mostly fixed, entry and exit use different thresholds on the composite rank $\rho_i$:

$$
m_{i,t}=\bigl(m_{i,t-1}\wedge \rho_i\ge\theta_{\mathrm{out}}\bigr)\ \vee\ \bigl(\neg m_{i,t-1}\wedge\rho_i\ge\theta_{\mathrm{in}}\bigr),\qquad \theta_{\mathrm{in}}=0.55,\ \theta_{\mathrm{out}}=0.45 .
$$

A member is only removed when its rank falls clearly below the entry level. This matters because every forced exit is a trade with a cost.

**Forced exits** override hysteresis: distress probability above $\bar p$, delisting notice, suspension beyond five sessions, or an approved takeover.

**Reason codes.** Every exclusion carries a code (e.g. `LIQ` liquidity, `DIST` distress, `HIST` short history, `KEEP` retained by hysteresis), so the funnel is auditable and reversible.

### 4.6 Step 5: Data-depth tiers

High-frequency data is expensive in cost, storage and processing. It is allocated only after a name has passed G1.

| Tier | Typical size | Data | Purpose |
|---|---|---|---|
| T1 | Top 40 per market | Full trades and quotes | Intraday sleeves, realised measures, order-flow models |
| T2 | Next 120 | One-minute bars | Intraday volatility and execution modelling |
| T3 | Remainder | Daily data; tick data only around events | Interday sleeves |

Allocation is a budgeted choice: maximise $\sum_{i,k}x_{ik}V_{ik}$ subject to data-cost and storage budgets, where $V_{ik}$ rises with signal strength, event sensitivity, liquidity and whether an eligible model needs ticks. A greedy value-per-cost rule is close to optimal and easy to explain.

### 4.7 Worked example: three names through one weekly screen

| Check | Large-cap miner | Pre-revenue explorer | Mid-cap industrial |
|---|---|---|---|
| Liquidity (MDTV) | A\$48m ✓ | A\$0.9m ✓ | A\$2.1m ✓ |
| Distress probability (12m) | 0.4% ✓ | 9.5% ✗ (runway 2.5 quarters) | 1.8% ✓ |
| Composite score $q_i$ | 0.81 | — | 0.52 |
| Archetypes matched | Quality-value, momentum, intraday | — | Quality-value |
| Composite rank $\rho_i$ | 0.92 | — | 0.48 (member last week) |
| **Outcome** | **In**, Tier 1 | **Out** (`DIST`) | **In**, kept by hysteresis (`KEEP`), on watch |

The industrial would not be admitted fresh ($0.48<0.55$) but is not removed ($0.48\ge0.45$), avoiding a round-trip trade on a marginal change.

---

## 5. Stage 2: The Common Forecast Representation

> **Principle.** Every model family speaks the same language at its output: a calibrated predictive distribution on a common quantile grid.

### 5.1 Why distributions rather than point forecasts

A point forecast says nothing about confidence. The optimiser, the risk model and the leverage decision all need *uncertainty*. A model that forecasts +1% with a narrow interval should be sized very differently from one that forecasts +1% with a wide interval. Requiring a full distribution from every model also allows all four layers to be scored and combined on equal terms.

### 5.2 Specification

For each (security $i$, date $t$, horizon $h$), every model emits quantiles on the grid

$$
\mathcal T=\{0.01,0.025,0.05,0.10,0.25,0.50,0.75,0.90,0.95,0.975,0.99\}.
$$

If quantiles cross, they are sorted (monotone rearrangement). The forecast target for return models is the excess return over the benchmark, measured from the next session so that a decision at $t$ can actually be executed:

$$
y^{(h)}_{i,t}=\sum_{s=2}^{h+1}\left(r_{i,t+s}-r_{b,t+s}\right).
$$

The label ends at $t+h+1$. This end date is stored because validation must purge any training observation whose label overlaps the test period (Section 10.1).

### 5.3 Scoring

**Pinball loss** for quantile $\tau$ and its average over the grid, an approximation to the continuous ranked probability score (CRPS):

$$
\rho_\tau(u)=u\left(\tau-\mathbb 1\{u<0\}\right),\qquad
\widehat{\mathrm{CRPS}}=\frac{2}{\lvert\mathcal T\rvert}\sum_{\tau\in\mathcal T}\rho_\tau\left(y-\hat q_\tau\right).
$$

**Interpretation.** Pinball loss penalises an under-prediction of the 90% quantile nine times more than an over-prediction. Minimising it therefore produces a forecast that genuinely sits at the 90th percentile. CRPS rewards forecasts that are both **calibrated** (outcomes fall inside intervals at the stated rate) and **sharp** (intervals are narrow).

**Calibration.** The probability integral transform $u_t=\hat F_t(y_t)$ should be uniform if the forecast distribution is correct. Coverage of the 90% interval should be close to 90%.

---

## 6. Stage 2A: Financial Model

> **Principle.** "Highest tier" means the model with the best out-of-sample, net-of-cost performance for the task, not the most complex model. Every candidate must beat a simple benchmark.

### 6.1 Why complexity is not the criterion

Equity returns have a very low signal-to-noise ratio. Monthly cross-sectional predictive $R^2$ is typically well below 1%. In this regime, flexible models fit noise easily, and their in-sample advantage rarely survives out of sample. Regularised and tree-based models dominate in published horse races, while deep sequence models rarely justify their added estimation risk. The financial layer therefore separates three tasks, each with models suited to its statistical structure.

### 6.2 Baselines (every model must beat these)

| Baseline | Forecast | Role |
|---|---|---|
| Random walk / zero excess return | $\hat y=0$ | The minimum bar for return models |
| Historical mean | Rolling mean excess return | Tests whether the model adds anything over drift |
| Factor model | Exposure to market, size, value, momentum, quality | Tests whether the model adds anything over known factors |
| EWMA volatility | $\sigma^2_t=\lambda\sigma^2_{t-1}+(1-\lambda)r^2_{t-1}$ | Benchmark for volatility models |

### 6.3 Return and earnings forecasting

**Elastic net (cross-sectional ranking).**

$$
\min_\beta\ \frac{1}{2n}\lVert y-X\beta\rVert^2+\lambda\left(\rho\lVert\beta\rVert_1+\frac{1-\rho}{2}\lVert\beta\rVert_2^2\right).
$$

*Qualitative role.* The $L_1$ term sets weak predictors to zero (feature selection); the $L_2$ term spreads weight across correlated predictors, such as several valuation ratios, rather than picking one arbitrarily. It is transparent: each coefficient states how much a feature moves the forecast.

**Gradient boosting (LightGBM / XGBoost).** An additive ensemble of shallow trees, $F_M(x)=\sum_{m=1}^M\nu\,f_m(x)$, each fitted to the gradient of the loss of the previous ensemble. Use the pinball loss to produce quantiles directly.

*Qualitative role.* Captures non-linearities and interactions (e.g. momentum works differently for high- and low-quality firms) without the analyst specifying them. Controls: shallow trees, learning rate $\nu$ small, early stopping on an inner validation fold, and **monotone constraints** where economics dictate the sign (e.g. higher distress probability should never raise the forecast).

**Features.** Earnings revisions (breadth, magnitude), standardised unexpected earnings, accruals, gross profitability, margins and their trend, investment and asset growth, leverage and its change, valuation ratios, momentum and reversal, idiosyncratic volatility, and the distress probability from Section 6.4. All features are standardised cross-sectionally *within each training fold*, never using full-sample statistics.

**Quantile regression and quantile random forests** provide the distributional output for the linear and tree families respectively.

### 6.4 Distress model

**Discrete-time hazard model.**

$$
P\left(\text{default in }(t,t+1]\mid\text{survived to }t,\ x_{i,t}\right)=\Lambda\left(\alpha(t)+\beta^\top x_{i,t}\right),
$$

where $\Lambda$ is the logistic function and $x_{i,t}$ includes DD, profitability, leverage, cash runway, excess return and volatility.

*Qualitative role.* Unlike a one-period classifier, the hazard model uses every firm-year of history, including firms that survived, and handles changing covariates naturally. Market-based inputs (DD, recent returns) react quickly; accounting inputs add stability. The output feeds the Stage 1 exclusion, the optimiser (as a penalty) and the position report.

### 6.5 Volatility forecasting

Volatility is far more forecastable than returns, and it directly determines position sizes and leverage. It is therefore modelled carefully.

**GJR-GARCH with Student-$t$ innovations.**

$$
r_t=\sigma_t z_t,\quad z_t\sim t_\nu\ (\text{standardised}),\qquad
\sigma^2_t=\omega+\left(\alpha+\gamma\,\mathbb 1\{r_{t-1}<0\}\right)r^2_{t-1}+\beta\sigma^2_{t-1}.
$$

*Interpretation.* $\alpha$ is the reaction to yesterday's shock, $\beta$ the persistence, and $\gamma$ the **leverage effect**: negative returns raise future volatility more than positive returns of the same size. Stationarity requires $\alpha+\gamma/2+\beta<1$. Student-$t$ innovations capture fat tails that remain after volatility clustering is removed. The half-life of a volatility shock is $\ln 0.5/\ln(\alpha+\gamma/2+\beta)$.

**EGARCH** models the log of variance, so positivity holds without parameter constraints, and the asymmetry enters through the sign of $z_{t-1}$:

$$
\ln\sigma^2_t=\omega+\alpha\left(\lvert z_{t-1}\rvert-\mathbb E\lvert z\rvert\right)+\gamma z_{t-1}+\beta\ln\sigma^2_{t-1}.
$$

**HAR on realised volatility** (when intraday data exist).

$$
\mathrm{RV}_{t+1}=\beta_0+\beta_d\mathrm{RV}_t+\beta_w\overline{\mathrm{RV}}^{(5)}_t+\beta_m\overline{\mathrm{RV}}^{(22)}_t+\beta_J J_t+\varepsilon_{t+1}.
$$

*Interpretation.* Daily, weekly and monthly averages proxy for traders operating at different horizons. This simple linear model reproduces the long memory of volatility and typically beats GARCH at short horizons because realised variance measures volatility far more precisely than squared daily returns. $J_t$ is the jump component (Section 8.4); jumps are less persistent than continuous volatility, so separating them improves forecasts.

### 6.6 Time-varying exposures (state-space layer)

Factor loadings drift as firms change. A random-walk coefficient model estimated with the Kalman filter tracks them:

$$
y_t=x_t^\top\beta_t+\varepsilon_t,\qquad \beta_t=\beta_{t-1}+\eta_t,
$$

$$
K_t=P_{t\mid t-1}x_t\left(x_t^\top P_{t\mid t-1}x_t+\sigma^2_\varepsilon\right)^{-1},\qquad
\hat\beta_{t\mid t}=\hat\beta_{t\mid t-1}+K_t\left(y_t-x_t^\top\hat\beta_{t\mid t-1}\right).
$$

*Interpretation.* The Kalman gain $K_t$ balances how much to trust the newest observation against the existing estimate. Large state noise $\eta_t$ makes betas adapt quickly but noisily. Only the **filtered** estimate $\hat\beta_{t\mid t}$ is used (principle P2).

---

## 7. Stage 2B: Macro Model (Economy Stickers)

> **Principle.** A company is exposed to the economies it actually earns and spends in. Each economy has its own state; each company's macro forecast is the sticker-weighted combination of those states, passed through firm-specific sensitivities that are estimated from data but anchored to the sticker.

### 7.1 Qualitative overview

The macro layer answers: *given the state of the economies this company depends on, what distribution of returns should we expect?* It works in four steps:

1. Estimate a **state for each economy** (activity, inflation, rates, financial conditions) and its **regime**.
2. Combine economy states into a **company-specific macro exposure** using the stickers.
3. Estimate how strongly each company actually responds to that exposure (**sensitivities**), shrinking noisy estimates toward what the sticker implies.
4. Translate scheduled releases and surprises into **event effects**, and macro scenarios into **firm shocks** via a transmission network.

### 7.2 Economy states: dynamic factor models

For each economy $c$, a dynamic factor model summarises $N_c$ standardised series (activity, prices, labour, rates, credit, commodities):

$$
x_{c,t}=\Lambda_c f_{c,t}+e_{c,t},\qquad f_{c,t}=A_cf_{c,t-1}+u_{c,t},\quad u_{c,t}\sim N(0,Q_c).
$$

*Interpretation.* A handful of factors $f_{c,t}$ (typically growth, inflation and financial conditions) capture the common movement of dozens of noisy indicators. Estimation uses the Kalman filter, which handles the "ragged edge" naturally: a series not yet released for the current month simply drops out of that month's measurement equation.

### 7.3 Regimes: Markov switching

$$
y_{c,t}\mid s_t=k\sim N(\mu_k,\sigma^2_k),\qquad P(s_t=j\mid s_{t-1}=i)=p_{ij},\quad k\in\{\text{expansion},\ \text{slowdown},\ \text{stress}\}.
$$

The Hamilton filter returns $\pi_t(k)=P(s_t=k\mid\mathcal F_t)$, the filtered probability of each regime given information up to $t$.

*Why it matters.* Many relationships (factor premia, correlations, the usefulness of individual models) differ by regime. Regime probabilities are used to condition sensitivities, to weight models in the combination stage (Section 10.4), and to mix covariance matrices (Section 11.1). Using probabilities rather than a hard regime label avoids abrupt switches when the evidence is ambiguous.

### 7.4 Nowcasts: mixed-frequency data

Quarterly targets such as GDP are estimated before release from monthly indicators using MIDAS:

$$
y^{Q}_t=\beta_0+\beta_1\sum_{j=0}^{J}w(j;\theta)\,x^{M}_{t-j/3}+\varepsilon_t,\qquad
w(j;\theta)=\frac{e^{\theta_1j+\theta_2j^2}}{\sum_l e^{\theta_1l+\theta_2l^2}} .
$$

The exponential Almon weights let the data decide how quickly older monthly observations lose relevance, using only two parameters. A Bayesian VAR with a Minnesota prior is the companion model.

### 7.5 From economies to companies: the sticker-weighted exposure

For company $i$ with sticker vector $\mathbf e_i$, the company-specific macro factor is

$$
f^{\mathrm{mac}}_{i,t}=\sum_{c=1}^{C}e_{i,c}\,f_{c,t}.
$$

A miner with 60% Australian, 30% Chinese and 10% US activity thus responds to a blend of three economies' states rather than to Australian conditions alone. Revenue and cost stickers can be applied to different factors (e.g. revenue stickers to demand factors, cost stickers to input-price and wage factors).

### 7.6 Sensitivities: shrinkage toward the sticker prior

Each company's response to macro factors is estimated with a time-series regression

$$
r_{i,t}=a_i+\sum_j b_{ij}\,\Delta f_{j,t}+\varepsilon_{i,t}.
$$

These estimates are noisy for short or volatile histories. The pipeline therefore uses a **hierarchical (empirical Bayes) model**: the sticker gives a prior $b^{\mathrm{prior}}_{ij}$ (the exposure-weighted average sensitivity of firms concentrated in each economy and industry), and the posterior is a precision-weighted average:

$$
\hat b_{ij}^{\mathrm{post}}=\lambda_{ij}\,\hat b_{ij}^{\mathrm{OLS}}+(1-\lambda_{ij})\,b^{\mathrm{prior}}_{ij},\qquad
\lambda_{ij}=\frac{\tau_j^2}{\tau_j^2+\mathrm{se}(\hat b_{ij})^2}.
$$

$\tau_j^2$ is the cross-sectional variance of true sensitivities around the prior, estimated from the whole universe.

*Interpretation.* When the firm's own estimate is precise ($\mathrm{se}$ small), $\lambda$ approaches 1 and the data dominate. When it is imprecise, the estimate is pulled toward what the sticker implies. This is exactly the behaviour wanted: the sticker is a prior belief that data can overturn, not a fixed loading.

**Worked example.** A company's estimated sensitivity to Chinese activity is $\hat b^{\mathrm{OLS}}=1.40$ with standard error 0.50. Its sticker implies a prior of 0.60, and the cross-sectional spread is $\tau=0.30$.

$$
\lambda=\frac{0.30^2}{0.30^2+0.50^2}=\frac{0.09}{0.34}=0.265,\qquad
\hat b^{\mathrm{post}}=0.265(1.40)+0.735(0.60)=0.371+0.441=0.81.
$$

The noisy estimate of 1.40 is moderated to 0.81. If ten more years of data reduced the standard error to 0.15, then $\lambda=0.09/(0.09+0.0225)=0.80$ and the posterior would be $0.80(1.40)+0.20(0.60)=1.24$, close to the firm's own estimate.

### 7.7 Scheduled macro events

For each release (central bank decisions, CPI, labour force, GDP, PMIs) the surprise is standardised:

$$
S_{c,t}=\frac{\text{actual}_{c,t}-\text{consensus}_{c,t}}{\mathrm{sd}(\text{actual}-\text{consensus})}.
$$

Interday, a company's expected response is $\hat b_{i,S}\,S_{c,t}$, scaled by its sticker weight on economy $c$. Intraday, the release time is treated as a known volatility event: forecast variance is raised around the release and intraday sleeves can reduce exposure beforehand.

### 7.8 Transmission network and macro stress

To translate a scenario (e.g. a +50 bp rate surprise) into firm-level shocks, sector responses at horizon $h$ are estimated with **local projections**:

$$
y_{t+h}-y_{t-1}=\alpha_h+\beta_h\,\varepsilon_t+\sum_l\gamma_{h,l}^\top w_{t-l}+u_{t+h},
$$

one regression per horizon, which is more robust to misspecification than iterating a VAR. Firm shocks then combine sector responses $\Gamma_s(h)$ with the firm's sector and sticker exposures $B_{is}$:

$$
\Delta r_i=\sum_sB_{is}\,\Gamma_s(h)\,\varepsilon .
$$

**Worked example (+50 bp policy-rate surprise, $h=5$).**

| Step | Calculation | Result |
|---|---|---|
| Sector response per +100 bp | Local projections | Real estate −4.2%, Financials −0.9%, Materials −1.1% |
| Scale to +50 bp | Multiply by 0.5 | −2.10%, −0.45%, −0.55% |
| Firm shocks | REIT $B=1.10$; bank $B=0.95$; miner $B=1.20$ | −2.31%, −0.43%, −0.66% |
| Portfolio (weights 10%, 15%, 20%, rest 55% at −0.90%) | $\sum_i w_i\Delta r_i$ | $0.10(-2.31)+0.15(-0.43)+0.20(-0.66)+0.55(-0.90)=-0.92\%$ |

### 7.9 FX channel

AUD/USD enters as an explicit factor. The firm's net FX exposure is approximately $e^{\mathrm{rev}}_{i,\mathrm{USD}}-e^{\mathrm{cost}}_{i,\mathrm{USD}}$: an Australian-listed exporter with USD revenue and AUD costs benefits from a weaker AUD. At portfolio level, US-sleeve holdings add a direct translation exposure, which is either hedged or budgeted explicitly (Section 11.4).

---

## 8. Stage 2C: Micro Model (Market Labels)

> **Principle.** Company-level microeconomic data are sparse and slow. Labelling companies by market lets the model borrow strength across peers who face the same demand, costs and competition.

### 8.1 Qualitative overview

The micro layer models the economics of the markets a company operates in: pricing power, input costs, demand, competitive intensity and margins. Interday, these drive earnings and therefore returns. Intraday, "micro" means **market microstructure**: the mechanics of how prices form from order flow.

### 8.2 Interday: panel and hierarchical models

**Two-way fixed-effects panel.**

$$
r_{it}=\alpha_i+\lambda_t+\beta^\top x_{it}+\varepsilon_{it},
$$

with standard errors clustered by firm and date. $\alpha_i$ absorbs permanent firm differences; $\lambda_t$ absorbs market-wide shocks on each date. The coefficients $\beta$ then measure how *changes* in micro variables (input-cost growth, price growth, market share) relate to *relative* returns.

**Hierarchical model by market label.** For market group $g$,

$$
\theta_i\sim N(\theta_g,\tau_g^2),\qquad \hat\theta_i=\lambda_i\hat\theta_i^{\mathrm{OLS}}+(1-\lambda_i)\bar\theta_g,\quad \lambda_i=\frac{\tau_g^2}{\tau_g^2+\mathrm{se}_i^2}.
$$

*Interpretation.* The structure mirrors the macro shrinkage: a firm with little data inherits the behaviour of its market peers; a firm with a long, clean history keeps its own estimate. Peer groups come from both market labels and data-driven clusters, because standard industry classifications often misplace diversified firms.

**Market variables modelled by group.** Output prices (e.g. commodity prices, producer price indices), input costs (energy, wages, freight), demand indicators (retail sales, housing activity, industrial production in the relevant economies via the stickers), and competitive structure (Herfindahl concentration, peer margin dispersion).

**Double machine learning for sensitivities.** When the question is how strongly a firm responds to a specific variable $D$ (e.g. an input-cost shock) in the presence of many controls $X$, the partially linear model with cross-fitting removes the bias that would arise from flexible control models:

$$
\tilde Y=Y-\hat\ell(X),\qquad \tilde D=D-\hat m(X),\qquad \hat\theta=\frac{\sum_i\tilde D_i\tilde Y_i}{\sum_i\tilde D_i^2}.
$$

The nuisance functions $\hat\ell$ and $\hat m$ are fitted on separate folds. This is reserved for the investable universe, because it is expensive and needs a credible identification argument per question.

**Earnings response.** $\mathrm{CAR}_i[-1,+1]=a+b\,\mathrm{SUE}_i+\gamma^\top z_i+u_i$, with $\mathrm{SUE}_i=(\mathrm{EPS}_i-\mathbb E[\mathrm{EPS}_i])/\hat\sigma_i$. The slope $b$, estimated by market group, tells the event-driven archetype how much an earnings surprise typically moves the price.

### 8.3 Intraday: microstructure models

| Model | Specification | What it captures |
|---|---|---|
| Hawkes process (trade arrivals) | $\lambda(t)=\mu+\alpha\sum_{t_i<t}e^{-\beta(t-t_i)}$, stationary if $\alpha/\beta<1$ | Self-excitation: trades cluster because trades trigger more trades. $\alpha/\beta$ is the share of activity that is endogenous. |
| ACD (durations) | $x_i=\psi_i\epsilon_i$, $\psi_i=\omega+\alpha x_{i-1}+\beta\psi_{i-1}$ | Time between trades, the analogue of GARCH for trading intensity |
| Order-flow imbalance | $\mathrm{OFI}_t=\sum(\Delta\text{bid depth}-\Delta\text{ask depth})$ over the interval | Short-horizon price pressure from the order book |
| Kyle's lambda | $\Delta p_j=\lambda\,\mathrm{sign}(q_j)\sqrt{\lvert q_j\rvert}+\varepsilon_j$ | Price impact per unit of signed volume; feeds the cost model |
| Intraday volume profile | Time-of-day seasonal plus residual | Expected liquidity for execution scheduling |

*Qualitative role.* These models forecast when liquidity will be available, how far prices move per unit traded, and whether order flow is temporarily pushing prices away from fair value. For intraday sleeves the edge is usually small relative to costs, so the cost forecasts from this layer are as important as the direction forecasts.

### 8.4 Realised measures from intraday data

After cleaning trades (removing prints outside the session, merging same-timestamp trades, and dropping prices far from a rolling median), returns are sampled on a regular grid (e.g. 5 minutes):

| Measure | Formula | Interpretation |
|---|---|---|
| Realised variance | $\mathrm{RV}=\sum_j r_j^2$ | Total variation, including jumps |
| Bipower variation | $\mathrm{BV}=\frac\pi2\sum_j\lvert r_j\rvert\lvert r_{j-1}\rvert$ | Robust to jumps; estimates continuous variation |
| Jump share | $\max\{0,1-\mathrm{BV}/\mathrm{RV}\}$ | Fraction of variation due to jumps |
| Realised kernel | $\gamma_0+2\sum_{h=1}^{H}k\!\left(\frac{h}{H+1}\right)\gamma_h$ | Corrects for microstructure noise (bid–ask bounce) |
| Semivariances | $\mathrm{RS}^{\pm}=\sum_j r_j^2\,\mathbb 1\{r_j\gtrless0\}$ | Downside versus upside variation |

These feed the HAR model (Section 6.5), the archetype forecastability test and the jump component $J_t$.

---

## 9. Stage 2D: News Model (LLM Extraction)

> **Principle.** The large language model is a **feature extractor**, not a forecaster. It reads text and produces structured, timestamped facts; a separate model trained on realised returns decides what those facts are worth.

### 9.1 Why this division of labour

LLMs are excellent at reading context (who is affected, through which channel, whether the news is genuinely new) but poorly calibrated as direct return forecasters, and their outputs on historical text are contaminated by knowledge of what happened next. Splitting extraction from forecasting addresses both problems: extraction quality can be checked against labelled examples, and calibration comes from a downstream model fitted to actual price reactions.

### 9.2 Processing steps

| Step | Method | Output |
|---|---|---|
| 1. Ingest | Timestamp, source credibility, entity linking to the knowledge layer | Clean story records |
| 2. De-duplicate | Near-duplicates (embedding cosine similarity above 0.9) collapse to one event | One record per event |
| 3. Structured extraction | LLM with a fixed schema (below) at deterministic settings | Event fields |
| 4. Novelty | $\nu_d=1-\max_{d'\in\mathcal W_{72h}}\cos(e_d,e_{d'})$ | How new the information is |
| 5. Priced-in check | Compare implied direction with the return since publication | Residual surprise |
| 6. Graph propagation | Spread effects along the relationship graph (Section 9.4) | Effects on related firms |
| 7. Downstream model | Calibrated model on realised returns (Section 9.5) | Forecast distribution |

### 9.3 Extraction schema

| Field | Example values |
|---|---|
| Event type | Earnings, guidance, M&A, regulation, litigation, supply disruption, management change, capital raise |
| Primary entities | Linked company identifiers |
| Direction | Positive, negative, ambiguous |
| Magnitude | Ordinal (1–5) or quantified (e.g. "guidance cut 8%") |
| Transmission channel | Demand, costs, supply, regulation, financing, competition |
| Affected parties | Suppliers, customers, competitors named or implied |
| Horizon | Immediate, quarters, long-term |
| Confidence | Model-reported, later recalibrated |

Text is treated as **untrusted data**: instructions embedded in an article are never followed, and only the structured fields leave this step.

### 9.4 Propagation along the relationship graph

Let $s^{(0)}$ be the vector of direct effects on the named firms and $W$ the row-normalised relationship graph (weight $W_{jk}$ = importance of firm $k$ to firm $j$, e.g. share of revenue from a customer). With a per-hop decay $\delta$,

$$
s=\sum_{n=0}^{\infty}(\delta W)^n s^{(0)}=(I-\delta W)^{-1}s^{(0)},\qquad \delta\,\rho(W)<1,
$$

and each effect decays over time as $e^{-t/\kappa}$.

*Interpretation.* A shock to one firm reaches its direct partners with weight $\delta W$, their partners with $\delta^2W^2$, and so on. The condition on the spectral radius $\rho(W)$ guarantees the effects fade with distance. This is where the LLM adds the most value, because it can extract relationships from filings and articles that no structured database contains.

**Worked example.** A key customer of a supplier announces a production cut implying a −2.0% direct effect on the customer. The supplier earns 40% of revenue from it ($W=0.4$) and $\delta=0.5$. The first-hop effect on the supplier is $-2.0\%\times0.4\times0.5=-0.40\%$, before calibration by the downstream model. A competitor linked with a positive substitution weight would receive a positive effect instead.

### 9.5 Downstream calibrated model

The shock probability is modelled with a logistic model on the extracted fields:

$$
P\left(\lvert\mathrm{AR}_{i,t}\rvert>2\hat\sigma_i\mid z_d\right)=\Lambda(\theta^\top z_d),
$$

where $z_d$ includes novelty, direction, magnitude, channel, source credibility, the residual surprise and the propagated effect. A quantile model on the same features produces the return distribution. Calibration is checked with the Brier score and reliability curves.

### 9.6 Horizon behaviour

| Sleeve | Treatment |
|---|---|
| Intraday | Most news in liquid names is priced within seconds to minutes. The intraday sleeve uses news mainly to *reduce* exposure and widen volatility forecasts, unless measured latency allows a directional edge. |
| Interday | Exploits documented slow diffusion: post-announcement drift, under-reaction to complex or second-order (propagated) news, and news about less-covered firms. |

### 9.7 Leakage control (critical)

An LLM trained after the backtest period has, in effect, read the future. Controls:

1. Evaluate only on news published after the model's training cut-off, or use a model whose cut-off precedes the test window.
2. Freeze the model version and prompt during any evaluation period.
3. Use strict publication timestamps; the model sees only text available at that moment.
4. Treat forward (live or paper) performance as the definitive validation for this layer.

---

## 10. Stage 2E: Validation, Eligibility and Signal Combination

> **Principle.** Validate first, then decide eligibility, then combine. A model that has not earned eligibility never receives a weight.

### 10.1 Purged, embargoed walk-forward validation

Financial observations overlap in time. A five-day return label at $t$ shares four days with the label at $t+1$. Standard cross-validation leaks this overlap and overstates skill. The pipeline uses:

- **Walk-forward folds:** train on the past, test on the following block, then roll forward.
- **Purging:** remove training observations whose label interval $[t,\ \text{label end}_t]$ overlaps the test window.
- **Embargo:** discard a further $e\ge h$ sessions after each test window to block serial-correlation leakage.
- **Nested tuning:** hyperparameters are chosen inside each training block on an inner purged split. The outer test blocks are used once.

### 10.2 Statistical tests and what each answers

| Test | Question answered |
|---|---|
| Diebold–Mariano (small-sample corrected) | Is model A more accurate than model B? |
| Giacomini–White | Is the advantage conditionally predictable (e.g. only in some regimes)? |
| Hansen's Superior Predictive Ability | Does *any* model beat the benchmark after accounting for the search over many models? |
| Model Confidence Set | Which models are statistically tied for best? |
| Deflated Sharpe ratio | Is a backtested Sharpe ratio significant given how many configurations were tried? |
| Probability of backtest overfitting (CSCV) | How often does the in-sample best rank below median out of sample? |
| Romano–Wolf step-down | Which individual signals survive family-wise error control? |

**Deflated Sharpe ratio.**

$$
\mathrm{DSR}=\Phi\!\left(\frac{(\widehat{SR}-SR_0)\sqrt{n-1}}{\sqrt{1-\hat\gamma_3\widehat{SR}+\frac{\hat\gamma_4-1}{4}\widehat{SR}^2}}\right),
\quad
SR_0=\sqrt{\mathbb V[\widehat{SR}_k]}\left((1-\gamma_E)\Phi^{-1}\!\left(1-\tfrac1N\right)+\gamma_E\Phi^{-1}\!\left(1-\tfrac1{Ne}\right)\right).
$$

$SR_0$ is the Sharpe ratio expected from the *best of $N$ worthless strategies*; $\gamma_E\approx0.5772$; $\hat\gamma_3,\hat\gamma_4$ are skewness and kurtosis of returns.

**Worked example.** A strategy shows an annualised Sharpe of 1.20 (0.0756 daily) over 1,260 sessions, with skew −0.4 and kurtosis 6. Forty configurations were tried, with trial Sharpe dispersion 0.50 annualised, giving $SR_0=0.0690$ daily (1.10 annualised).

$$
\mathrm{DSR}=\Phi\!\left(\frac{(0.0756-0.0690)\sqrt{1259}}{\sqrt{1+0.4(0.0756)+1.25(0.0756)^2}}\right)=\Phi\!\left(\frac{0.234}{1.019}\right)=\Phi(0.23)=0.59 .
$$

A naive test against zero gives 0.996. After deflation the evidence is barely better than a coin flip, so the signal fails eligibility. This is why the trial count $N$ must be recorded automatically rather than reported from memory.

### 10.3 Eligibility gate (◆ G3)

| Criterion | Default rule | Why |
|---|---|---|
| Convergence | Optimiser succeeded; Hessian positive definite; parameters not at bounds | Unconverged models produce arbitrary forecasts |
| Stability | Parameter stability tests across folds; key coefficient signs stable | Unstable models are fitting noise |
| Calibration | 90% interval coverage within ±3 percentage points | Mis-sized uncertainty mis-sizes positions |
| Out-of-sample skill | In the 90% model confidence set, or DM $p<0.10$ against the baseline | Must add accuracy |
| Economic value | Net-of-cost information coefficient $>0$ | Accuracy that cannot be traded is worthless |
| Overfitting | PBO $<0.2$; DSR $>0.95$ for trading signals | Protects against data snooping |

### 10.4 Combination of eligible models

**Inverse-loss weights** with exponential discounting:

$$
w_{k,t}\propto\left(\sum_{s<t}\delta^{t-s}L_{k,s}\right)^{-1},\qquad \delta=0.97 .
$$

**Stacking:** choose $w\ge0$, $\mathbf1^\top w=1$ to minimise the loss of the combined out-of-fold forecasts.

**Regime-conditional weights:** $w_{k,t}=\sum_s\pi_t(s)\,w^{(s)}_k$, so that models which work in stress receive more weight when stress is likely.

**Distribution combination (quantile averaging):** $\hat q_\tau=\sum_kw_k\hat q_{k,\tau}$.

**Shrinkage toward equal weights** when evidence is thin: $w\leftarrow\lambda w+(1-\lambda)\mathbf1/K$. Equal weights are notoriously hard to beat when weights are estimated with error.

**Worked example (5-day excess return, realised $y=-1.80\%$).**

| Model | $\hat q_{0.05}$ | $\hat q_{0.50}$ | $\hat q_{0.95}$ | Trailing CRPS | Weight |
|---|---:|---:|---:|---:|---:|
| A: HAR + GJR, Student-$t$ | −3.45% | 0.30% | 4.05% | 0.0071 | 0.315 |
| B: Gradient-boosting quantiles | −4.83% | 0.10% | 5.03% | 0.0069 | 0.324 |
| C: Quantile random forest | −3.49% | −0.20% | 3.09% | 0.0062 | 0.361 |
| **Combination** | **−3.91%** | **0.05%** | **4.02%** | — | 1.000 |

Weights: $1/0.0071=140.8$, $1/0.0069=144.9$, $1/0.0062=161.3$; total 447.1, giving 0.315, 0.324 and 0.361. Combined 5% quantile: $0.315(-3.45)+0.324(-4.83)+0.361(-3.49)=-3.91\%$. The combination stays close to the best model while keeping a wider, safer interval: it hedges model risk rather than chasing the latest winner.

### 10.5 Conformal calibration

Combined intervals are recalibrated with **conformalised quantile regression** on a held-out fold:

$$
E_j=\max\left\{\hat q_{\mathrm{lo}}(x_j)-y_j,\ y_j-\hat q_{\mathrm{hi}}(x_j)\right\},\qquad
C(x)=\left[\hat q_{\mathrm{lo}}(x)-\hat Q,\ \hat q_{\mathrm{hi}}(x)+\hat Q\right],
$$

where $\hat Q$ is the $\lceil(1-\alpha)(n+1)\rceil$-th smallest $E_j$. This guarantees coverage under exchangeability. Because markets drift, **adaptive conformal** updates the target level online: $\alpha_{t+1}=\alpha_t+\gamma(\alpha-\mathrm{err}_t)$, widening intervals after misses and narrowing them after a run of hits.

### 10.6 Alpha decay and horizon choice

$$
\mathrm{IC}(h)=\operatorname{corr}_{\mathrm{rank}}\left(s_{i,t},y^{(h)}_{i,t}\right)\approx\mathrm{IC}_0\,e^{-h/\kappa},\qquad \text{half-life}=\kappa\ln2 .
$$

The trading horizon for each signal and sleeve maximises the net information ratio,

$$
\mathrm{IR}\approx\mathrm{TC}\cdot\mathrm{IC}\cdot\sqrt{\mathrm{BR}}-\text{cost drag},
$$

where BR is breadth (independent bets per year) and TC the transfer coefficient (how much of the signal survives portfolio constraints). Fast-decaying signals belong in intraday sleeves only if their costs are low enough; slow signals belong in interday sleeves. A signal stays selected only if $\overline{\mathrm{IC}}/\mathrm{sd}(\mathrm{IC})\ge0.3$.

---

## 11. Stages 3–4: Risk Model, Optimiser and Sleeve Allocator

> **Principle.** Risk is estimated several ways and the most conservative credible estimate binds. The optimiser never sees a number the risk engine has not checked.

### 11.1 Covariance estimation

| Estimator | Specification | Qualitative role |
|---|---|---|
| Ledoit–Wolf shrinkage | $\hat\Sigma=\hat\delta F+(1-\hat\delta)S$ | With hundreds of stocks the sample covariance $S$ is badly conditioned; shrinking toward a structured target $F$ with data-chosen intensity $\hat\delta$ gives a stable, invertible matrix |
| Factor model | $\Sigma=B\Omega B^\top+D$ | Explains co-movement through a few factors (market, sector, style, macro via stickers, FX), plus diagonal idiosyncratic risk; interpretable exposures |
| DCC-GARCH | $Q_t=(1-a-b)\bar Q+a\,z_{t-1}z_{t-1}^\top+bQ_{t-1}$, $R_t=\mathrm{diag}(Q_t)^{-1/2}Q_t\,\mathrm{diag}(Q_t)^{-1/2}$ | Correlations that rise in sell-offs, when diversification is most needed and least available |
| Regime mixture | $\Sigma_t=\sum_k\pi_t(k)\left[\Sigma^{(k)}+(\mu^{(k)}-\bar\mu_t)(\mu^{(k)}-\bar\mu_t)^\top\right]$ | Blends calm and stress covariances by regime probability; the second term adds uncertainty about which regime holds |

**Cross-market treatment.** ASX and US sessions do not overlap, so same-day returns understate correlation (the US reaction to Australian news appears the next day). The pipeline estimates cross-market covariance on overlapping multi-day returns or adds lead–lag terms.

### 11.2 Tail risk: VaR, ES and their backtests

$$
\mathrm{VaR}_\alpha=-\inf\{x:P(R_p\le x)>1-\alpha\},\qquad \mathrm{ES}_\alpha=-\mathbb E[R_p\mid R_p\le-\mathrm{VaR}_\alpha].
$$

Three engines, the most conservative credible one binding:

1. **Filtered historical simulation:** historical standardised residuals rescaled by today's GARCH volatility, $\tilde r_s=\hat\sigma_{t+1}\hat z_s$.
2. **Monte Carlo:** GJR-$t$ paths with Student-$t$ copula shocks to capture joint tail dependence.
3. **Extreme value theory:** a generalised Pareto fit to losses beyond a threshold $u$,

$$
\mathrm{VaR}_p=u+\frac{\hat\sigma}{\hat\xi}\left[\left(\frac{n}{N_u}(1-p)\right)^{-\hat\xi}-1\right],\qquad
\mathrm{ES}_p=\frac{\mathrm{VaR}_p+\hat\sigma-\hat\xi u}{1-\hat\xi}.
$$

ES contributions are allocated with the Euler rule $\mathrm{ES}_i=w_i\,\partial\mathrm{ES}/\partial w_i$, which sums to total ES.

**Backtests.** Kupiec's unconditional-coverage test (are exceptions at the right rate?), Christoffersen's independence test (do exceptions cluster?) and the Acerbi–Székely test for ES. An engine that fails is demoted.

**Worked example (99% one-day VaR, 250 sessions, 2.5 expected exceptions).** With 9 exceptions, $\hat p=0.036$ and

$$
\mathrm{LR}_{\mathrm{uc}}=-2\left[241\ln0.99+9\ln0.01-241\ln0.964-9\ln0.036\right]=10.23,\quad p=0.0014,
$$

so the VaR is understated and the engine is replaced. With 2 exceptions, $\mathrm{LR}_{\mathrm{uc}}=0.11$ ($p=0.74$): consistent with correct coverage.

### 11.3 Stress library

| Scenario | Construction |
|---|---|
| Global financial crisis, COVID crash, 2022 rate shock | Historical replay of factor returns |
| Policy-rate surprise (each economy) | Transmission network (Section 7.8) |
| China activity shock | Applied through China stickers: commodity prices, AUD, credit |
| US recession | Applied through US stickers; plus overnight gap for US sleeves |
| AUD −10% | FX channel using revenue and cost stickers |
| Liquidity drought | ADV × 0.3 and spreads × 3 in exit costs |
| Contagion | Default of a large firm propagated through the relationship graph |

Each scenario has a loss budget. The sticker-based scenarios are an improvement over generic country shocks, because they hit each company in proportion to its actual economic exposure.

### 11.4 Portfolio optimiser (per sleeve)

**Return inputs: Black–Litterman.**

$$
\mu_{\mathrm{BL}}=\left[(\tau\Sigma)^{-1}+P^\top\Omega^{-1}P\right]^{-1}\left[(\tau\Sigma)^{-1}\pi+P^\top\Omega^{-1}q\right].
$$

*Interpretation.* The equilibrium prior $\pi$ (returns implied by market weights) is updated by the combined forecasts $(P,q)$, with confidence $\Omega$ taken from their predictive variances. Confident forecasts move the portfolio a lot; uncertain ones barely move it. This directly addresses the main weakness of mean–variance optimisation: its extreme sensitivity to errors in expected returns.

**Objective.**

$$
\max_w\ \mu_{\mathrm{BL}}^\top w-\frac{\gamma}{2}w^\top\hat\Sigma w-\sum_i\left(c_i\lvert\Delta w_i\rvert+\eta_i\lvert\Delta w_i\rvert^{3/2}\right)-\kappa\sum_ip^{\mathrm{dist}}_i\lvert w_i\rvert .
$$

The linear cost term covers spread and commission; the $3/2$-power term is market impact (the square-root law integrated over the trade); the last term penalises exposure to names with elevated distress probability.

**Constraints.**

- Position caps $\lvert w_i\rvert\le\min\{w_{\max},\ \phi\,\mathrm{ADV}_i\,d/\mathrm{NAV}\}$, so every position can be exited in $d$ days at participation $\phi$.
- Sector, country (via stickers), factor and FX exposure limits.
- Turnover limit; long-only or long-short rules; borrow availability for shorts.
- CVaR limit over scenarios (Rockafellar–Uryasev): $\zeta+\frac1{(1-\beta)S}\sum_su_s\le\overline{\mathrm{CVaR}}$, $u_s\ge-r_s^\top w-\zeta$, $u_s\ge0$.

All terms are convex, so the problem is solved to global optimality.

**Alternatives** used as challengers: hierarchical risk parity (robust when the covariance is unreliable) and minimum-CVaR portfolios.

**Infeasibility protocol.** Relax in order turnover → sector → tracking error. Never relax liquidity or tail limits. If still infeasible, hold current weights and flag.

**Risk-limit gate (◆ G4).** Ex-ante volatility ≤ 1.1 × target; ES within limit (max of the engines); single name ≤ 8%, sector ≤ 30%; liquidation ≤ 3 days at 20% ADV; every stress within budget.

### 11.5 Sleeve capital allocator

Each sleeve is treated as an asset with its own expected return, volatility and correlation with other sleeves. Capital is divided by **risk budgeting**:

$$
\mathrm{RC}_k=\frac{w_k(\Sigma_{\mathrm{sleeve}}w)_k}{\sigma_p}=b_k\,\sigma_p,\qquad \sum_kb_k=1 .
$$

Budgets $b_k$ start equal and move toward sleeves with stronger out-of-sample evidence, with bounds. The intraday sleeves have a hard cap so their losses cannot consume interday risk capacity. FX exposure from US sleeves is either hedged or given its own explicit budget.

---

## 12. Stage 5: Leverage and Position Report

> **Principle.** Relative sizes come from the optimiser; total leverage is a separate decision governed by forecast uncertainty, volatility, drawdown and margin. The report states what to hold, how much, why, and when the position is wrong.

### 12.1 Why sizing and leverage are separated

The optimiser answers "in what proportions?". Leverage answers "how much in total?". Mixing them lets an over-confident return forecast inflate both at once. Separating them lets leverage respond to things the optimiser does not see: realised drawdown, margin conditions and financing costs.

### 12.2 Kelly sizing under estimation uncertainty

For a portfolio with expected excess return $\mu$ and variance $\sigma^2$, the growth-optimal (Kelly) leverage is

$$
f^\ast=\frac{\mu}{\sigma^2}.
$$

*Why only a fraction.* Full Kelly maximises long-run growth only if $\mu$ and $\sigma$ are known exactly. With estimation error, over-betting is far more damaging than under-betting: growth at twice Kelly is zero, while half Kelly retains about 75% of the maximum growth with half the volatility. The pipeline therefore uses

$$
f=c\cdot\frac{\mu_{\mathrm{BL}}}{\sigma^2+\mathrm{Var}(\hat\mu)},\qquad c\in[0.25,0.5],
$$

using the shrunk return from Black–Litterman and adding estimation variance to the denominator.

**Worked example.** Raw forecast $\mu=8\%$, $\sigma=20\%$: full Kelly is $0.08/0.04=2.0\times$. After shrinkage $\mu_{\mathrm{BL}}=5\%$, with $\mathrm{se}(\hat\mu)=4\%$:

$$
f^{\ast}=\frac{0.05}{0.04+0.0016}=1.20,\qquad f_{c=0.25}=0.30,\qquad f_{c=0.5}=0.60 .
$$

The headline figure of 2.0× falls to 0.3–0.6× once uncertainty is respected. Kelly here acts as a **cap**, not a target.

### 12.3 Leverage rule

$$
L_t=\min\left\{\underbrace{\frac{\sigma^\ast}{\hat\sigma_{p,t}}}_{\text{vol target}},\ \underbrace{f_t}_{\text{fractional Kelly}},\ \underbrace{L_{\max}}_{\text{policy}},\ \underbrace{L^{\mathrm{margin}}_t}_{\text{margin headroom}}\right\}\times\underbrace{\max\left\{0,1-\frac{\mathrm{DD}_t}{\mathrm{DD}_{\max}}\right\}}_{\text{drawdown control}} .
$$

| Component | Purpose |
|---|---|
| Volatility target $\sigma^\ast/\hat\sigma_p$ | Keeps risk stable: leverage falls automatically when forecast volatility rises |
| Fractional Kelly | Caps leverage by the strength of the evidence |
| Policy maximum | Hard ceiling regardless of models |
| Margin headroom | Largest leverage at which the worst stress scenario leaves margin above the call level by a safety buffer |
| Drawdown control | Reduces leverage linearly as the drawdown from peak approaches the maximum tolerable drawdown; at $\mathrm{DD}_{\max}$ exposure is zero |

**Worked example.** Target volatility 10%, forecast portfolio volatility 8%: vol target gives 1.25×. Fractional Kelly 0.60× would bind for a single strategy, but for a diversified multi-sleeve book, suppose the Kelly cap is 1.8× and policy maximum is 1.5×; then $\min\{1.25,1.8,1.5,1.6\}=1.25\times$. With a current drawdown of 6% and $\mathrm{DD}_{\max}=20\%$, the multiplier is $1-6/20=0.70$, so $L=0.875\times$.

**Intraday versus overnight.** Intraday leverage may be higher because exposure is closed before the session ends and stop-loss rules can act. Overnight leverage must survive gap risk without any intervention, so it is computed with gap-inclusive stress scenarios. US positions held during Sydney night hours are the main case.

**Financing cost.** A leveraged position must earn more than its financing and borrow costs. The net expected return used throughout is $\mu-(L-1)r_{\mathrm{fin}}$ for longs and $\mu-r_{\mathrm{borrow}}$ for shorts.

**Leverage and margin gate (◆ G5).** Passes when margin headroom under the worst stress exceeds the buffer, the drawdown rule is satisfied, and financing cost is within budget. On failure, gross exposure is reduced until all three hold.

### 12.4 Decision layer: from target weights to actions

Each name moves through states: FLAT → LONG/SHORT → HOLD / INCREASE / REDUCE → FLAT. A change is only made if its expected benefit over the signal horizon exceeds its expected cost by a margin $m$:

$$
\lvert\hat\alpha_i\rvert\,h\,\lvert\Delta w_i\rvert>(1+m)\left(c_i\lvert\Delta w_i\rvert+\eta_i\lvert\Delta w_i\rvert^{3/2}\right)\iff\lvert\Delta w_i\rvert>b_i .
$$

This defines a **no-trade band** $b_i$ around each current position. Small, low-conviction changes are suppressed, which is where most avoidable turnover comes from.

**Worked example.** NAV A\$1m, ADV A\$2m, $\sigma=2\%$ daily, impact coefficient 0.7, half-spread 10 bp, commission 5 bp, margin $m=1$.

| Case | Notional | Impact $0.7\sigma\sqrt{Q/\mathrm{ADV}}$ | Total cost | Expected benefit | Decision |
|---|---:|---:|---:|---:|---|
| $\Delta w=0.6\%$, $\lvert\hat\alpha\rvert h=0.8\%$ | A\$6,000 | 0.077% | 0.227% = A\$13.60 | A\$48.00 | $48.00>2\times13.60$ → **trade** |
| $\Delta w=0.2\%$, $\lvert\hat\alpha\rvert h=0.3\%$ | A\$2,000 | 0.044% | 0.194% = A\$3.89 | A\$6.00 | $6.00<2\times3.89$ → **hold** |

**Invalidation conditions.** Every position records the conditions under which its thesis is wrong: forecast sign flip, price beyond the 5% forecast quantile, stress-regime probability above 0.6, an adverse news shock, or a distress-probability jump. These drive exits and appear in the report.

**NO-TRADE gate (◆ G6).** All rule sets must pass: data (all inputs passed G2), model (enough eligible models per combination), risk (post-trade ES and stress within limits), liquidity (participation and spread limits), market state (no halt or pending corporate action), and the kill switch (Section 13.2).

### 12.5 Contents of the report (per company and per sleeve)

| Section | Content |
|---|---|
| Position | Direction, target weight, current weight, change, action |
| Leverage | Contribution to gross exposure; intraday versus overnight; financing cost |
| Forecast | Median and 5–95% interval at the chosen horizon; contribution of each of the four layers; model agreement |
| Risk | Volatility forecast, contribution to ES, worst three stress scenarios for the name |
| Survival | Distress probability and its trend |
| Exposures | Economy stickers, sector, factors, FX |
| Costs | Expected spread, impact and borrow cost of the trade |
| Exit plan | Invalidation conditions, stop levels, time stop |
| Sleeve summary | Gross and net exposure, leverage components, margin headroom, drawdown status, gate outcomes |

---

## 13. Stages 6–7: Execution, Hard Limits, Monitoring and Learning

> **Principle.** Execution is modelled, not assumed; hard limits sit outside the models; and models change only through controlled evaluation.

### 13.1 Execution modelling

**Expected cost (square-root law).**

$$
\mathbb E[\text{cost}]\approx\frac{s}{2}+Y\sigma\sqrt{\frac{Q}{V}},
$$

where $s$ is the spread, $Q$ the order size, $V$ daily volume and $Y$ a coefficient calibrated from realised fills.

**Optimal scheduling (Almgren–Chriss).** For a large order of size $X$ over horizon $T$, the remaining holding at time $t_j$ is

$$
x_j=X\frac{\sinh\left(\kappa(T-t_j)\right)}{\sinh(\kappa T)},\qquad\kappa=\sqrt{\lambda\sigma^2/\tilde\eta}.
$$

*Interpretation.* Trading fast reduces exposure to price drift but raises impact; trading slowly does the reverse. The risk-aversion parameter $\lambda$ sets the balance. High urgency (e.g. a risk-reducing exit) front-loads the schedule.

**Default venues.** Interday rebalances use the closing auction where the order is a small share of expected auction volume, otherwise a VWAP schedule. Intraday sleeves use limit orders with participation caps.

**Implementation shortfall.** Measured against the decision price $p_d$:
$\mathrm{IS}=\sum_jq_j(p_j-p_d)+\left(Q-\sum_jq_j\right)(p_{\mathrm{end}}-p_d)$, decomposed into impact, delay and opportunity cost. It is used to recalibrate $Y$, closing the loop between the cost model in the optimiser and actual trading.

### 13.2 Hard limits independent of the models

These rules are deliberately simple and live outside the forecasting logic, so that a model error cannot override them:

| Limit | Default |
|---|---|
| Daily loss | Halt new risk if loss exceeds $3\hat\sigma_p$ |
| Drawdown | Halt all trading beyond the maximum tolerable drawdown |
| Model-reality divergence | Realised versus predicted P&L $z$-score above 4 twice in five days |
| Per-name and gross caps | Fixed limits regardless of optimiser output |
| Intraday | Flatten before session close unless explicitly an overnight sleeve |

A halt requires deliberate manual review before trading resumes.

### 13.3 Monitoring

| Type | Method | Threshold |
|---|---|---|
| Data drift | Population stability index $\mathrm{PSI}=\sum_b(a_b-e_b)\ln(a_b/e_b)$ | <0.1 stable; 0.1–0.25 watch; >0.25 drift |
| Concept drift | Page–Hinkley on rolling IC and forecast loss: $m_T=\sum_{t\le T}(x_t-\bar x_t-\delta)$, alarm if $m_T-\min_{t\le T}m_t>\lambda$ | Alarm |
| Calibration drift | Rolling 90% interval coverage | Outside 87–93% for 20 sessions |
| Alpha decay | Half-life shortening | >50% shorter than at admission |
| Risk-model failure | Kupiec / Christoffersen | $p<0.01$ |
| Cost-model error | Predicted versus realised shortfall | Error >30% |

### 13.4 Recalibration policy and model lifecycle

| Trigger | Action | Control |
|---|---|---|
| Loss worse but model still in confidence set | Reweight combination | Automatic |
| Calibration drift | Refit conformal layer; widen intervals | Automatic |
| PSI > 0.25 on key features | Retrain on recent window, run in shadow | ◆ G7 |
| Concept-drift alarm | Reselect signals; search challengers | ◆ G7 |
| Risk-model failure | Switch engine; recalibrate | ◆ G4 |
| Universe funnel shift > 20% | Review Stage 1 thresholds | Manual |

**Lifecycle.** candidate → (G3) → shadow (runs on live data without influence) → challenger → (G7) → champion → retired. Promotion requires the challenger to beat the champion over at least 60 shadow sessions (DM or MCS), with calibration intact. A demoted champion is kept for one cycle as a fallback.

### 13.5 Performance attribution

| Level | Method | Question answered |
|---|---|---|
| Stage | Selection, forecast, allocation, execution | Which stage added or lost value? |
| Layer | Contribution of financial, macro, micro and news models | Which forecast layer is earning its place? |
| Sector / country | Brinson–Fachler allocation, selection, interaction | Where did relative returns come from? |
| Factor | $r_{p,t}=\beta^\top f_t+\epsilon_t$ | Is the return just known factor exposure? |
| Execution | Implementation shortfall by order type | Is the cost model accurate? |

Attribution results feed back into Stage 1 (screen thresholds), Stage 2E (combination weights) and the allocator (sleeve budgets).

---

## 14. Improvements Relative to Version 1

| # | Version 1 | Version 2 | Why it matters |
|---|---|---|---|
| 1 | Models output "expected return and variance" | Every model emits a **quantile distribution on a common grid**, scored by CRPS | Uncertainty flows into sizing and leverage; all four layers combine on equal terms |
| 2 | Combination and validation listed separately | Fixed order: **validate → eligibility gate → combine → calibrate** | Models that have not earned trust never receive weight |
| 3 | Stickers as tags used as priors | Stickers formalised as **versioned revenue and cost exposure vectors**, with **empirical-Bayes shrinkage** of sensitivities toward them | Multi-economy firms handled properly; noisy estimates stabilised; FX exposure measured |
| 4 | Macro and news propagation designed separately | One **relationship graph and transmission network** shared by macro stress, news propagation and contagion scenarios | Consistent view of how shocks spread between firms |
| 5 | Survival filter of separate thresholds | **Hazard model** producing a single distress probability, with thresholds kept as reason codes | Better calibrated exclusion; the same probability penalises exposure in the optimiser |
| 6 | "Mostly fixed" universe via turnover band | **Hysteresis**, forced-exit rules and reason codes | Stable universe with explicit, auditable exceptions |
| 7 | Intraday data for all intraday names | **Data-depth tiers** allocated by value per cost | Expensive data goes where models can use it |
| 8 | Leverage via fractional Kelly and vol target | **Leverage rule** combining vol target, uncertainty-adjusted Kelly, policy cap, margin headroom and **drawdown control**; separate intraday and overnight treatment; dedicated gate | Leverage responds to drawdown and margin conditions, not only forecasts |
| 9 | Costs as optimiser penalties | Costs in the optimiser **and** a **no-trade band** in the decision layer, plus invalidation conditions per position | Removes low-value turnover; every position has an explicit exit thesis |
| 10 | Overfitting control listed | **Trial counts enter the deflated Sharpe ratio**; nested purged validation; Hansen SPA and MCS | The cost of searching many models is charged explicitly |
| 11 | Generic stress tests | **Sticker-driven and graph-driven scenarios** plus regime-mixture covariance and cross-market lead–lag treatment | Stresses hit each firm in proportion to its real exposures |
| 12 | Gates implicit | **Seven explicit gates** with numeric pass criteria | Every exclusion and stop is measurable and explainable |
| 13 | News model reads text into features | Explicit **novelty and priced-in checks**, graph propagation with spectral-radius condition, calibrated shock probability, and strict leakage protocol | Separates genuinely new information from noise and from hindsight |
| 14 | Sleeve allocation by risk budgeting | Budgets that move with out-of-sample evidence, hard intraday cap, explicit FX budget | Capital follows evidence while protecting the core book |

---

## 15. Notation, Glossary and Open Questions

### 15.1 Notation

| Symbol | Meaning | Symbol | Meaning |
|---|---|---|---|
| $i$, $t$ | Security; decision time | $k$, $\tau$ | Knowledge time; valid time |
| $h$ | Forecast horizon (sessions) | $r_{i,t}$ | Log return |
| $\mathbf e_i$ | Economy sticker vector of firm $i$ | $f_{c,t}$ | State factors of economy $c$ |
| $\pi_t(k)$ | Filtered probability of regime $k$ | $W$ | Relationship graph |
| $\hat q_\tau$ | Forecast quantile at level $\tau$ | $p^{\mathrm{dist}}_i$ | Distress probability |
| $w$, $\Sigma$ | Portfolio weights; covariance | $L_t$ | Portfolio leverage |
| $\mathrm{DD}_t$ | Drawdown from peak | $b_i$ | No-trade band |

### 15.2 Glossary

| Term | Meaning |
|---|---|
| Point-in-time | Data as it was known at a given moment |
| Bitemporal | Stored with both valid time and knowledge time |
| Sleeve | An independently configured market × horizon book |
| Archetype | A strategy family that determines selection criteria |
| CRPS | Continuous ranked probability score; rewards calibrated, sharp distributions |
| Conformal calibration | Adjusts intervals to achieve target coverage on held-out data |
| Purging / embargo | Removing overlapping or adjacent observations to prevent validation leakage |
| DSR / PBO | Deflated Sharpe ratio; probability of backtest overfitting |
| SPA / MCS | Superior predictive ability test; model confidence set |
| ES | Expected shortfall: average loss beyond VaR |
| Fractional Kelly | A fraction of growth-optimal leverage, used to allow for estimation error |
| Implementation shortfall | Difference between the decision price and the achieved execution |

### 15.3 Open questions that shape the next design step

1. **Capital and instruments:** cash equities on margin, CFDs or futures? This determines margin mechanics, financing cost and the realistic leverage ceiling.
2. **Maximum tolerable drawdown** and target volatility per sleeve: these set $\mathrm{DD}_{\max}$ and $\sigma^\ast$ in the leverage rule.
3. **Broker and data provider for US markets:** the reference architecture covers ASX through LSEG; US intraday data depth and cost need confirming.
4. **News sources and latency:** licensed wire versus exchange announcements only; this decides whether the intraday news signal can be directional.
5. **Short selling:** whether sleeves are long-only or long-short, given ASX borrow constraints.
6. **Definition of "highest tier":** whether you want a fixed set of model families or an open challenger process governed by the eligibility and promotion gates (the design above assumes the latter).
