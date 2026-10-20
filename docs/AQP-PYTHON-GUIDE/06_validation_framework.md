# Part 06 — The validation framework (build the judge before the contestants)

**Goal:** decide *how a model will be judged* before writing any model. That means the target and its label end date, chronological folds with purging and embargo, proper scores for quantile forecasts, a test of whether one forecast is really better than another, a net-of-cost trading test, and a penalty for having tried many models.

**Design:** §5 (target and label end date), §10 (validation protocol, G3 criteria), §17 (roadmap phase 3, *"leakage tests work; trials and fold boundaries logged"*).

These are pure functions on arrays and tables, so you can test each one against numbers you work out by hand. Part 08 wires them into the Stage 02E service.

---

## 6.1 The target and its label end date

Design §5:

```text
y_{i,t}^{(h)} = Σ_{s=2}^{h+1} ( r_{i,t+s} − r_{b,t+s} )
```

This is the benchmark-relative (excess) return over `h` sessions, **starting two sessions after the decision**. The decision is taken after the close on day `t`, the order executes at the close of `t+1`, and so the first return the position can earn is on `t+2`. The last return in the sum is on day `t+h+1`, the **label end date**. Until that day has closed, the label is unknown.

```text
day:      t      t+1      t+2   t+3   t+4   t+5   t+6        (h = 5)
          decide execute  |-------- label window --------|
                                                     label_end = t+6
```

**File:** `src/aqp_python/stages/stage_02_forecasting/label_builder.py`

```python
"""Target: y(h)_t = sum_{s=2}^{h+1} (r_i,t+s - r_b,t+s), with its label_end date for purging.

The sum starts at s=2 because a decision taken after the close on day t is
executed at the close of t+1, so the first return the position earns is t+2.
"""
import pandas as pd


def excess_returns_wide(prices: pd.DataFrame, benchmark: pd.DataFrame,
                        trading_days: pd.DatetimeIndex) -> pd.DataFrame:
    wide = prices.pivot(index="date", columns="security_id", values="return").reindex(trading_days)
    bench = benchmark.set_index("date")["benchmark_return"].reindex(trading_days)
    return wide.sub(bench, axis=0)


def build_labels(prices: pd.DataFrame, benchmark: pd.DataFrame,
                 trading_days: pd.DatetimeIndex, horizon: int) -> pd.DataFrame:
    excess = excess_returns_wide(prices, benchmark, trading_days)
    # rolling(h).sum() at k covers k-h+1..k; shifting by -(h+1) aligns it to t = k-h-1.
    label = excess.rolling(horizon, min_periods=horizon).sum().shift(-(horizon + 1))
    label_end = pd.Series(trading_days, index=trading_days).shift(-(horizon + 1))
    label.index.name = "date"
    long = label.reset_index().melt(id_vars="date", var_name="security_id", value_name="label")
    long["label_end"] = long["date"].map(label_end)
    return long.dropna(subset=["label", "label_end"]).reset_index(drop=True)
```

The rolling-sum trick: `rolling(h).sum()` at row `k` adds rows `k−h+1..k`. Shifting the result up by `h+1` places that sum on row `t = k−h−1`, which is exactly `Σ_{s=2}^{h+1}`. The unit test in 6.6 checks this on numbers you can add up yourself. `dropna` removes the last `h+1` rows of each security, whose futures are incomplete. A row with an unknown label must never be used for training.

## 6.2 Walk-forward folds, purging and embargo

```mermaid
gantt
    dateFormat  YYYY-MM-DD
    axisFormat  %b
    section Fold k
    train (past only)            :done, 2024-01-01, 2024-09-20
    purged: label ends inside test :crit, 2024-09-20, 2024-09-30
    test block                   :active, 2024-09-30, 2024-12-31
    embargo (if later data used) :crit, 2024-12-31, 2025-01-08
```

- **Walk-forward:** test blocks move forward in time, and each model is trained only on data from before its test block.
- **Purging:** a training row whose label window `[date, label_end]` overlaps the test block is removed, because its label contains returns from the test period.
- **Embargo:** rows just after the test block are removed, in case later data is used (for example in cross-validation schemes that train on both sides). This build uses expanding windows (past only), so the embargo is a second line of defence, and it is tested on its own.
- **Non-overlapping test dates:** test dates are spaced `step_days = 5` apart, so consecutive test labels do not overlap and the per-date scores are closer to independent.

