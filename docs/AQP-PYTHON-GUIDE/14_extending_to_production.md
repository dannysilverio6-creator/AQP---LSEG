# Part 14 — Extending the spine toward production

**Goal:** know exactly where each remaining piece of the design plugs in, add it without breaking what works, and know what has to be true before real money is involved. In v14 that includes three things the v12 guide did not have: a **language-model extractor** behind the news engine's protocol, **calibrated event risk** and **graph propagation** as further Z_news candidates, and the **promotion path** by which a shadow-candidate news challenger may, after review, become the approved arm.

| Roadmap phase (v14) | Status after Part 13 | Remaining work |
|---|---|---|
| 1 Contracts, configuration, storage | done | — |
| 2 Screening and universe (filter first) | done for CSV data | licensed screening source, index membership history |
| 3 Detailed data expansion | done for CSV data | licensed detailed source, macro vintages (14.3), intraday tiers (14.4) |
| 4 No-news baselines | done | GARCH volatility (14.2), more benchmarks |
| 5 News feature engine | lexicon extractor | LLM extractor (14.5), event risk and propagation (14.6) |
| 6 Paired evaluation | done | fold-stability checks, PBO, SPA/MCS (14.8) |
| 7 Portfolio and sizing | done for one sleeve | DCC covariance through R (14.7), more sleeves (14.9) |
| 8 Execution, safety, monitoring | done in paper mode | scheduling and operational drills (14.10) |
| 9 Promotion and deployment | G7 gate | shadow tracking, promotion review (14.11), sign-off (14.12) |

---

## 14.1 The extension pattern

Every addition follows the same six steps. If a step is skipped, the addition is not done.

1. **Contract first.** If the addition produces something new, define its shape in `contracts\` (as `NewsFeatures` was defined in Part 08 before the engine in Part 09).
2. **Pure module.** Put the method in the module the repository overview names, with no file I/O and no provider calls inside.
3. **Configuration.** Every threshold or parameter goes in YAML, and the validator learns about any new required key.
4. **Hook.** Wire it in at one place: the model registry, a stage service, the extractor protocol, or the connector.
5. **Tests.** A unit test with hand-checkable numbers, a failure-path test, and (if it touches time) a look-ahead test. Mark them `extension` until they are wired into the default run.
6. **Evidence.** A new **forecasting** input is a new experiment: register it (Part 10) and let G3 decide. A new **news** feature changes the news version and therefore needs a new experiment id.

The modules below already exist as empty files from the repository shell (Part 01). They are tested (`python -m pytest -m extension`) but **not wired into the default run**, so the results from Part 13 do not change.

## 14.2 Financial layer: GJR-GARCH volatility

Design 2A names GJR-GARCH with Student-t innovations as the daily volatility model, and EWMA as its baseline:

**File:** `src/aqp_python/stages/stage_02_forecasting/financial/volatility_garch.py`

```python
"""GJR-GARCH(1,1) with Student-t innovations (design 2A), as an h-day volatility forecast.

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

**What it does.** Fits the model on returns up to the last row only, refuses short histories, and treats non-convergence as a model **failure**, never as a forecast. `gamma` measures the leverage effect: negative returns raise future volatility more than positive ones. `arch` is imported inside the function, so the package works without it.

**How to wire it in.**

1. Add a point-in-time feature `garch_vol` to `core_feature_builder.py`: for each security, refit monthly on returns *up to* the refit date and filter forward with fixed parameters until the next refit. Never fit on the full sample and read off "historical" values; that uses future data.
2. Add a model class that uses `garch_vol` instead of `ewma_vol` as its scale, register it, and add it to `forecast_models.yaml` with `role: candidate`.
3. Register a new experiment for it (it is a candidate, so it needs one) and let G3 decide. If it does not beat the EWMA-scaled ridge out of sample, EWMA stays.

## 14.3 Macroeconomic layer

Design 2B: combine economy-level states with each firm's **dated revenue and cost exposures**, kept separate. The exposures come from the Company Reference and Exposure Dataset built in Part 06.

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/company_macro_exposure.py`

```python
"""Exposure-weighted macro factors (design 2B): revenue and cost exposures are kept separate.

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

