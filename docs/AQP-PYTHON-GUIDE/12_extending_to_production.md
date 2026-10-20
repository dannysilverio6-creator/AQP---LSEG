# Part 12 — Extending the spine toward production

**Goal:** know exactly where each remaining piece of the design plugs in, how to add it without breaking what works, and what has to be true before real money is involved.

You have finished roadmap phases 1–6 for one sleeve, and phase 7 can begin (paper trading on real data). This part covers the rest.

| Roadmap phase (design §17) | Status after Part 11 | Remaining work |
|---|---|---|
| 1 Data foundation | done for CSV data | LSEG source (12.9), macro vintages, index membership history |
| 2 Baseline research | done | more benchmarks (factor model) |
| 3 Validation | done | stability checks across folds (12.6), PBO, MCS/SPA |
| 4 Forecast layers | financial layer (simple) | GARCH (12.2), macro (12.3), micro (12.4), news (12.5) |
| 5 Risk/optimisation | done for one sleeve | factor/DCC covariance (12.8), sleeve allocator (12.7) |
| 6 Execution simulation | done | scheduling, cancel/replace rules |
| 7 Paper trading | loop works on synthetic dates | real data, scheduled runs, latency/restart drills (12.10) |
| 8 Controlled deployment | **not started** | written sign-off, live broker adapter, incident response (12.11) |
| 9 Challenger lifecycle | G7 gate (12.9) | shadow runner, release and rollback tooling |

---

## 12.1 The extension pattern

Every addition follows the same six steps. If a step is skipped, the addition is not done.

1. **Contract first.** If the addition produces something new, define its shape in `contracts/` (for example `EventRecord` in 12.5).
2. **Pure module.** Put the method in the module the repository overview names, with no file I/O and no provider calls inside.
3. **Configuration.** Every threshold or parameter goes in YAML, and the validator learns about any new required key.
4. **Hook.** Wire it in at one place: the model registry, the stage service, or the source.
5. **Tests.** Write a unit test with hand-checkable numbers, a failure-path test, and (if it touches time) a look-ahead test.
6. **Checkpoint.** Run the stage checkpoint and a full run, and compare the manifest and report with the previous run.

New packages need their folders and `__init__.py` files first. Run this once before typing the modules in this part:

```powershell
$new = @("src\aqp_python\stages\stage_02_forecasting\macro", "src\aqp_python\stages\stage_02_forecasting\micro_market",
         "src\aqp_python\stages\stage_02_forecasting\micro_market\intraday", "src\aqp_python\stages\stage_02_forecasting\news",
         "r_models", "r_models\adapters")
foreach ($d in $new) {
    New-Item -ItemType Directory -Force -Path $d | Out-Null
    if (-not (Test-Path "$d\__init__.py")) { New-Item -ItemType File -Path "$d\__init__.py" | Out-Null }
}
New-Item -ItemType Directory -Force -Path "r_models\scripts" | Out-Null
```

The extension modules below were written and tested with the reference implementation (`tests\unit\test_extensions.py`, marker `batch08`). They are **not wired into the default run**, so the core results from Part 11 do not change.

## 12.2 Financial layer: GJR-GARCH volatility

Design §6 names GJR-GARCH with Student-t innovations as the daily volatility model, and EWMA as its baseline. Here is the estimator:

**File:** `src/aqp_python/stages/stage_02_forecasting/financial/volatility_garch.py`

```python
"""GJR-GARCH(1,1) with Student-t innovations (design §6), as an h-day volatility forecast.

sigma_t^2 = omega + (alpha + gamma * 1{r_{t-1} < 0}) r_{t-1}^2 + beta sigma_{t-1}^2

The EWMA volatility used by the core models is the design's baseline/control; this
estimator must beat it out of sample before it replaces it. Requires `pip install arch`.
"""
import math

import pandas as pd

MIN_OBSERVATIONS = 500


def gjr_garch_forecast(daily_returns: pd.Series, horizon: int) -> dict:
    """Fit on returns up to the last row only, and forecast the next `horizon` days' volatility."""
    from arch import arch_model  # optional dependency, imported on use

    clean = daily_returns.dropna()
    if len(clean) < MIN_OBSERVATIONS:
        raise ValueError(f"need {MIN_OBSERVATIONS} returns, got {len(clean)}")
    model = arch_model(clean * 100, mean="Zero", vol="GARCH", p=1, o=1, q=1, dist="t")
    fit = model.fit(disp="off")
    if fit.convergence_flag != 0:
        raise ValueError("GJR-GARCH did not converge; record a model failure, never a forecast")
    variance = fit.forecast(horizon=horizon, reindex=False).variance.iloc[-1].sum() / 100 ** 2
    params = fit.params.to_dict()
    return {"sigma_h": math.sqrt(variance), "gamma": params["gamma[1]"], "nu": params["nu"],
            "persistence": params["alpha[1]"] + params["beta[1]"] + params["gamma[1]"] / 2}
```

