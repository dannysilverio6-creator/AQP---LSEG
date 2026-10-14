# Multi-Market Quantitative Investment Pipeline

**Scope:** ASX and US equities, interday and intraday, live capital
**Structure:** Archetype selection, then per-company forecasting, then portfolio optimisation, then leverage and position report, with monitoring and validation around all stages.

---

## 1. Pipeline at a Glance

```
Stage 0   Point-in-time data + Company Knowledge Layer
   |
Stage 1   Archetype Builder (selection only)  ->  universe per sleeve
   |
Stage 2   Forecast models: Financial | Macro | Micro | News (LLM)
   |
Stage 2E  Signal combination and calibration
   |
Stage 3   Risk model (covariance, tails, exposures)
   |
Stage 4   Portfolio optimiser (per sleeve) + sleeve capital allocator
   |
Stage 5   Leverage and position report
   |
Stage 6   Execution + independent hard risk limits
   |
Stage 7   Monitoring and feedback  --> retrain Stage 2, revise Stage 1

Validation harness applies to Stages 2, 4 and 5 (Section 11).
```

### Sleeves

The pipeline runs as four independent sleeves with a shared architecture and separate configuration (universe filters, calendar, cost model, data feed, risk budget).

| Sleeve | Calendar and currency | Distinguishing constraints |
|---|---|---|
| ASX interday | ASX hours, AUD | ASX costs, short-borrow availability |
| US interday | US hours (overnight in Sydney), USD | FX risk, overnight gap risk |
| ASX intraday | ASX hours with auctions, AUD | Tick size, auction handling, spread sensitivity |
| US intraday | US hours (overnight in Sydney), USD | Market impact, data cost, latency |

---

## 2. Stage 0: Point-in-Time Data and Company Knowledge Layer

### 2.1 Data inputs

| Source | Requirements |
|---|---|
| Market data | Daily and intraday bars, ticks, order book (L1/L2). Adjusted for splits and dividends. **Must include delisted names.** |
| Fundamentals | Filings and consensus estimates stored with announcement timestamps. Restatement-aware. |
| Macro series | Real-time vintages (values as first published, not later revised). Release calendar for RBA, FOMC, CPI, GDP and jobs data. |
| News and filings | Wire, regulatory announcements, transcripts, each with strict publication timestamps. |
| Borrow and margin | Short availability, borrow cost, margin rates. |

### 2.2 Company Knowledge Layer

Built once, versioned, and read by every later stage so that selection, forecasting and exposure constraints stay consistent.

1. **Macro stickers:** each company's revenue and cost exposure by economy, taken from segment disclosures rather than listing location or headquarters.
2. **Market and industry labels.**
3. **Data-driven peer clusters:** clustering on returns and business-description embeddings, used alongside standard classifications.
4. **Relationship graph:** suppliers, customers and competitors, extracted from filings and articles.
5. **Point-in-time membership and delisting history.**

---

## 3. Stage 1: Archetype Builder

Purpose: select companies only. No return forecasting happens here.

| Step | Detail |
|---|---|
| 1. Hard survival filter (all sleeves) | Altman Z-score or Merton distance-to-default, interest coverage, net debt / EBITDA, refinancing wall, going-concern flags. Exclude names below threshold. |
| 2. Liquidity screen | ADV, spread in basis points, depth, free float. Intraday sleeves use tight limits; interday sleeves are capacity-aware. |
| 3. Strategy archetypes | Momentum / trend, quality-value, mean reversion / pairs, event-driven, intraday liquid large-cap. Each name carries an archetype tag. |
| 4. Reconstitution rules | Quarterly universe update, turnover band for stability, entry and exit hysteresis. |

**Output:** a universe per sleeve with an archetype tag per name.

The aim of "mostly fixed" membership is achieved by the turnover band and hysteresis, not by freezing the universe. The survival filter remains a hard exclusion at every reconstitution.

---

## 4. Stage 2A: Financial Model

"Highest tier" is defined by task and by out-of-sample performance rather than by model complexity. Every model must beat a simple benchmark out of sample.

