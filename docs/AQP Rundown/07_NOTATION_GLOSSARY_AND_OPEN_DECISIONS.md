# Notation, Glossary and Open Decisions

> **AQP Architecture Version 14** · [Document index](README.md) · [Overall design](00_AQP_OVERALL_DESIGN.md)


**In scope:** Shared notation, abbreviations and implementation decisions still to settle.

---

| Symbol / term | Meaning |
|---|---|
| $t$, $a$, $v$ | Decision time, first-available timestamp and valid time |
| $h$, $r_{i,t}$ | Forecast horizon and security return |
| $e^{rev}_{i,c}$, $e^{cost}_{i,c}$ | Security $i$'s dated revenue and cost exposure shares for economy $c$ |
| $f_{c,t}$ | Macroeconomic state/factor for economy $c$ at time $t$ |
| $q_{i,t}(\tau)$ | Forecast quantile at probability level $\tau$ |
| $Z^{news}_{i,t,h}$ | Versioned, point-in-time quantitative news covariate vector produced by NLP/LLM extraction and validation; indexed to the relevant entity, decision time and horizon |
| No-news baseline / news challenger | Matched core-feature model / same model family and target with quantitative news covariates added |
| Incremental news value | Paired out-of-sample change in forecast and economic metrics attributable to adding news features under the registered experiment |
| $\hat{\Sigma}$ | Estimated covariance matrix |
| $p_i^{dist}$ | Estimated distress/default probability |
| $w_i$, $L_t$, $DD_t$ | Portfolio weight, leverage and drawdown |
| $b_i$ | No-trade band for security $i$ |
| Purging / embargo | Removing overlapping labels and a defined buffer around test periods during validation |

## Acronyms and established model names

| Term | Meaning |
|---|---|
| VaR / ES | Value at Risk / Expected Shortfall; some literature uses Conditional Value at Risk (CVaR) for related tail-risk measures |
| CRPS / PIT | Continuous Ranked Probability Score / Probability Integral Transform |
| GARCH / GJR-GARCH / EGARCH | Generalised Autoregressive Conditional Heteroskedasticity / Glosten–Jagannathan–Runkle GARCH / Exponential GARCH |
| HAR-RV | Heterogeneous Autoregressive model of Realised Volatility |
| MIDAS / DCC-GARCH | Mixed Data Sampling model / Dynamic Conditional Correlation GARCH model |
| DML | Double/debiased machine learning (academic terminology; both forms are used in the literature) |
| EWMA | Exponentially weighted moving average |
| DSR / PBO | Deflated Sharpe Ratio / Probability of Backtest Overfitting |
| SPA / MCS | Superior Predictive Ability test / Model Confidence Set |
| NAV / ADV / FX | Net asset value / average daily volume / foreign exchange |
| OLS / EVT / BVAR | Ordinary Least Squares / Extreme Value Theory / Bayesian Vector Autoregression |
| Champion / challenger | Current approved model / candidate model undergoing independent evaluation |
| G1–G7 | Project-specific gate identifiers, not external regulatory rule numbers |

## Decisions required before implementation

1. Instruments and capital structure: cash equities on margin, CFDs or futures; available capital and
   financing arrangements.
2. Per-sleeve target volatility, maximum drawdown, per-name/sector/gross limits and margin buffer.
3. US-market broker and provider; licensed data coverage and intraday data cost/depth.
4. Licensed news sources, publication/first-seen timestamp quality, NLP/LLM and prompt versioning,
   retention/privacy requirements, minimum coverage, latency budgets, missingness policy, and criteria
   for any intraday directional use.
5. Long-only versus long-short and permitted borrow, especially for ASX names.
6. Whether universe re-formation is only by deliberate decision or on a defined schedule.
7. Whether model families are fixed or open to a challenger process governed by G3/G7.

---

---

**Related documents:** [Overall design](00_AQP_OVERALL_DESIGN.md) · [Operating contract and run outputs](08_OPERATING_CONTRACT_AND_RUN_OUTPUTS.md) · [Implementation roadmap and readiness gates](09_IMPLEMENTATION_ROADMAP_AND_READINESS_GATES.md) · [Selected technical references](10_SELECTED_TECHNICAL_REFERENCES.md)