```powershell
python -m pip install arch
```

**How to wire it in.**

1. Add a point-in-time feature `garch_vol` in `financial_features.py`: for each security, refit monthly on returns *up to* the refit date and filter forward with the fixed parameters until the next refit. Never fit on the full sample and read off "historical" values, because that uses future data.
2. Add a model class (for example `GarchLocationScale`) that uses `garch_vol` instead of `ewma_vol` as its scale. Register it in `forecast_registry.py` and add it to `forecast_models.yaml` with `role: candidate`.
3. G3 compares it with the baselines and the existing candidate. If it does not beat them out of sample, it gets no weight, and EWMA stays.

**HAR-RV** (intraday realised volatility) follows the same pattern, but needs the intraday data of 12.4. **Distress hazard** (design §6) is a discrete-time logit on firm-year rows. Its output feeds `distress_probability` in the reference dataset, which Stages 01 and 03 already use. **Dynamic exposures** must use the *filtered* Kalman estimate, never the smoothed one.

## 12.3 Macroeconomic layer

Design §7: combine economy-level states with each firm's **dated revenue and cost exposures**, kept separate.

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/company_macro_exposure.py`

```python
"""Exposure-weighted macro factors (design §7): revenue and cost exposures are kept separate.

f_rev_i = sum_c e_rev_ic f_c        f_cost_i = sum_c e_cost_ic f_c
"""
import pandas as pd


def exposure_weighted_factor(exposures: pd.DataFrame, factor_by_economy: pd.Series) -> pd.Series:
    """exposures: securities x economies (dated shares from the Company Reference dataset).

    Unknown exposures stay unknown (NaN) - they are never silently treated as zero.
    """
    missing = sorted(set(exposures.columns) - set(factor_by_economy.index))
    if missing:
        raise ValueError(f"no factor value for economies {missing}")
    weighted = exposures.mul(factor_by_economy[exposures.columns], axis=1)
    return weighted.sum(axis=1, min_count=len(exposures.columns))
```

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/macro_sensitivity.py`

```python
"""Shrink noisy firm sensitivities toward priors (design §7).

b_post = lambda * b_ols + (1 - lambda) * b_prior,   lambda = tau^2 / (tau^2 + se^2)
"""
import pandas as pd


def shrink_sensitivities(b_ols: pd.Series, standard_error: pd.Series, b_prior: pd.Series,
                         tau: float) -> pd.Series:
    if tau <= 0:
        raise ValueError("tau must be positive")
    weight = tau ** 2 / (tau ** 2 + standard_error ** 2)
    return weight * b_ols + (1 - weight) * b_prior
```

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/release_surprise.py`

```python
"""Standardised release surprise (design §7): S = (actual - consensus) / sd(past surprises)."""
import pandas as pd


def standardised_surprise(actual: float, consensus: float, past_surprises: pd.Series,
                          min_history: int = 8) -> float:
    """Only surprises from releases BEFORE this one may be in `past_surprises`."""
    history = past_surprises.dropna()
    if len(history) < min_history or history.std() == 0:
        raise ValueError("not enough surprise history to standardise")
    return float((actual - consensus) / history.std())
```

**How to wire it in.**

1. **Data (Stage 00):** add a macro source that stores every release with its **official release timestamp** and every revision as a new vintage, the same `availability_ts` discipline as prices. Query with `latest_vintage(keys=["series", "period"])`.
2. **States:** start with something simple and testable (for example a 3-month change in each economy's activity index) before a dynamic factor model or Markov-switching regimes. Use filtered states only.
3. **Features:** for each security and date, `exposure_weighted_factor(revenue_shares_as_of_date, factor_as_of_date)`. The revenue shares come from the reference data you already have (`revenue_share_au/us/cn`).
4. Add the new feature names to `forecast_models.yaml → features`. The ridge model uses them with no code change, and G3 decides whether they help.

## 12.4 Firm and microstructure layer (intraday)

**File:** `src/aqp_python/stages/stage_02_forecasting/micro_market/intraday/realised_measures.py`

```python
"""Realised measures from one session of intraday log returns (design §8)."""
import numpy as np