**What it does.** `f_i = Σ_c e_ic f_c`. `min_count` makes the sum NaN if any exposure is unknown: the same rule as missing news. An unknown exposure is not a zero exposure.

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/macro_sensitivity.py`

```python
"""Shrink noisy firm sensitivities toward priors (design 2B).

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

**What it does.** Bayesian shrinkage: a sensitivity estimated with a large standard error is pulled towards its prior, one estimated precisely keeps its own value.

**File:** `src/aqp_python/stages/stage_02_forecasting/macro/release_surprise.py`

```python
"""Standardised release surprise (design 2B): S = (actual - consensus) / sd(past surprises)."""
import pandas as pd


def standardised_surprise(actual: float, consensus: float, past_surprises: pd.Series,
                          min_history: int = 8) -> float:
    """Only surprises from releases BEFORE this one may be in `past_surprises`."""
    history = past_surprises.dropna()
    if len(history) < min_history or history.std() == 0:
        raise ValueError("not enough surprise history to standardise")
    return float((actual - consensus) / history.std())
```

**How to wire it in.** Store every macro release with its **official release timestamp**, and every revision as a new vintage (the vintage store from Part 03), and request it through the data request in Part 06. Build the exposure-weighted factors as new X_core features (they are not news, so they go in X_core and both arms see them). That changes the core feature version, so register a new experiment.

## 14.4 Firm and microstructure layer (intraday)

**File:** `src/aqp_python/stages/stage_02_forecasting/firm_microstructure/intraday/realised_measures.py`

```python
"""Realised measures from one session of intraday log returns (design 2C)."""
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

Bipower variation is robust to jumps, so `RV − BV` estimates the jump part; the unit test checks it with one artificial jump.

**How to wire it in.** Intraday data is expensive, which is why v14 assigns **coverage tiers** at universe formation (Part 05): T1 names may request trades and quotes, T2 one-minute bars, T3 daily data only. `coverage_tiers.yaml` already lists the sources each tier wants, and `financial_extractor.py` already reports `trades_quotes` and `minute_bars` as unavailable from the sample provider. When a provider supplies them, the data request (Part 06) will ask for them only for the tiers that want them, and daily realised measures can be computed for those names. Names without intraday data fall back to daily estimates (design 2C).

## 14.5 A language-model extractor behind the same protocol

Part 09 built the news engine around the `EventExtractor` protocol and a deterministic lexicon. The design's language-model extractor plugs into the same place.

**File:** `src/aqp_python/connectors/llm/llm_client.py`

```python
"""The ONLY place a language-model provider may be called (Part 14). Used only by Stage 02D.