**File:** `src/aqp_python/stages/stage_02_forecasting/validation/walk_forward_folds.py`

```python
"""Chronological folds with purging and embargo (train only on the past)."""
from dataclasses import dataclass

import numpy as np
import pandas as pd


@dataclass(frozen=True)
class Fold:
    number: int
    test_dates: pd.DatetimeIndex

    @property
    def test_start(self) -> pd.Timestamp:
        return self.test_dates[0]

    @property
    def test_end(self) -> pd.Timestamp:
        return self.test_dates[-1]


def make_folds(trading_days: pd.DatetimeIndex, first_date: pd.Timestamp, last_date: pd.Timestamp, *,
               min_train_days: int, n_folds: int, step_days: int) -> list[Fold]:
    """Evaluate every `step_days`-th session after a burn-in, split into contiguous blocks.

    first_date/last_date bound the usable research rows; the first test date sits
    `min_train_days` sessions after first_date so fold 1 has a real training set.
    """
    start = trading_days.searchsorted(first_date) + min_train_days
    candidates = trading_days[start:]
    candidates = candidates[candidates <= last_date][::step_days]
    if len(candidates) < n_folds * 5:
        raise ValueError("not enough history for the requested number of folds")
    return [Fold(i + 1, pd.DatetimeIndex(block))
            for i, block in enumerate(np.array_split(candidates.to_numpy(), n_folds))]


def purge_and_embargo(rows: pd.DataFrame, fold: Fold, trading_days: pd.DatetimeIndex,
                      embargo_days: int) -> pd.DataFrame:
    """Training rows for a fold.

    Purge: drop any row whose label window [date, label_end] touches the test period.
    Embargo: drop rows starting within `embargo_days` sessions after the test period.
    """
    overlaps = (rows["date"] <= fold.test_end) & (rows["label_end"] >= fold.test_start)
    end_position = trading_days.searchsorted(fold.test_end)
    embargo_end = trading_days[min(end_position + embargo_days, len(trading_days) - 1)]
    embargoed = (rows["date"] > fold.test_end) & (rows["date"] <= embargo_end)
    return rows.loc[~overlaps & ~embargoed]


def expanding_train(rows: pd.DataFrame, fold: Fold, trading_days: pd.DatetimeIndex,
                    embargo_days: int) -> pd.DataFrame:
    """Expanding window: past rows only, then purge/embargo as a second line of defence."""
    past = rows.loc[rows["date"] < fold.test_start]
    return purge_and_embargo(past, fold, trading_days, embargo_days)
```

`make_folds` starts the first test date `min_train_days` sessions after the first usable row, so fold 1 has a real training set. Then it splits the candidate dates into `n_folds` contiguous blocks.

## 6.3 Scoring quantile forecasts

The **pinball loss** for quantile `τ` and error `e = y − q_τ` is `ρ_τ(e) = max(τe, (τ−1)e)`. Under-predicting the 95% quantile costs 0.95 per unit; over-predicting it costs only 0.05. Averaged over the grid, it is a *proper* score: on average, a forecaster minimises it by reporting their true belief. It is the design's primary score.

**Coverage:** the share of outcomes inside the 5%–95% interval. A calibrated forecast covers about 90%. **Sharpness:** the average width of that interval. Among calibrated forecasts, narrower is better.

**File:** `src/aqp_python/stages/stage_02_forecasting/validation/forecast_scoring.py`

```python
"""Proper scoring rules for quantile forecasts."""
import numpy as np

from aqp_python.contracts.quantile_grid import QUANTILE_GRID

TAUS = np.asarray(QUANTILE_GRID)
LOWER_90, UPPER_90 = list(QUANTILE_GRID).index(0.05), list(QUANTILE_GRID).index(0.95)


def pinball_loss(y: np.ndarray, q: np.ndarray) -> np.ndarray:
    """Mean pinball loss across the grid, one value per row.

    rho_tau(e) = max(tau * e, (tau - 1) * e),  with e = y - q_tau.
    """
    error = y[:, None] - q
    return np.maximum(TAUS * error, (TAUS - 1) * error).mean(axis=1)


def coverage_90(y: np.ndarray, q: np.ndarray) -> float:
    """Share of outcomes inside the 5%-95% interval. Target 0.90."""
    inside = (y >= q[:, LOWER_90]) & (y <= q[:, UPPER_90])
    return float(inside.mean())


def width_90(q: np.ndarray) -> float:
    """Sharpness: average width of the 90% interval (narrower is better if calibrated)."""
    return float((q[:, UPPER_90] - q[:, LOWER_90]).mean())
```