def realised_variance(r: np.ndarray) -> float:
    return float(np.sum(r ** 2))


def bipower_variation(r: np.ndarray) -> float:
    """(pi/2) sum |r_i||r_{i-1}|: robust to a few large jumps."""
    return float(np.pi / 2 * np.sum(np.abs(r[1:]) * np.abs(r[:-1])))


def jump_component(r: np.ndarray) -> float:
    return max(realised_variance(r) - bipower_variation(r), 0.0)


def semivariances(r: np.ndarray) -> tuple[float, float]:
    """(downside, upside) realised semivariance."""
    return float(np.sum(r[r < 0] ** 2)), float(np.sum(r[r > 0] ** 2))
```

Bipower variation is robust to jumps, so `RV − BV` estimates the jump part. That is what the unit test checks with one artificial jump.

**How to wire it in.** Intraday data is expensive, so the design's **data tiers** decide who gets it: T1 (about 40 names per market) gets trades and quotes, T2 gets one-minute bars, and T3 stays daily. Assign tiers at universe formation (add a `data_tier` column to `formation_decisions.csv`) and keep them fixed within the version. Daily realised measures become features for the interday sleeve, and execution-cost estimates for the intraday sleeves. Use them only where timestamps and sessions are reliable; otherwise fall back to the daily estimates (design §8).

## 12.5 News and event layer

The contract an LLM extractor must fill:

**File:** `src/aqp_python/contracts/event_schema.py`

```python
"""Structured news event (design §9). An LLM may fill these fields; it never trades."""
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class EventRecord:
    event_id: str
    security_id: str
    source: str
    published_at: datetime          # publication time stated by the source
    processed_at: datetime          # when our system finished extracting it
    event_type: str
    direction: int                  # -1, 0, +1
    magnitude: float                # model-specific scale, >= 0
    confidence: float               # 0..1
    novelty: float                  # 0..1, 1 = new information
    evidence: str                   # supporting text span or reference, for audit
    prompt_version: str
    model_version: str
```

**File:** `src/aqp_python/stages/stage_02_forecasting/news/event_schema_validator.py`

```python
"""Reject events that are malformed or were not usable at the decision time."""
from datetime import datetime

from aqp_python.contracts.event_schema import EventRecord


def validate_event(event: EventRecord, decision_time: datetime) -> list[str]:
    """Return a list of problems; an empty list means the event may be used."""
    problems = []
    if event.published_at.tzinfo is None or event.processed_at.tzinfo is None:
        problems.append("timestamps must be timezone-aware")
        return problems
    if event.processed_at < event.published_at:
        problems.append("processed before it was published")
    if event.processed_at > decision_time:
        problems.append("not processed by the decision time (look-ahead)")
    if event.direction not in (-1, 0, 1):
        problems.append("direction must be -1, 0 or +1")
    if not 0 <= event.confidence <= 1 or not 0 <= event.novelty <= 1:
        problems.append("confidence and novelty must be in [0, 1]")
    if not event.evidence.strip():
        problems.append("missing evidence span")
    return problems
```

**File:** `src/aqp_python/stages/stage_02_forecasting/news/graph_propagation.py`

```python
"""Propagate an event shock through the relationship graph with decay (design §9).

effect = sum_{k=1..K} (delta W)^k shock.  The series converges only if the spectral
radius of delta W is below one, so that is checked before anything is propagated.
"""
import numpy as np


def propagate(weights: np.ndarray, shock: np.ndarray, decay: float, order: int = 3) -> np.ndarray:
    operator = decay * np.asarray(weights, dtype=float)
    radius = max(abs(np.linalg.eigvals(operator)))
    if radius >= 1:
        raise ValueError(f"unstable propagation: spectral radius {radius:.3f} >= 1")
    effect, term = np.zeros_like(shock, dtype=float), np.asarray(shock, dtype=float)
    for _ in range(order):
        term = operator @ term
        effect += term
    return effect