Which provider, model and data-retention terms are open decision 4 (docs/decisions). Until it is
decided, `provider_client` refuses to run, so nothing can call a model by accident.
"""
import os
from typing import Protocol


class CompletionClient(Protocol):
    def complete(self, prompt: str) -> str: ...


def provider_client(api_key_env: str = "LLM_API_KEY") -> CompletionClient:
    if not os.environ.get(api_key_env):
        raise RuntimeError(f"{api_key_env} is not set; LLM extraction is disabled")
    raise NotImplementedError("choose and approve an LLM provider first (open decision 4)")
```

**What it does.** The only place in the package allowed to call a model provider. `CompletionClient` is the protocol: one method, text in, text out. `provider_client` **refuses to run** until open decision 4 (which provider, which model, which data-retention terms) has been made, so nothing can call a model by accident.

**File:** `src/aqp_python/stages/stage_02d_news_features/llm_event_extractor.py` — component 3 of 3: LLMExtractor: the same protocol, filled by a language model (Part 14)

```python
EMPTY_EVENT = {"event_type": None, "direction": 0, "severity": 0.0, "confidence": 0.0, "evidence": ""}


class LLMExtractor:
    """The model fills the fixed schema; it never trades. Malformed replies become empty events,
    which the evidence validator rejects - they are never repaired by guesswork."""

    def __init__(self, prompt: PromptSpec, client) -> None:     # client: connectors.llm CompletionClient
        self.prompt, self._client = prompt, client

    def extract(self, headline: str, body: str) -> dict:
        import json
        text = f"{self.prompt.instructions}\n\nHEADLINE: {headline}\nBODY: {body}\nJSON:"
        try:
            reply = json.loads(self._client.complete(text))
        except (json.JSONDecodeError, TypeError, ValueError):
            return dict(EMPTY_EVENT)
        if not isinstance(reply, dict):
            return dict(EMPTY_EVENT)
        return {key: reply.get(key, EMPTY_EVENT[key]) for key in EMPTY_EVENT}
```

**What it does.** Sends the **registered** prompt (`llm-v1` from Part 09's prompt registry), parses the reply as JSON and keeps exactly the five schema fields. A reply that is not JSON, or not an object, becomes an empty event, which the evidence validator rejects. It never repairs a reply by guessing. **Why it lives in the stage, not the connector:** the connector knows about providers; the stage knows about events. The import-linter contracts keep the connector from importing any stage.

Try it with a fake client, so no model is called:

```python
import pandas as pd

from aqp_python.stages.stage_02d_news_features.evidence_validator import validate_events
from aqp_python.stages.stage_02d_news_features.llm_event_extractor import LLMExtractor, extract_events
from aqp_python.stages.stage_02d_news_features.prompt_registry import EVENT_TYPES, get_prompt


class FakeClient:
    """Returns a fixed reply, as a model might."""
    def __init__(self, reply):
        self.reply = reply

    def complete(self, prompt):
        return self.reply


story = pd.DataFrame({"headline": ["Synthetic Company 01 raises full-year guidance"], "body": [""],
                      "status": ["OK"]})
replies = {
    "faithful": '{"event_type": "guidance_raised", "direction": 1, "severity": 1.0, "confidence": 0.8, '
                '"evidence": "raises full-year guidance"}',
    "invented evidence": '{"event_type": "guidance_raised", "direction": 1, "severity": 1.0, '
                         '"confidence": 0.8, "evidence": "record profit"}',
    "made-up type": '{"event_type": "takeover_rumour", "direction": 1, "severity": 0.5, "confidence": 0.9, '
                    '"evidence": "raises full-year guidance"}',
    "not JSON": "The company looks great, buy it!",
}
for label, reply in replies.items():
    extractor = LLMExtractor(get_prompt("llm-v1"), FakeClient(reply))
    event = validate_events(extract_events(story, extractor), EVENT_TYPES).iloc[0]
    print(f"{label:<18} {event['status']:<24} {event['extraction_problems']}")
```

```text
faithful           OK                       
invented evidence  NEWS_EXTRACTION_INVALID  evidence is not a quote from the story
made-up type       NEWS_EXTRACTION_INVALID  unknown event_type
not JSON           NEWS_EXTRACTION_INVALID  unknown event_type; evidence is not a quote from the story
```

Only the faithful reply survives. The "buy it!" reply shows the design's rule at work: the model's opinion about the stock is not in the schema, so it cannot reach anything.

**How to wire it in.**

1. Make open decision 4. Implement `provider_client` for the approved provider, with a timeout and retry policy, and log the model name and version on every call.
2. In `stage_02d_news_features\service.py`, build `LLMExtractor(prompt, provider_client())` when `extractor: llm`, instead of falling back.
3. Set `extractor: llm` and `prompt_version: llm-v1` in `news_features.yaml`. The news version changes (Part 09), so **register a new experiment id**: the challenger built on LLM features is a different challenger.
4. Freeze the prompt for the whole evaluation period. Run the lexicon and LLM challengers side by side as two experiments; Holm (Part 10) charges for testing both.
5. Monitor `NEWS_EXTRACTION_INVALID` counts in the story log. A rising rejection rate is an early sign that the model or the provider has changed.

## 14.6 More Z_news candidates: propagation and calibrated event risk

**File:** `src/aqp_python/stages/stage_02d_news_features/graph_propagation.py`

```python
"""Propagate an event shock through the relationship graph with decay (design 2D).

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