## 6.4 Is the candidate really better? Diebold–Mariano

A candidate's average pinball loss will always differ slightly from the baseline's. The question is whether the difference is larger than chance.

```text
d_t   = L_candidate,t − L_baseline,t          one number per test date
DM    = mean(d) / sqrt( HAC variance of d / n )
p     = Φ(DM)                                  one-sided: small p ⇒ candidate has lower loss
```

**HAC** (Newey–West) variance allows for correlation between nearby `d_t`. Losses are averaged *per date* first, because rows on the same date share market moves and are not independent.

**File:** `src/aqp_python/stages/stage_02_forecasting/validation/forecast_comparison_tests.py`

```python
"""Diebold-Mariano test on a time series of loss differences, with HAC variance."""
import numpy as np
from scipy.stats import norm


def newey_west_variance(x: np.ndarray, lags: int) -> float:
    x = x - x.mean()
    n = len(x)
    variance = x @ x / n
    for lag in range(1, lags + 1):
        weight = 1 - lag / (lags + 1)                 # Bartlett kernel
        variance += 2 * weight * (x[lag:] @ x[:-lag]) / n
    return float(variance)


def diebold_mariano(loss_candidate: np.ndarray, loss_baseline: np.ndarray,
                    lags: int) -> tuple[float, float]:
    """One-sided test of H0: equal accuracy vs H1: candidate has lower expected loss.

    Pass per-date average losses (one number per test date), not per-row losses:
    rows on the same date are cross-sectionally dependent.
    Returns (statistic, p_value); a small p-value favours the candidate.
    """
    d = np.asarray(loss_candidate) - np.asarray(loss_baseline)
    n = len(d)
    if n < 10:
        raise ValueError("too few observations for a Diebold-Mariano test")
    variance = newey_west_variance(d, lags)
    if variance <= 0:
        return 0.0, 0.5
    statistic = d.mean() / np.sqrt(variance / n)
    return float(statistic), float(norm.cdf(statistic))
```

## 6.5 Does it make money after costs? And how many times did we try?

**Net of cost.** A simple, transparent test portfolio: on each test date, go long the top 20% of names by forecast median and short the bottom 20%, with equal weights. Earn their realised labels, and pay `cost × turnover` (spread plus commission per unit traded). The annualised Sharpe ratio of the net returns must be positive (G3).

**File:** `src/aqp_python/stages/stage_02_forecasting/validation/net_of_cost_evaluation.py`

```python
"""Does the forecast make money after costs? A simple long-short test portfolio."""
import numpy as np
import pandas as pd


def long_short_returns(oos: pd.DataFrame, fraction: float, cost_per_unit_turnover: float) -> pd.Series:
    """Per test date: long the top `fraction` by forecast median, short the bottom, equal weights.

    oos needs columns: date, security_id, median, label. Returns net returns per date.
    Test dates are spaced `step_days` apart so the h-day labels do not overlap.
    """
    returns, previous = {}, pd.Series(dtype=float)
    for date, day in oos.groupby("date", sort=True):
        n = int(np.floor(len(day) * fraction))
        if n < 1:
            continue
        ranked = day.sort_values("median")
        weights = pd.Series(0.0, index=day["security_id"])
        weights[ranked["security_id"].iloc[-n:]] = 1.0 / n
        weights[ranked["security_id"].iloc[:n]] = -1.0 / n
        gross = float((weights * day.set_index("security_id")["label"]).sum())
        turnover = float(weights.sub(previous, fill_value=0.0).abs().sum())
        returns[date] = gross - turnover * cost_per_unit_turnover
        previous = weights
    return pd.Series(returns, name="net_return")


def annualised_sharpe(returns: pd.Series, periods_per_year: float) -> float:
    if len(returns) < 2 or returns.std() == 0:
        return 0.0
    return float(returns.mean() / returns.std() * np.sqrt(periods_per_year))
```

**Charging for search** (design §2, §10). If you try 100 strategies, the best one will look good by luck. The **deflated Sharpe ratio** (Bailey and López de Prado, 2014) asks: *what is the probability that the true Sharpe ratio exceeds what the best of N trials would reach by chance?*