```

Design §9 rules that this code enforces or that you must keep:

- An event counts at the decision time only if it was **processed** by then (`processed_at ≤ decision_time`), not merely published.
- The LLM **extracts fields**. It never trades and never declares an event profitable. A calibrated statistical model (for example a logit on novelty, direction, magnitude and propagated effect) produces the event probability, and it is validated with Brier scores and reliability curves.
- **Freeze** the prompt and model version during evaluation (`prompt_version`, `model_version` on every record), and keep a prompt registry.
- Propagation through the relationship graph is refused unless the spectral radius of `δW` is below one.

## 12.6 Validation: what to add next

- **Stability across folds** (the G3 criterion not yet implemented): record each fold's ridge coefficients in `out_of_sample_predictions`. Then add a `sign_stability` metric (the share of folds where each coefficient has its full-sample sign) to `score_models`, and a `min_sign_stability` threshold to `gates.yaml` and G3.
- **PBO, SPA and MCS** (design §10): useful once there are many candidates. Add them to `overfitting_controls.py` and `forecast_comparison_tests.py` with their own unit tests.
- **Conformal calibration** of the combined quantiles on held-out data, and **horizon selection**, which chooses the horizon with the best net information ratio.

## 12.7 More sleeves

To add `us_interday`:

1. `configs\sleeves\sleeve_us_interday.yaml`, copied from the ASX file. Change `market: US`, `currency: USD`, costs, limits and stress scenarios (add an AUD/USD shock and an overnight-gap scenario).
2. One line in `pipeline.yaml → sleeves`.
3. A US data source and calendar (Stage 00), and a USD→AUD translation for NAV.
4. **Cross-market alignment** (design §11): ASX and US sessions do not overlap, so use multi-day returns or lead–lag terms when estimating cross-market covariance.
5. A **sleeve allocator** above the per-sleeve optimisers that splits capital by risk budget, with a hard cap on intraday risk, before Stage 05.

Run it with `--sleeve us_interday`. Each sleeve keeps its own paper book, universe version and run folders.

## 12.8 The R boundary

Design §15 of the overview: R models are called only through an adapter with a documented input/output contract, a timeout, and failures recorded as model failures.

**File:** `r_models/adapters/r_model_adapter.py`

```python
"""Call an R script with JSON in and JSON out. R is never imported into Python.

    Rscript <script.R> <input.json> <output.json>

A non-zero exit, a timeout, missing output or missing keys raise RModelError.
The caller records that as a model failure - never as a forecast.
"""
import json
import subprocess
import tempfile
from pathlib import Path


class RModelError(RuntimeError):
    pass


def run_r_model(script: Path, payload: dict, required_keys: list[str], *,
                rscript: str = "Rscript", timeout_seconds: int = 120) -> dict:
    with tempfile.TemporaryDirectory() as folder:
        source, target = Path(folder) / "input.json", Path(folder) / "output.json"
        source.write_text(json.dumps(payload), encoding="utf-8")
        try:
            completed = subprocess.run([rscript, str(script), str(source), str(target)],
                                       capture_output=True, text=True, timeout=timeout_seconds, check=False)
        except (OSError, subprocess.TimeoutExpired) as error:
            raise RModelError(f"could not run {script.name}: {error}") from error
        if completed.returncode != 0:
            raise RModelError(f"{script.name} exited {completed.returncode}: {completed.stderr[-500:]}")
        if not target.is_file():
            raise RModelError(f"{script.name} wrote no output")
        result = json.loads(target.read_text(encoding="utf-8"))
    missing = [key for key in required_keys if key not in result]
    if missing:
        raise RModelError(f"{script.name} output is missing {missing}")
    return result
```

A template R script for DCC-GARCH (it needs R with `jsonlite` and `rmgarch`; the Python adapter's contract is tested with a stand-in script):

**File:** `r_models/scripts/dcc_garch.R`

```text
# DCC-GARCH correlation forecast (design §11). TEMPLATE: needs R with jsonlite and rmgarch.
# Called by r_models/adapters/r_model_adapter.py:  Rscript dcc_garch.R input.json output.json
# input.json : {"returns": [[...], ...] (T x N, decimal), "names": [...], "horizon": 5}
# output.json: {"covariance": [[...]] (N x N, h-day), "converged": true}
suppressPackageStartupMessages({library(jsonlite); library(rmgarch)})
args <- commandArgs(trailingOnly = TRUE)
input <- fromJSON(args[1])
returns <- as.matrix(input$returns)
uspec <- multispec(replicate(ncol(returns), ugarchspec(
  mean.model = list(armaOrder = c(0, 0)), variance.model = list(model = "gjrGARCH"),
  distribution.model = "std")))