| Task | Model families | Notes |
|---|---|---|
| Return and earnings forecast | Elastic net regression; gradient boosting (LightGBM, XGBoost) | Features: earnings revisions, accruals, margins, leverage, quality. Benchmarked against naive and factor baselines. |
| Volatility forecast | GARCH family (GJR, EGARCH); HAR on realised volatility | Feeds the risk model and optimiser directly. Intraday uses realised variance from bars. |
| Distress | Merton structural model; discrete-time hazard model | Produces a survival probability per name, used in Stages 1, 3 and 5. |
| Time-varying loadings | Kalman filter (state-space) | Optional layer for drifting factor exposures. |

---

## 5. Stage 2B: Macro Model

1. **Read stickers** from the knowledge layer.
2. **Regime model:** Markov-switching or hidden Markov model over growth, inflation, rates and risk-off states.
3. **Stock-level macro sensitivities:** time-varying betas to macro factors, **shrunk toward the sticker-implied exposure** and estimated hierarchically so that short-history names get stable estimates.
4. **Event calendar module:** surprise defined as actual minus consensus. Used as a volatility and direction trigger in intraday sleeves.
5. **FX channel:** AUD/USD treated as an explicit risk factor.

**Cautions:** stickers must be versioned because exposure changes over time; macro data must use real-time vintages to avoid look-ahead.

---

## 6. Stage 2C: Micro Model

1. **Labels and peer groups:** market and industry labels plus data-driven peer clusters.
2. **Hierarchical or panel models:** borrow strength across peers for sparse, low-frequency variables such as pricing, input costs, demand, competition and margins.
3. **Horizon-specific features:**
   - Interday: valuation and sector structure.
   - Intraday: spread, order-book imbalance, volume profile, VWAP deviation.

---

## 7. Stage 2D: News Model (LLM as Feature Extractor)

| Step | Detail |
|---|---|
| 1. Ingest and timestamp | Deduplicate, link entities, score source reliability. |
| 2. LLM structured extraction | Output fields: event type, affected entities, direction, magnitude, confidence, transmission channel. Structured fields rather than free-text sentiment. |
| 3. Novelty and priced-in check | Compare with the price reaction since publication. Most news in liquid names is priced within seconds to minutes. |
| 4. Relationship-graph propagation | Spread the event to suppliers, customers and peers, with decay by graph distance and time. |
| 5. Downstream return model | Structured fields enter a model trained on realised returns. The LLM extracts features; calibration comes from the downstream model. |

**Leakage control:** an LLM trained after the backtest period already knows what happened. Evaluate only on news after the model's training cutoff, enforce strict timestamps, and treat forward paper trading as the real validation for this layer.

### Relative importance of the four layers by horizon

| Layer | Interday | Intraday |
|---|---|---|
| Financial | Core driver | Filter only (distress exclusion) |
| Macro | Regime and factor loadings | Scheduled-event triggers |
| Micro | Valuation and sector structure | Microstructure |
| News | Multi-day drift | Fast decay, latency-sensitive |

---

## 8. Stage 2E: Signal Combination

- **Stacking or Bayesian model averaging**, with weights learned out of sample and separate weights per sleeve horizon.
- **Uncertainty calibration** using conformal or quantile methods, checked with reliability diagrams.
- **Output per name:** expected return, forecast variance, confidence, and survival probability.

---

## 9. Stage 3 and Stage 4: Risk Model, Optimiser, Allocator

### 9.1 Risk model (Stage 3)

- **Covariance:** Ledoit-Wolf shrinkage; factor model plus idiosyncratic risk; DCC-GARCH for time variation.
- **Tail and gap risk:** expected shortfall; overnight gap risk for US positions.
- **Exposure map:** sector, country, macro factor, FX and style factors.

### 9.2 Optimiser (Stage 4, per sleeve)

| Component | Detail |
|---|---|
| Return inputs | Black-Litterman: forecasts enter as views over an equilibrium prior. |
| Objective options | Shrunk mean-variance; hierarchical risk parity; CVaR minimisation. |
| Cost-aware objective | Spread, commission, market impact, borrow cost, turnover penalty included in the objective. |
| Constraints | Position caps, sector and factor limits, liquidity participation limit, long-only or long-short rules. |