```text
SR̂      per-period Sharpe of the net returns
σ_SR    = sqrt( (1 − γ₃·SR̂ + (γ₄−1)/4 · SR̂²) / (n−1) )        γ₃ skew, γ₄ kurtosis
SR₀     = σ_SR · [ (1−γ) Φ⁻¹(1 − 1/N) + γ Φ⁻¹(1 − 1/(N e)) ]   γ = 0.5772 (Euler)
DSR     = Φ( (SR̂ − SR₀) / σ_SR )
```

With one trial, `SR₀ = 0` and DSR is the probability that the Sharpe ratio is positive. Each extra trial raises the bar.

**File:** `src/aqp_python/stages/stage_02_forecasting/validation/overfitting_controls.py`

```python
"""Deflated Sharpe ratio (Bailey and Lopez de Prado, 2014): charge for the number of trials."""
import numpy as np
from scipy.stats import kurtosis, norm, skew

EULER_GAMMA = 0.5772156649


def sharpe_std(sr: float, n: int, skewness: float, kurt: float) -> float:
    """Standard error of a per-period Sharpe ratio under non-normal returns."""
    return float(np.sqrt(max(1 - skewness * sr + (kurt - 1) / 4 * sr ** 2, 1e-12) / (n - 1)))


def expected_max_sharpe(n_trials: int, sr_dispersion: float) -> float:
    """Sharpe ratio a best-of-N search would reach by luck alone (per period)."""
    if n_trials <= 1:
        return 0.0
    return sr_dispersion * ((1 - EULER_GAMMA) * norm.ppf(1 - 1 / n_trials)
                            + EULER_GAMMA * norm.ppf(1 - 1 / (n_trials * np.e)))


def deflated_sharpe_ratio(returns: np.ndarray, n_trials: int) -> float:
    """Probability that the true Sharpe exceeds the luck benchmark of N trials.

    Uses the estimation standard error as the dispersion of Sharpe ratios across
    trials - an approximation documented in docs/decisions when better data exist.
    """
    returns = np.asarray(returns, dtype=float)
    n = len(returns)
    if n < 10 or returns.std(ddof=1) == 0:
        return 0.0
    sr = returns.mean() / returns.std(ddof=1)
    g3, g4 = float(skew(returns)), float(kurtosis(returns, fisher=False))
    sigma = sharpe_std(sr, n, g3, g4)
    benchmark = expected_max_sharpe(n_trials, sigma)
    return float(norm.cdf((sr - benchmark) / sigma))
```

`N` comes from a **trial registry** that remembers every distinct model configuration ever evaluated:

**File:** `src/aqp_python/storage/trial_registry.py`

```python
"""Charge for search: every distinct model configuration ever evaluated is a trial."""
import hashlib
import json
from datetime import UTC, datetime
from pathlib import Path


def trial_key(model_name: str, model_version: str, params: dict, features: list[str]) -> str:
    payload = json.dumps({"m": model_name, "v": model_version, "p": params, "f": sorted(features)},
                         sort_keys=True)
    return hashlib.sha256(payload.encode("utf-8")).hexdigest()[:16]


def record_trial(registry_path: Path, key: str, record: dict) -> int:
    """Append the trial if new and return the number of distinct trials so far.

    Re-running an identical configuration is not a new trial, so the count
    measures research breadth, not how often the pipeline ran.
    """
    registry_path.parent.mkdir(parents=True, exist_ok=True)
    keys = set()
    if registry_path.is_file():
        with registry_path.open("r", encoding="utf-8") as handle:
            keys = {json.loads(line)["trial_key"] for line in handle if line.strip()}
    if key not in keys:
        entry = {"trial_key": key, "recorded_at": datetime.now(UTC).isoformat(), **record}
        with registry_path.open("a", encoding="utf-8") as handle:
            handle.write(json.dumps(entry, default=str) + "\n")
        keys.add(key)
    return len(keys)
```

The trial key hashes the model name, version, parameters and feature list. Re-running the same configuration is not a new trial, so running the pipeline daily does not raise the bar artificially. Changing `alpha` from 10 to 5 *is* a new trial, and stays counted even if you change it back.

## 6.6 Tests

**File:** `tests/unit/test_validation_framework.py`