spec <- dccspec(uspec, dccOrder = c(1, 1), distribution = "mvt")
fit <- dccfit(spec, data = returns)
forecast <- dccforecast(fit, n.ahead = input$horizon)
cov_path <- rcov(forecast)[[1]]
h_day <- apply(cov_path, c(1, 2), sum)
write_json(list(covariance = h_day, converged = fit@mfit$convergence == 0), args[2],
           digits = NA, auto_unbox = TRUE)
```

**How to wire it in as a covariance engine with an approved fallback.** This is also where `DEGRADED` (Part 02) is used for the first time:

```python
# in stages_03_04_portfolio/service.py (sketch)
try:
    result_json = run_r_model(DCC_SCRIPT, {"returns": daily.to_numpy().tolist(), "horizon": horizon},
                              ["covariance", "converged"])
    covariance = pd.DataFrame(result_json["covariance"], index=ids, columns=ids)
except RModelError as error:
    covariance = ledoit_wolf_covariance(daily, horizon)                  # approved fallback
    result.degrade(RC.MODEL_FIT_FAILED, f"DCC unavailable, Ledoit-Wolf fallback: {error}")
```

List the fallback in a `configs\fallbacks.yaml` and test that path. Design §2: a fallback must be approved and tested, and may never bypass data integrity, margin, hard limits, kill switches or reconciliation.

## 12.9 Model lifecycle and G7

**File:** `src/aqp_python/gates/g7_model_promotion.py`

```python
"""G7 - model promotion. A challenger never replaces the approved model automatically."""
from collections.abc import Mapping

from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G7"


def check_promotion(challenger: str, evidence: Mapping, rules: Mapping) -> GateResult:
    """All conditions in design §13 must hold; otherwise keep the current approved model."""
    failures = []
    if not evidence.get("g3_eligible"):
        failures.append("NOT_G3_ELIGIBLE")
    if evidence.get("shadow_sessions", 0) < rules["min_shadow_sessions"]:
        failures.append("SHADOW_TOO_SHORT")
    if not evidence.get("calibration_intact"):
        failures.append("CALIBRATION_BROKEN")
    if not evidence.get("dm_pvalue_vs_champion", 1.0) <= rules["max_dm_pvalue_vs_champion"]:
        failures.append("NOT_BETTER_THAN_CHAMPION")
    if not evidence.get("rollback_tested"):
        failures.append("NO_TESTED_ROLLBACK")
    if not evidence.get("independent_approval"):
        failures.append("NO_INDEPENDENT_APPROVAL")
    if failures:
        return GateResult(GATE, GateOutcome.BLOCK, challenger, tuple(failures),
                          "keep the current approved model", dict(evidence))
    return GateResult(GATE, GateOutcome.PASS, challenger, (), "approved for versioned release", dict(evidence))
```

**File:** `configs/model_lifecycle.yaml`

```yaml
# G7 promotion policy (design §13). Used by model-lifecycle tooling, never by a trading run.
min_shadow_sessions: 60
max_dm_pvalue_vs_champion: 0.05
```

The lifecycle in design §13: `candidate → G3 eligible → shadow → G7 approved → retired`. To build it:

1. **Shadow role.** Add `role: shadow` to `forecast_models.yaml`. Shadow models are forecast, validated and logged every run, but `signal_combiner` gives them weight 0, and they have no path to orders.
2. **Evidence.** Each run already writes `oos_forecasts.csv` and the live forecasts. A `model_lifecycle/shadow_runner.py` collects these across runs, and `promotion_review.py` builds the evidence dictionary for `check_promotion`.
3. **Release.** Promotion means a reviewed commit that changes `role: shadow` to `role: candidate` (and retires the old model), tagged in Git. **Rollback** is the reverse commit, and it must be tested before promotion.
4. `model_lifecycle` must never import the broker or Stage 06. Add that rule to `test_hard_limits_independence.py` when the package is created.

## 12.10 Paper trading on real data

Phase 7's exit criterion: *"Reconciliation, stale-data, restart, latency and kill-switch tests pass."* With a real data source:

1. **Schedule** a daily paper run after the data is published. Create `scripts\run_paper_today.ps1`:

```powershell
Set-Location (Join-Path $PSScriptRoot "..")
$today = Get-Date -Format "yyyy-MM-dd"
& .\.venv\Scripts\python.exe -m aqp_python.cli run --as-of $today --mode paper *>> .\outputs\scheduler.log
exit $LASTEXITCODE
```

   and register it once (17:15 local time, weekdays):

```powershell
$script = (Resolve-Path .\scripts\run_paper_today.ps1).Path
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$script`""   # Bypass applies to this one process only
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday,Tuesday,Wednesday,Thursday,Friday -At 5:15PM
Register-ScheduledTask -TaskName "AQP paper asx_interday" -Action $action -Trigger $trigger
```

   The synthetic data ends on 2026-10-16, so scheduled runs only make sense with a live source. Until then they will block at Stage 00 (`DATA_STALE`), which is the correct behaviour.