**What it does.** Spreads an event's shock to related companies (suppliers, customers, competitors) through a weight matrix `W`, decaying at each step: `Σ_k (δW)^k shock`. The series converges only if the spectral radius of `δW` is below one, so that is checked first.

```python
import numpy as np

from aqp_python.stages.stage_02d_news_features.graph_propagation import propagate

# A supplies B (weight 0.6); B supplies C (weight 0.5). A shock of -1 hits A.
weights = np.array([[0.0, 0.0, 0.0],
                    [0.6, 0.0, 0.0],
                    [0.0, 0.5, 0.0]])
print("effect on A, B, C:", propagate(weights, np.array([-1.0, 0.0, 0.0]), decay=0.8, order=3).round(3))
try:
    propagate(np.array([[0.0, 1.0], [1.0, 0.0]]), np.array([1.0, 0.0]), decay=1.2)
except ValueError as error:
    print("refused:", error)
```

```text
effect on A, B, C: [ 0.    -0.48  -0.192]
refused: unstable propagation: spectral radius 1.200 >= 1
```

B receives `0.8 × 0.6 = 0.48` of the shock, and C `0.8 × 0.5 × 0.48 = 0.192`. A itself is not changed: propagation adds spill-overs, and the direct effect is already in A's own features.

**File:** `src/aqp_python/stages/stage_02d_news_features/event_risk_calibration.py` — component 1 of 2: fit_event_risk: a calibrated logistic model on Z_news

```python
"""Calibrated event risk (design 2D): P(|AR| > c * sigma | z_news) = logistic(theta' z_news).

The probability is a FEATURE, never a forecast or an order. It must pass its own
out-of-sample checks (Brier score, log loss, reliability) before anyone may use it.
"""
import numpy as np
import pandas as pd
from sklearn.linear_model import LogisticRegression


def abnormal_move(labels: pd.Series, sigma_h: pd.Series, threshold: float) -> pd.Series:
    """1 if the horizon's abnormal return is larger than `threshold` standard deviations."""
    return (labels.abs() > threshold * sigma_h).astype(int)


def fit_event_risk(train: pd.DataFrame, features: list[str], target: str = "abnormal") -> LogisticRegression:
    """Fit only on rows whose label window closed before the decision (the caller must purge)."""
    rows = train.dropna(subset=features + [target])
    return LogisticRegression(C=1.0, max_iter=1000).fit(rows[features].to_numpy(), rows[target].to_numpy())
```

**File:** `src/aqp_python/stages/stage_02d_news_features/event_risk_calibration.py` — component 2 of 2: calibration evidence: Brier, log loss and reliability

```python
def calibration_evidence(probability: np.ndarray, outcome: np.ndarray, bins: int = 5) -> dict:
    probability = np.clip(np.asarray(probability, dtype=float), 1e-6, 1 - 1e-6)
    outcome = np.asarray(outcome, dtype=float)
    edges = np.quantile(probability, np.linspace(0, 1, bins + 1))
    groups = np.clip(np.searchsorted(edges, probability, side="right") - 1, 0, bins - 1)
    reliability = [(float(probability[groups == b].mean()), float(outcome[groups == b].mean()))
                   for b in range(bins) if (groups == b).any()]
    return {"brier": float(np.mean((probability - outcome) ** 2)),
            "log_loss": float(-np.mean(outcome * np.log(probability) + (1 - outcome) * np.log(1 - probability))),
            "reliability": reliability}
```