### 9.3 Sleeve capital allocator (Stage 4B)

- Risk budgeting across sleeves (equal risk contribution or Bayesian), including cross-sleeve correlation.
- An intraday risk cap so intraday losses cannot consume interday risk capacity.

---

## 10. Stage 5 to Stage 7: Leverage, Execution, Monitoring

### 10.1 Leverage and position report (Stage 5)

| Component | Detail |
|---|---|
| Position sizing | Fractional Kelly (typically 0.25 to 0.5 of full Kelly, because estimation error makes full Kelly dangerous) and volatility targeting. Sizing is kept separate from leverage. |
| Leverage decision | Gross and net exposure per sleeve; intraday versus overnight leverage; margin use and financing cost. |
| Stress tests | Historical crises, factor shocks, gap risk, FX shock, liquidity squeeze; drawdown and margin-call distance. |

**Report per company:** direction, size, leverage, stop and exit rules, expected return, risk contribution, cost estimate, survival probability, and model agreement.

### 10.2 Execution and independent risk controls (Stage 6)

- **Hard limits outside model code:** daily loss limit, per-name and gross caps, kill switch, automatic de-leverage. A model bug cannot override them.
- **Execution algorithms:** TWAP, VWAP, participation and limit-order logic; ASX auction handling.
- **Broker reconciliation:** positions, cash and margin; audit log of every decision.

### 10.3 Monitoring and feedback (Stage 7)

- **Forecast versus realised:** error by model, layer, sleeve and regime.
- **Drift detection:** PSI, CUSUM, change-point tests.
- **Model decay handling:** automatic down-weighting, retrain schedule, champion-challenger.
- **Performance attribution:** selection, forecast, allocation and execution.
- **Feedback:** weights and retraining flow back to Stages 2 and 2E; attribution results revise the Stage 1 screens.

---

## 11. Validation Harness

Applies to every stage that produces a forecast, allocation or size.

1. **Walk-forward evaluation** with purged and embargoed cross-validation.
2. **Overfitting control:** deflated Sharpe ratio, multiple-testing correction, untouched final holdout.
3. **Leakage control:** point-in-time data only; real-time macro vintages; LLM evaluated only after its training cutoff.
4. **Baseline test:** every model beats a simple benchmark out of sample.

### Deployment gates

| Gate | Requirement |
|---|---|
| 1. Research | Out-of-sample results pass the checks above. |
| 2. Paper trading | Realised versus simulated fills compared. Intraday sleeves need the longest period here. |
| 3. Small live capital | Hard limits active and tested. |
| 4. Capital ramp | Scale in steps tied to out-of-sample results, with predefined shutdown criteria per sleeve. |

---

## 12. Practical Considerations for Live Capital

- **Currency:** decide whether AUD/USD exposure is hedged or left as a sleeve-level exposure.
- **Costs:** intraday gross edge is often fully consumed by costs; treat intraday backtests as screening only.
- **Borrow:** ASX short availability and cost constrain long-short sleeves.
- **Operations:** broker API reliability, data licensing cost (tick and minute data are expensive), position reconciliation.
- **Tax:** confirm the treatment of frequent trading and US-listed holdings with an accountant before scaling.

---

## 13. Suggested Build Order

1. Build Stage 0 and the knowledge layer, then a simple Stage 1 universe filter.
2. Implement one baseline model per layer, a shrinkage-covariance optimiser and a volatility-targeted sizing report, end to end, on the interday sleeves.
3. Add the monitoring loop and hard limits before any capital is committed.
4. Run intraday sleeves in extended paper trading before any live capital.
5. Improve each stage individually, measuring the effect on out-of-sample results.

---

## 14. Open Questions

- Target capital size and instruments (cash equities on margin, futures, CFDs)?
- Broker and data provider (these constrain intraday design)?
- News sources and latency available (licensed wire, filings, free feeds)?
- Definition of "highest tier" for the financial layer: specific model families or an ensemble?