2. **Drills** (record date, result and reviewer in `docs\readiness\`): stale data (stop the source; expect BLOCKED); restart mid-run (kill the process during Stage 02; the next run must start cleanly, with the book unchanged); kill switch; a reconciliation break (edit `paper_book.json` by hand; expect `RECONCILIATION_FAILED`, the kill switch engaged and an incident logged); latency (time from data publication to orders, from `run.log`).

3. **Monitor for weeks, not days.** Watch coverage alerts, drawdown, implementation shortfall versus the cost model, and whether the live G3 evidence still holds.

## 12.11 Controlled deployment (phase 8)

This is a sign-off process, not a test batch. Do not write a live broker adapter until all of the following exist and have been reviewed by someone other than the author:

- [ ] Written decisions on all seven open design questions (below), with the limits in YAML.
- [ ] At least the agreed paper-trading period with reconciliation clean every day.
- [ ] A live adapter implementing `connectors/broker/broker_interface.py`, used only by Stage 06, with broker-side limits configured as a second, independent safety net.
- [ ] A tested rollback (switch back to paper mode) and an incident-response runbook.
- [ ] A small, approved initial allocation.

Keep a decision record in the repository. `docs\decisions\open_decisions.md`:

```markdown
| # | Decision (design §15) | Options | Current setting | Config key | Owner | Status |
|---|---|---|---|---|---|---|
| 1 | Instruments and capital structure | cash equities on margin / CFDs / futures | cash equities, no margin | sleeve.sizing.initial_margin_rate | | open |
| 2 | Per-sleeve vol target, drawdown, name/sector/gross limits, margin buffer | | 12% / 15% / 8% / 30% / 1.0 / 20% | sleeve.*, risk_limits.* | | provisional |
| 3 | US broker and data provider; intraday coverage | | none | sleeves, connectors | | open |
| 4 | News sources and latency; directional intraday use | | not used | forecast_models | | open |
| 5 | Long-only vs long-short; ASX borrow | | long-only | sleeve.portfolio.long_only | | provisional |
| 6 | Universe re-formation: deliberate or scheduled | | deliberate (new version_label) | universe.version_label | | provisional |
| 7 | Fixed model families or open challenger process | | open, governed by G3/G7 | model_lifecycle.yaml | | provisional |
```

and `docs\decisions\decision_log.md`, one dated entry per change of any limit, code or model: *what changed, why, evidence, who approved it, and the commit*.

## 12.12 Tests for the extensions

**File:** `tests/unit/test_extensions.py`

```python
"""Tests for the Part 12 extension modules. They are not wired into the default run."""
import sys
from datetime import UTC, datetime, timedelta

import numpy as np
import pandas as pd
import pytest

from aqp_python.contracts.event_schema import EventRecord
from aqp_python.gates.g7_model_promotion import check_promotion
from aqp_python.stages.stage_02_forecasting.macro.company_macro_exposure import exposure_weighted_factor
from aqp_python.stages.stage_02_forecasting.macro.macro_sensitivity import shrink_sensitivities
from aqp_python.stages.stage_02_forecasting.macro.release_surprise import standardised_surprise
from aqp_python.stages.stage_02_forecasting.micro_market.intraday.realised_measures import (
    bipower_variation,
    jump_component,
    realised_variance,
)
from aqp_python.stages.stage_02_forecasting.news.event_schema_validator import validate_event
from aqp_python.stages.stage_02_forecasting.news.graph_propagation import propagate
from r_models.adapters.r_model_adapter import RModelError, run_r_model

pytestmark = pytest.mark.batch08


def test_revenue_exposure_weighting_keeps_unknowns_unknown():
    exposures = pd.DataFrame({"AU": [0.6, np.nan], "CN": [0.4, 0.5]}, index=["A", "B"])
    out = exposure_weighted_factor(exposures, pd.Series({"AU": 1.0, "CN": -2.0}))
    assert out["A"] == pytest.approx(-0.2) and np.isnan(out["B"])


def test_noisy_sensitivities_shrink_toward_the_prior():
    b = shrink_sensitivities(pd.Series([2.0, 2.0]), pd.Series([0.1, 10.0]), pd.Series([0.0, 0.0]), tau=1.0)
    assert b.iloc[0] > 1.9 and b.iloc[1] < 0.05


def test_surprise_is_standardised_by_past_surprises_only():
    assert standardised_surprise(3.0, 2.0, pd.Series([0.5, -0.5] * 4)) == pytest.approx(1 / 0.5345, rel=1e-3)


def test_bipower_variation_ignores_a_single_jump():
    r = np.random.default_rng(0).normal(0, 0.001, 390)
    r[200] = 0.03
    assert realised_variance(r) > 2 * bipower_variation(r) and jump_component(r) > 0.0008


def test_propagation_refuses_an_unstable_graph():
    w = np.array([[0.0, 1.0], [1.0, 0.0]])
    assert propagate(w, np.array([1.0, 0.0]), decay=0.5, order=2)[1] == pytest.approx(0.5)
    with pytest.raises(ValueError, match="spectral radius"):
        propagate(w, np.array([1.0, 0.0]), decay=1.2)


def test_events_processed_after_the_decision_are_rejected():
    t = datetime(2026, 10, 16, 6, tzinfo=UTC)
    event = EventRecord("e1", "SYN01", "wire", t - timedelta(hours=2), t + timedelta(minutes=5), "guidance",
                        1, 0.5, 0.8, 0.9, "raises FY guidance", "p1", "m1")
    assert any("look-ahead" in p for p in validate_event(event, t))


def test_g7_requires_every_condition():
    rules = {"min_shadow_sessions": 60, "max_dm_pvalue_vs_champion": 0.05}
    good = {"g3_eligible": True, "shadow_sessions": 61, "calibration_intact": True,
            "dm_pvalue_vs_champion": 0.01, "rollback_tested": True, "independent_approval": True}
    assert check_promotion("c", good, rules).passed
    result = check_promotion("c", {**good, "shadow_sessions": 20, "independent_approval": False}, rules)
    assert set(result.reason_codes) == {"SHADOW_TOO_SHORT", "NO_INDEPENDENT_APPROVAL"}


def test_garch_reacts_to_recent_volatility():
    pytest.importorskip("arch")
    from aqp_python.stages.stage_02_forecasting.financial.volatility_garch import gjr_garch_forecast
    rng = np.random.default_rng(1)
    calm = rng.standard_t(6, 900) * 0.008
    stormy = np.concatenate([calm, rng.standard_t(6, 30) * 0.03])
    low, high = (gjr_garch_forecast(pd.Series(x), horizon=5)["sigma_h"] for x in (calm, stormy))
    assert high > 1.5 * low


def test_r_adapter_contract_with_a_stand_in_script(tmp_path):
    script = tmp_path / "fake_model.py"        # stands in for an R script: same argv contract
    script.write_text("import json, sys\n"
                      "data = json.load(open(sys.argv[1]))\n"
                      "json.dump({'covariance': [[1.0]], 'n': len(data['returns'])}, open(sys.argv[2], 'w'))\n")
    result = run_r_model(script, {"returns": [[0.1], [0.2]]}, ["covariance"], rscript=sys.executable)
    assert result["n"] == 2
    with pytest.raises(RModelError, match="missing"):
        run_r_model(script, {"returns": []}, ["converged"], rscript=sys.executable)
    script.write_text("import sys\nsys.exit(3)\n")
    with pytest.raises(RModelError, match="exited 3"):
        run_r_model(script, {}, [], rscript=sys.executable)
```

```powershell
python -m pytest -m batch08 -q          # 9 passed (the GARCH test is skipped if arch is not installed)
```

---

**The rule to keep:** research models propose forecasts; the validated pipeline proposes target positions; only independent risk and execution controls authorise orders. Every extension should leave that sentence true, and the invariants in `tests\invariants\` are there to keep it so.