**What it does.** A logistic model for the probability of an abnormal move larger than `c` standard deviations, given Z_news, and the evidence needed to trust it: the **Brier score** (mean squared error of the probabilities; lower is better), the **log loss**, and a **reliability table** of (average predicted, observed frequency) per probability bucket. A calibrated model has the two numbers close in every bucket.

**How to wire either in.** Each would be a new column in Z_news, published by Stage 02D like the three existing ones. That changes the news version and the challenger's feature list, so it is a new experiment, and the ablation in Part 10 must show the new column itself helps. The relationship weights `W` need a dated source (open decision 4 covers news sources; a supply-chain dataset is a separate licence), and the calibration model must be fitted inside each training fold, like every other model.

## 14.7 The R boundary

Repository overview: R models are called only through an adapter with a documented input/output contract, a timeout, and failures recorded as model failures.

**File:** `r_models/adapters/r_model_adapter.py` — component 1 of 2: docstring and the error type

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
```

**File:** `r_models/adapters/r_model_adapter.py` — component 2 of 2: run_r_model

```python
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

A template R script for DCC-GARCH (it needs R with `jsonlite` and `rmgarch`; the adapter's contract is tested with a stand-in script):

**File:** `r_models/scripts/dcc_garch.R`

```r
# DCC-GARCH correlation forecast (design Stages 3-4). TEMPLATE: needs R with jsonlite and rmgarch.
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

**How to wire it in as a covariance engine with an approved fallback:**

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

List the fallback in `configs\fallbacks.yaml` next to `news_features_unavailable`, and add a test like `test_fallback_behaviour.py` for it. A fallback must be approved and tested, and may never bypass data integrity, margin, hard limits, kill switches or reconciliation.

## 14.8 Validation: what to add next

- **Stability across folds:** record each fold's ridge coefficients in `out_of_sample_predictions`, add a `sign_stability` metric to `score_models`, and a `min_sign_stability` threshold to G3. For a news challenger, check that the news coefficients keep their sign.
- **PBO, SPA and MCS:** useful once there are many candidates and experiments. Add them to `overfitting_controls.py` and `forecast_comparison_tests.py` with their own unit tests.
- **Conformal calibration** of the combined quantiles on held-out data.

## 14.9 More sleeves

To add `us_interday`:

1. `configs\sleeves\sleeve_us_interday.yaml`, copied from the ASX file, with US costs, limits and stress scenarios (add an AUD/USD shock and an overnight-gap scenario).
2. One line in `pipeline.yaml → sleeves`.
3. A US screening source and a detailed source, a US calendar, and a USD → AUD translation for NAV.
4. **Its own experiments** in `experiments.yaml` (the experiment id already contains the sleeve). Evidence from the ASX sleeve is not evidence for the US sleeve.
5. A **sleeve allocator** above the per-sleeve optimisers that splits capital by risk budget, before Stage 05.

Each sleeve keeps its own paper book, universe version, data requests and run folders.

## 14.10 Paper trading on real data

1. **Schedule** a daily paper run after the data is published. Create `scripts\run_paper_today.ps1`:

```powershell
Set-Location (Join-Path $PSScriptRoot "..")
$today = Get-Date -Format "yyyy-MM-dd"
& .\.venv\Scripts\python.exe scripts\run_pipeline.py --as-of $today --mode paper *>> .\outputs\scheduler.log
exit $LASTEXITCODE
```

   and register it once (17:15 local time, weekdays):

```powershell
$script = (Resolve-Path .\scripts\run_paper_today.ps1).Path
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$script`""   # Bypass applies to this one process only
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday,Tuesday,Wednesday,Thursday,Friday -At 5:15PM
Register-ScheduledTask -TaskName "AQP paper asx_interday" -Action $action -Trigger $trigger
```

   The synthetic data ends on 2026-10-16, so scheduled runs only make sense with a live source. Until then they block at screening (`DATA_STALE`), which is the correct behaviour.

2. **Drills** (record date, result and reviewer in `docs\readiness\`): stale data; restart mid-run (kill the process during Stage 02E; the next run must start cleanly with the book unchanged); kill switch; a reconciliation break (edit `paper_book.json` by hand; expect `RECONCILIATION_FAILED`, the kill switch engaged and an incident logged); **news feed late** (delay the news file past 17:00; expect stories marked `NEWS_NOT_YET_AVAILABLE` and lower coverage); **news feed down** (drill 3 of Part 13, now on real data); latency from data publication to orders, from `run.log`.

3. **Monitor for weeks, not days**: coverage alerts, drawdown, implementation shortfall against the cost model, news coverage and drift, and whether the G3 evidence and the paired evidence still hold.

## 14.11 Promotion: from shadow candidate to approved arm (G7)

Part 10 ended with the news challenger as a `SHADOW_CANDIDATE`: it passed every test, it is recorded in every run, and it does not trade. This is how it may, after review, become the approved arm.

**File:** `src/aqp_python/gates/g7_model_promotion.py`

```python
"""G7 - model promotion. A challenger never replaces the approved model automatically."""
from collections.abc import Mapping

from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G7"


def check_promotion(challenger: str, evidence: Mapping, rules: Mapping) -> GateResult:
    """All conditions in design Stage 7 must hold; otherwise keep the current approved model."""
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

**What it does.** Six conditions, all required: the challenger is G3-eligible; it has been a shadow candidate for at least `min_shadow_sessions` (60, from `model_lifecycle.yaml`); its calibration is intact; it beats the currently approved arm (`dm_pvalue_vs_champion`); a rollback has been tested; and someone other than its author approved it. Any failure keeps the current approved arm.

**The promotion procedure.**

1. **Collect the evidence.** Every research or paper run writes `validation\experiments.csv` and `monitoring\readiness.json` (which lists shadow candidates). A `model_lifecycle\shadow_tracker.py` (planned) collects these across runs into the G7 evidence dictionary: the number of sessions as a shadow candidate, the rolling coverage, and the paired DM test of the challenger against the approved arm on all shadow sessions.
2. **Review.** Run `check_promotion`. If it passes, an independent reviewer signs the decision log entry.
3. **Release.** Promotion is a reviewed commit that changes **one line**, `approved_arm: BASELINE` → `approved_arm: NEWS_CHALLENGER`, in `experiments.yaml`, tagged in Git. Because `approved_arm` is excluded from the experiment's hash (Part 10), the experiment and its history are unchanged; and `test_the_shipped_configuration_approves_only_baselines` (Part 10) will fail until it is updated in the same reviewed commit. That failure is deliberate: promotion must be visible in the test suite too.
4. **Keep testing.** After promotion the challenger trades **only while it still passes every test** (`select_slot`, Part 10). If it stops passing, the run falls back to the baseline and records `APPROVED_ARM_INELIGIBLE`. **Rollback** is the reverse commit, and it must be tested before promotion.

See step 3 and step 4 on the sample data. Change the approved arm and run (then change it back):

```powershell
(Get-Content configs\experiments.yaml) -replace 'approved_arm: BASELINE', 'approved_arm: NEWS_CHALLENGER' | Set-Content configs\experiments.yaml
python scripts\run_pipeline.py --as-of 2026-10-16
git checkout configs\experiments.yaml
```

```text
2026-10-21 17:48:41,520 INFO    aqp_python.orchestration.orchestrator | stage_02e_validation     PASS 
status : PASS
```

```text
   approved_arm                            used_slot challenger_decision
NEWS_CHALLENGER ridge_location_scale@NEWS_CHALLENGER    SHADOW_CANDIDATE
```

The run now trades the news challenger, because it still passes every test. In a real promotion this step comes only after G7.

`model_lifecycle` must never import the broker or Stage 06: the import-linter contract `lifecycle-no-orders` (Part 13) already enforces it, before the package has any code.

## 14.12 Controlled deployment

This is a sign-off process, not a test batch. Do not write a live broker adapter until all of the following exist and have been reviewed by someone other than the author:

- [ ] Written decisions on all seven open design questions (below), with the limits in YAML.
- [ ] At least the agreed paper-trading period with reconciliation clean every day.
- [ ] A live adapter implementing `connectors\broker\broker_interface.py`, used only by Stage 06, with broker-side limits configured as a second, independent safety net.
- [ ] A tested rollback (switch back to paper mode) and an incident-response runbook.
- [ ] A small, approved initial allocation.

Keep a decision record in the repository. `docs\decisions\open_decisions.md`:

```markdown
| # | Decision | Options | Current setting | Config key | Owner | Status |
|---|---|---|---|---|---|---|
| 1 | Instruments and capital structure | cash equities on margin / CFDs / futures | cash equities, no margin | sleeve.sizing.initial_margin_rate | | open |
| 2 | Per-sleeve vol target, drawdown, name/sector/gross limits, margin buffer | | 10% / 15% / 8% / 30% / 1.0 / 20% | sleeve.*, risk_limits.* | | provisional |
| 3 | Screening and detailed data providers; intraday coverage by tier | | synthetic CSV | screening_request, detailed_data_request, coverage_tiers | | open |
| 4 | News sources, latency, LLM provider and data retention | lexicon / LLM; providers | lexicon, no provider | news_features.extractor, connectors/llm | | open |
| 5 | Long-only vs long-short; ASX borrow | | long-only | sleeve.portfolio.long_only | | provisional |
| 6 | Universe re-formation: deliberate or scheduled | | deliberate (new version_label) | universe.version_label | | provisional |
| 7 | Promotion policy for news challengers | shadow length, reviewer | 60 sessions, independent reviewer | model_lifecycle.yaml | | provisional |
```

and `docs\decisions\decision_log.md`, one dated entry per change of any limit, code, model, prompt or approved arm: *what changed, why, the evidence, who approved it, and the commit*.

## 14.13 Tests for the extensions

**File:** `tests/unit/test_extensions.py`

```python
"""Tests for the Part 14 extension modules. They are not wired into the default run."""

import numpy as np
import pandas as pd
import pytest

from aqp_python.gates.g7_model_promotion import check_promotion
from aqp_python.stages.stage_02_forecasting.firm_microstructure.intraday.realised_measures import (
    bipower_variation,
    jump_component,
    realised_variance,
)
from aqp_python.stages.stage_02_forecasting.macro.company_macro_exposure import exposure_weighted_factor
from aqp_python.stages.stage_02_forecasting.macro.macro_sensitivity import shrink_sensitivities
from aqp_python.stages.stage_02_forecasting.macro.release_surprise import standardised_surprise
from aqp_python.stages.stage_02d_news_features.evidence_validator import validate_events
from aqp_python.stages.stage_02d_news_features.llm_event_extractor import LLMExtractor, extract_events
from aqp_python.stages.stage_02d_news_features.prompt_registry import EVENT_TYPES, get_prompt

pytestmark = pytest.mark.extension


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


class FakeClient:
    def __init__(self, reply: str) -> None:
        self.reply = reply

    def complete(self, prompt: str) -> str:
        assert "Do not infer price moves" in prompt          # the registered prompt is what was sent
        return self.reply


def test_an_llm_extractor_is_judged_by_the_same_validator():
    stories = pd.DataFrame({"headline": ["A raises full-year guidance"] * 2, "body": ["", ""], "status": ["OK"] * 2})
    good = ('{"event_type": "guidance_raised", "direction": 1, "severity": 1.0, "confidence": 0.8,'
            ' "evidence": "raises full-year guidance"}')
    invented = good.replace("raises full-year guidance", "record profit")
    prompt = get_prompt("llm-v1")
    for reply, status in ((good, "OK"), (invented, "NEWS_EXTRACTION_INVALID"), ("not json", "NEWS_EXTRACTION_INVALID")):
        events = validate_events(extract_events(stories.head(1), LLMExtractor(prompt, FakeClient(reply))), EVENT_TYPES)
        assert events.loc[0, "status"] == status


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
```

`test_an_llm_extractor_is_judged_by_the_same_validator` is the try-it from 14.5 as a test. `FakeClient` also checks that the registered prompt, with its *"Do not infer price moves"* instruction, is what was sent.

**File:** `tests/unit/test_graph_propagation.py`

```python
import numpy as np
import pytest

from aqp_python.stages.stage_02d_news_features.graph_propagation import propagate

pytestmark = pytest.mark.extension


def test_propagation_decays_and_refuses_an_unstable_graph():
    w = np.array([[0.0, 1.0], [1.0, 0.0]])
    assert propagate(w, np.array([1.0, 0.0]), decay=0.5, order=2)[1] == pytest.approx(0.5)
    with pytest.raises(ValueError, match="spectral radius"):
        propagate(w, np.array([1.0, 0.0]), decay=1.2)
```

**File:** `tests/unit/test_event_risk_calibration.py`

```python
import numpy as np
import pandas as pd
import pytest

from aqp_python.stages.stage_02d_news_features.event_risk_calibration import (
    abnormal_move,
    calibration_evidence,
    fit_event_risk,
)

pytestmark = pytest.mark.extension


def test_abnormal_move_is_measured_in_volatility_units():
    labels, sigma = pd.Series([0.01, -0.05, 0.03]), pd.Series([0.02, 0.02, 0.02])
    assert abnormal_move(labels, sigma, threshold=1.0).tolist() == [0, 1, 1]


def test_a_calibrated_model_beats_the_base_rate_and_is_reliable():
    rng = np.random.default_rng(0)
    intensity = rng.exponential(1.0, 6000)
    outcome = rng.uniform(size=6000) < 1 / (1 + np.exp(-(-2.0 + 1.2 * intensity)))
    frame = pd.DataFrame({"news_intensity_decay": intensity, "abnormal": outcome.astype(int)})
    train, test = frame.iloc[:4000], frame.iloc[4000:]
    model = fit_event_risk(train, ["news_intensity_decay"])
    p = model.predict_proba(test[["news_intensity_decay"]].to_numpy())[:, 1]
    evidence = calibration_evidence(p, test["abnormal"].to_numpy())
    base_rate = calibration_evidence(np.full(len(test), train["abnormal"].mean()), test["abnormal"].to_numpy())
    assert evidence["brier"] < base_rate["brier"]
    assert all(abs(predicted - observed) < 0.06 for predicted, observed in evidence["reliability"])
```

The second test is a calibration test of the kind 14.6 asks for: data generated from a known logistic model, a fit on the first 4,000 rows, and a check on the last 2,000 that the model beats the base rate and is reliable in every bucket.

**File:** `tests/integration/test_r_adapter.py`

```python
import sys

import pytest

from r_models.adapters.r_model_adapter import RModelError, run_r_model

pytestmark = pytest.mark.extension


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
python -m pytest -m extension -q
```

```text
11 passed in 1.85s
```

The GARCH test is skipped if `arch` is not installed (`python -m pip install arch`).

---

**The rule to keep:** research models and the news engine propose forecasts and covariates; the paired evaluation decides what may be used; only a reviewed G7 change promotes a challenger; and only independent risk and execution controls authorise orders. Every extension should leave that sentence true, and the invariants in `tests\invariants\` are there to keep it so.