```python
import numpy as np
import pandas as pd
import pytest

from aqp_python.stages.stage_02_forecasting.label_builder import build_labels
from aqp_python.stages.stage_02_forecasting.validation.forecast_comparison_tests import diebold_mariano
from aqp_python.stages.stage_02_forecasting.validation.forecast_scoring import coverage_90, pinball_loss
from aqp_python.stages.stage_02_forecasting.validation.overfitting_controls import deflated_sharpe_ratio
from aqp_python.stages.stage_02_forecasting.validation.walk_forward_folds import (
    Fold,
    make_folds,
    purge_and_embargo,
)
from aqp_python.storage.trial_registry import record_trial

pytestmark = pytest.mark.batch03
DAYS = pd.bdate_range("2026-01-01", periods=12)


def test_label_skips_one_session_and_sums_h_excess_returns():
    returns = [0.0, 0.01, 0.02, 0.03, 0.04, 0.05, 0.06, 0.07, 0.08, 0.09, 0.10, 0.11]
    prices = pd.DataFrame({"security_id": "A", "date": DAYS, "return": returns})
    bench = pd.DataFrame({"date": DAYS, "benchmark_return": 0.01})
    labels = build_labels(prices, bench, DAYS, horizon=3).set_index("date")
    # t = day 0: sum of excess returns on days 2, 3, 4 = (0.02+0.03+0.04) - 3*0.01
    assert labels.loc[DAYS[0], "label"] == pytest.approx(0.06)
    assert labels.loc[DAYS[0], "label_end"] == DAYS[4]
    assert labels.index.max() == DAYS[-5]            # later rows have incomplete futures


def test_folds_are_chronological_and_disjoint():
    days = pd.bdate_range("2020-01-01", periods=600)
    folds = make_folds(days, days[0], days[-10], min_train_days=200, n_folds=4, step_days=5)
    ends = [f.test_end for f in folds]
    assert ends == sorted(ends)
    assert all(a.test_end < b.test_start for a, b in zip(folds, folds[1:]))


def test_purge_removes_overlapping_labels_and_embargo_removes_the_gap():
    days = pd.bdate_range("2026-01-01", periods=40)
    rows = pd.DataFrame({"date": days, "label_end": days.shift(6, freq="B")})
    fold = Fold(1, days[15:20])
    kept = purge_and_embargo(rows, fold, days, embargo_days=5)
    assert not ((kept["date"] <= fold.test_end) & (kept["label_end"] >= fold.test_start)).any()
    assert not kept["date"].between(days[20], days[24]).any()
    assert days[14 - 6] in set(kept["date"])          # label ends before the test start: kept


def test_pinball_of_a_perfect_point_forecast_is_zero():
    y = np.array([0.01, -0.02])
    assert pinball_loss(y, np.repeat(y[:, None], 11, axis=1)).tolist() == [0.0, 0.0]


def test_coverage_counts_outcomes_inside_the_90_interval():
    q = np.tile(np.linspace(-1, 1, 11), (4, 1))
    assert coverage_90(np.array([0.0, 0.5, 2.0, -2.0]), q) == 0.5


def test_diebold_mariano_detects_a_clearly_better_forecast():
    rng = np.random.default_rng(0)
    base = rng.uniform(1.0, 2.0, 200)
    _, p_better = diebold_mariano(base - 0.3, base, lags=1)
    _, p_same = diebold_mariano(base + rng.normal(0, 0.01, 200), base, lags=1)
    assert p_better < 0.01 and p_same > 0.05


def test_more_trials_lower_the_deflated_sharpe():
    returns = np.random.default_rng(1).normal(0.002, 0.01, 250)
    assert deflated_sharpe_ratio(returns, 1) > deflated_sharpe_ratio(returns, 100)


def test_identical_trials_are_counted_once(tmp_path):
    path = tmp_path / "trials.jsonl"
    assert record_trial(path, "k1", {}) == 1
    assert record_trial(path, "k1", {}) == 1
    assert record_trial(path, "k2", {}) == 2
```

Read `test_label_skips_one_session_and_sums_h_excess_returns` with a pencil. Day 0's label is the excess returns of days 2, 3 and 4: `(0.02 − 0.01) + (0.03 − 0.01) + (0.04 − 0.01) = 0.06`, and its label end is day 4.

```powershell
python -m pytest -m batch03 -q
```

```text
........                                                                 [100%]
8 passed
```

## Checkpoint

- [ ] You can state why the label starts at `t+2` and what `label_end` is used for.
- [ ] You can explain purging and embargo with the diagram above.
- [ ] `python -m pytest -m batch03 -q` → 8 passed.

```powershell
git add .
git commit -m "Validation framework: labels, purged walk-forward folds, pinball, DM, net-of-cost, DSR, trial registry"
```

Next: [Part 07 — Stage 02 forecasting](07_stage02_forecasting.md).
