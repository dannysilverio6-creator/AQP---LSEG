# Part 05 — Stage 01: a fixed, versioned research universe and its coverage tiers (G1)

**Goal:** choose the companies the research may study, **once per universe version**, with a reason code for every rejection, and assign each admitted company a data-coverage tier. Afterwards, apply only the mandatory removals the design lists. A low score never removes a member, and a member that leaves never comes back within the version.

**Design:** Stage 1. The order of filters is *corporate eligibility/distress → liquidity/tradability → data depth and stability → strategy classification → category score → admission*. *"Historical formation must include later-delisted companies"*; *"No score-based removal, replacement or re-entry is allowed within a universe version"*; *"The screen… must not use the post-filter NLP news score as an admission criterion."* **Overview:** §4 row "01 — Universe".

**Selection is not forecasting, and it is not news.** Stage 01 never imports any Stage 2 package. A name being in the universe means it may be *studied*, not that it will be *bought*. This part ends with two invariant tests that prove it.

```mermaid
flowchart TB
    A([formation date 2025-06-30]) --> B["metrics as of formation<br/>using reference vintages known then"]
    B --> C{G1 hard filters}
    C -->|NOT_LISTED / DIST / LIQ / HIST / DATA| R[rejected + reason]
    C --> D[robust sector-relative scores]
    D --> E{"score ≥ 40 and within capacity 30?"}
    E -->|no| R2[BELOW_THRESHOLD / CAPACITY]
    E -->|yes| T["coverage tier T1 / T2 / T3<br/>by formation liquidity"]
    T --> F[("frozen version ASX_20250630_v1")]
    F --> G([every later run: load, never re-form])
    G --> H{G1 mandatory removal checks today}
    H -->|EXIT-DLST / SUSP / DIST / LIQ| X["exit log; never re-enter"]
    H --> I(["active members = members − exits − quarantined"])
```

## What you will build in this part

| Section | File | Components |
|---|---|---|
| 5.1 | `stage_01_universe/universe_metrics.py` | 2 |
| 5.2 | `gates/g1_universe.py` | 2 |
| 5.3 | `stage_01_universe/composite_score.py`, `strategy_classification.py` | 2 + 1 |
| 5.4 | `stage_01_universe/universe_versioning.py`, `coverage_tier_assignment.py` | 1 + 1 |
| 5.5 | `contracts/universe_schema.py`, `stage_01_universe/universe_formation.py` | 1 + 2 |
| 5.6 | `storage/universe_store.py`, `stage_01_universe/mandatory_removals.py` | 2 + 1 |
| 5.7 | `stage_01_universe/service.py` | 4 |
| 5.8 | tests and two invariants | |

---

## 5.1 Metrics, as of a date

**File:** `src/aqp_python/stages/stage_01_universe/universe_metrics.py` — component 1 of 2: imports and listed_mask

```python
"""Point-in-time metrics used by formation filters, scoring and mandatory removals."""
import numpy as np
import pandas as pd


def listed_mask(reference: pd.DataFrame, date: pd.Timestamp) -> pd.Series:
    return (reference["listing_date"] <= date) & (
        reference["delisting_date"].isna() | (reference["delisting_date"] > date))
```

**File:** `src/aqp_python/stages/stage_01_universe/universe_metrics.py` — component 2 of 2: compute_metrics: one row per candidate, as of a date

```python
def compute_metrics(prices: pd.DataFrame, benchmark: pd.DataFrame, reference: pd.DataFrame,
                    trading_days: pd.DatetimeIndex, as_of_date: pd.Timestamp,
                    lookback_days: int) -> pd.DataFrame:
    """One row per security in `reference`, using only prices dated <= as_of_date."""
    prices = prices.loc[prices["date"] <= as_of_date]
    days = trading_days[trading_days <= as_of_date]
    recent_start = days[-lookback_days] if len(days) >= lookback_days else days[0]
    year_start = days[-252] if len(days) >= 252 else days[0]
    bench = benchmark.set_index("date")["benchmark_return"]
    listed = listed_mask(reference, as_of_date).to_numpy()

    rows = []
    for (security_id, ref), is_listed in zip(reference.set_index("security_id").iterrows(), listed):
        group = prices.loc[prices["security_id"] == security_id].set_index("date").sort_index()
        recent = group.loc[group.index >= recent_start]
        year = group.loc[group.index >= year_start]
        history = int(((days >= group.index.min()) & (days <= group.index.max())).sum()) if len(group) else 0
        row = {
            "security_id": security_id, "sector": ref["sector"], "listed": bool(is_listed),
            "distress_probability": float(ref["distress_probability"]),
            "quality_score": float(ref["quality_score"]),
            "history_days": len(group),
            "missing_ratio": 1 - len(group) / history if history else 1.0,
            "days_since_last_trade": int((days > group.index.max()).sum()) if len(group) else 10**6,
            "median_dollar_volume": float(recent["dollar_volume"].median()) if len(recent) else 0.0,
            "zero_volume_ratio": float((recent["volume"] <= 0).mean()) if len(recent) else 1.0,
        }
        r = year["return"].dropna()
        dollar = year["dollar_volume"].reindex(r.index)
        row["amihud_illiquidity"] = float((r.abs() / dollar.replace(0, np.nan)).mean() * 1e6) if len(r) else np.nan
        row["volatility_252"] = float(r.std() * np.sqrt(252)) if len(r) > 20 else np.nan
        row["log_dollar_volume"] = float(np.log1p(row["median_dollar_volume"]))
        b = bench.reindex(r.index)
        half = len(r) // 2
        if half > 40:
            betas = [np.cov(r.iloc[s], b.iloc[s])[0, 1] / b.iloc[s].var()
                     for s in (slice(0, half), slice(half, None))]
            row["beta_instability"] = float(abs(betas[1] - betas[0]))
        else:
            row["beta_instability"] = np.nan
        closes = group["adj_close"]
        if len(closes) > 252:
            row["momentum_12_1"] = float(closes.iloc[-22] / closes.iloc[-253] - 1)
        else:
            row["momentum_12_1"] = np.nan
        rows.append(row)
    return pd.DataFrame(rows).set_index("security_id")
```

**What it does.** One row per candidate in `reference`, using only prices dated on or before `as_of_date`. The same function serves two purposes: it is called with the **formation date** to form the universe, and with **today's date** to check mandatory removals.

**Why it loops over the reference table.** The reference table includes companies that have since been delisted. That is the survivorship rule in practice: SYN05, delisted in 2023, is *evaluated* (and rejected as `NOT_LISTED`) instead of silently missing.

| Category | Metrics computed here | Direction |
|---|---|---|
| Liquidity | median daily dollar volume (log), Amihud illiquidity `mean(\|r\| / dollar volume)`, zero-volume ratio | +, −, − |
| Stability | annualised volatility, beta instability (first-half vs second-half beta over the last year) | −, − |
| Suitability | quality score, distress probability | +, − |

Momentum 12-1 is computed too. It is not scored; it routes strategy classification. Note what is *not* here: nothing about news. The coverage inventory from Part 04 is not even passed to this function.

## 5.2 The G1 gate

**File:** `src/aqp_python/gates/g1_universe.py` — component 1 of 2: imports and check_formation: ordered hard filters

```python
"""G1 - universe gate: hard formation filters and mandatory-removal checks."""
from collections.abc import Mapping

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G1"


def check_formation(security_id: str, metrics: Mapping, rules: Mapping) -> GateResult:
    """Ordered hard filters: corporate status -> distress -> liquidity -> history/data."""
    codes = []
    if not metrics["listed"]:
        codes.append(RC.NOT_LISTED)
    if metrics["distress_probability"] > rules["max_distress_probability"]:
        codes.append(RC.DIST)
    if (metrics["median_dollar_volume"] < rules["min_median_dollar_volume"]
            or metrics["zero_volume_ratio"] > rules["max_zero_volume_ratio"]):
        codes.append(RC.LIQ)
    if metrics["history_days"] < rules["min_history_days"]:
        codes.append(RC.HIST)
    if metrics["missing_ratio"] > rules["max_missing_ratio"]:
        codes.append(RC.DATA)
    if codes:
        return GateResult(GATE, GateOutcome.BLOCK, security_id, tuple(codes), "rejected at formation")
    return GateResult(GATE, GateOutcome.PASS, security_id)
```

**What it does.** The hard filters, in the design's order: corporate status, distress, liquidity, history, data quality. Fail any one and you are out, whatever your score.

**File:** `src/aqp_python/gates/g1_universe.py` — component 2 of 2: check_membership: mandatory removals only

```python
def check_membership(security_id: str, metrics: Mapping, rules: Mapping) -> GateResult:
    """Existing members may only leave for mandatory reasons - never because of a score."""
    removal = rules["mandatory_removal"]
    codes = []
    if not metrics["listed"]:
        codes.append(RC.EXIT_DLST)
    elif metrics["days_since_last_trade"] > removal["suspension_days"]:
        codes.append(RC.EXIT_SUSP)
    if metrics["distress_probability"] > removal["distress_probability"]:
        codes.append(RC.EXIT_DIST)
    floor = removal["liquidity_floor_ratio"] * rules["min_median_dollar_volume"]
    if metrics["median_dollar_volume"] < floor:
        codes.append(RC.EXIT_LIQ)
    if codes:
        return GateResult(GATE, GateOutcome.BLOCK, security_id, tuple(codes), "mandatory removal")
    return GateResult(GATE, GateOutcome.PASS, security_id)
```

**What it does.** The **only** way an existing member can leave, and its codes are exactly the design's exit codes. **Notice what it does not check:** score. A member whose composite score falls from 70 to 30 stays.

The overview plans separate files (`corporate_status_filter.py`, `distress_screen.py`, `liquidity_screen.py`, `data_history_check.py`). Each would be about five lines, so they are combined in the gate here. Split them out when they grow (Appendix B).

## 5.3 Robust, sector-relative scoring

Design Stage 1:

```text
z_ij = (x_ij − median_s(x_j)) / (1.4826 · MAD_s(x_j))         robust z-score within sector s
C_ik = Σ_{j∈k} ω_jk Φ(d_j z_ij)                                category score, 0..1
S_i  = 100 Σ_k w_k C_ik                                        composite score, 0..100
```

`1.4826 · MAD` estimates the standard deviation for normal data, but a single outlier barely moves it, unlike the ordinary standard deviation. `Φ` (the standard normal CDF) squashes each z into 0..1, so no single extreme metric dominates. `d_j` flips metrics where lower is better.

**File:** `src/aqp_python/stages/stage_01_universe/composite_score.py` — component 1 of 2: imports and robust_z

```python
"""Robust, sector-relative scoring: z = (x - median) / (1.4826 MAD), C = sum w Phi(d z), S = 100 sum w C."""
import numpy as np
import pandas as pd
from scipy.stats import norm


def robust_z(values: pd.Series, groups: pd.Series) -> pd.Series:
    """Standardise within each group; a group with no dispersion scores 0, not infinity."""
    def _one(x: pd.Series) -> pd.Series:
        median = x.median()
        mad = (x - median).abs().median() * 1.4826
        if not np.isfinite(mad) or mad == 0:
            return pd.Series(0.0, index=x.index)
        return (x - median) / mad
    return values.groupby(groups, group_keys=False).apply(_one).reindex(values.index)
```

**What it does.** Standardises within each group. A group with no dispersion (MAD = 0) scores 0 instead of dividing by zero. The overview's `score_standardisation.py` is this function (Appendix B).

**File:** `src/aqp_python/stages/stage_01_universe/composite_score.py` — component 2 of 2: composite_scores: category and composite scores

```python
def composite_scores(metrics: pd.DataFrame, category_metrics: dict,
                     category_weights: dict) -> pd.DataFrame:
    """Return one column per category score (0-1) plus `composite_score` (0-100)."""
    out = pd.DataFrame(index=metrics.index)
    for category, spec in category_metrics.items():
        total = pd.Series(0.0, index=metrics.index)
        for metric, (direction, weight) in spec.items():
            z = robust_z(metrics[metric].astype(float), metrics["sector"]).fillna(0.0)
            total += weight * norm.cdf(direction * z)
        out[f"score_{category}"] = total
    out["composite_score"] = 100 * sum(
        weight * out[f"score_{category}"] for category, weight in category_weights.items())
    return out
```

**What it does.** Combines metrics into category scores and the composite. A missing metric is filled with `z = 0` (the sector median), which neither helps nor hurts. This is the one place in the project where filling a missing value is correct: a *score* needs a neutral value, whereas a *feature* (Parts 06 and 09) must keep its missingness visible.

```python
import pandas as pd

from aqp_python.stages.stage_01_universe.composite_score import composite_scores, robust_z

values = pd.Series([1.0, 2.0, 3.0, 100.0])
print("robust z:", robust_z(values, pd.Series(["s"] * 4)).round(3).tolist())
print("plain z: ", ((values - values.mean()) / values.std()).round(3).tolist())
metrics = pd.DataFrame({"sector": ["x"] * 3, "volatility": [0.10, 0.20, 0.40]}, index=["calm", "mid", "wild"])
print(composite_scores(metrics, {"stability": {"volatility": [-1, 1.0]}}, {"stability": 1.0})["composite_score"].round(1))
```

```text
robust z: [-1.012, -0.337, 0.337, 65.763]
plain z:  [-0.52, -0.5, -0.48, 1.5]
calm    75.0
mid     50.0
wild     8.9
Name: composite_score, dtype: float64
```

With the robust z, the outlier (100) does not squash the other three together; with the plain z it does. A direction of −1 makes the calmest name score highest.

**File:** `src/aqp_python/stages/stage_01_universe/strategy_classification.py`

```python
"""Route each admitted company to a modelling hypothesis. This is not a trade signal."""
import pandas as pd


def classify(metrics: pd.DataFrame, rules: dict) -> pd.Series:
    def _one(row: pd.Series) -> str:
        if abs(row.get("momentum_12_1", 0.0) or 0.0) >= rules["momentum_abs_threshold"]:
            return "momentum_trend"
        if row["quality_score"] >= rules["quality_threshold"]:
            return "quality_value"
        return "mean_reversion"
    return metrics.apply(_one, axis=1)
```

**What it does.** Routes each admitted company to a modelling hypothesis (design Stage 1 table). It is not a trade signal. In this build every member feeds the same experiment; Part 14 shows how a class can select experiments.

## 5.4 Versions and coverage tiers

**File:** `src/aqp_python/stages/stage_01_universe/universe_versioning.py`

```python
"""A universe version is named by market, formation date and label - a re-formation is a new name."""
import pandas as pd


def universe_id_for(market: str, rules: dict) -> str:
    return f"{market}_{pd.Timestamp(rules['formation_date']):%Y%m%d}_{rules['version_label']}"
```

The universe ID encodes market, formation date and label (`ASX_20250630_v1`). A deliberate re-formation (design Stage 1) means changing `formation_date` or `version_label`, which creates a new ID.

**File:** `src/aqp_python/stages/stage_01_universe/coverage_tier_assignment.py`

```python
"""Data-coverage tiers (T1/T2/T3): how deeply each ADMITTED name is researched in Stage 2.

Ranked by formation liquidity. A tier controls the data request, never membership.
"""
import pandas as pd


def assign_tiers(admitted: pd.DataFrame, tiers: dict) -> pd.Series:
    """admitted: formation decisions for admitted names (needs median_dollar_volume)."""
    rank = admitted["median_dollar_volume"].rank(ascending=False, method="first").astype(int)
    labels = pd.Series(index=admitted.index, dtype=object)
    lower = 0
    for name, spec in tiers.items():               # YAML order: T1, T2, T3
        upper = spec["max_rank"] if spec["max_rank"] is not None else len(admitted)
        labels[(rank > lower) & (rank <= upper)] = name
        lower = upper
    return labels
```

**What it does.** Ranks the admitted names by formation liquidity and labels them T1, T2 or T3 using `coverage_tiers.yaml`.

**Why tiers are assigned at formation.** Design Stage 1: tiering *"governs the Stage 2 data request and does not silently replace or re-rank members of the frozen universe."* Assigning tiers once, inside the frozen version, makes them as stable as the membership itself.

```python
import pandas as pd

from aqp_python.stages.stage_01_universe.coverage_tier_assignment import assign_tiers

admitted = pd.DataFrame({"median_dollar_volume": [5e6, 9e6, 1e6, 7e6, 3e6]}, index=list("ABCDE"))
tiers = {"T1": {"max_rank": 1}, "T2": {"max_rank": 3}, "T3": {"max_rank": None}}
print(assign_tiers(admitted, tiers).to_dict())
```

```text
{'A': 'T2', 'B': 'T1', 'C': 'T3', 'D': 'T2', 'E': 'T3'}
```

## 5.5 Forming a version

**File:** `src/aqp_python/contracts/universe_schema.py`

```python
"""Stage 01 hand-off: one frozen universe version and the members usable in this run."""
from dataclasses import dataclass

import pandas as pd

# One row per member of a frozen universe version. The coverage tier is fixed with the version.
UNIVERSE_MEMBER_COLUMNS = ("security_id", "sector", "strategy_class", "composite_score", "coverage_tier")


@dataclass(frozen=True)
class UniverseSelection:
    universe_id: str
    members: pd.DataFrame         # UNIVERSE_MEMBER_COLUMNS, frozen at formation
    active_ids: tuple[str, ...]   # members minus mandatory exits minus quarantined names

    @property
    def coverage_tiers(self) -> dict[str, str]:
        return dict(zip(self.members["security_id"], self.members["coverage_tier"]))
```

**What it does.** The Stage 01 hand-off. `coverage_tiers` gives Stage 2 a `security_id → tier` mapping without exposing the formation internals.

**File:** `src/aqp_python/stages/stage_01_universe/universe_formation.py` — component 1 of 2: imports

```python
"""Form a universe version once: hard filters (G1) -> scores -> threshold -> capacity -> tiers."""
import pandas as pd

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.contracts.universe_schema import UNIVERSE_MEMBER_COLUMNS
from aqp_python.gates.g1_universe import check_formation
from aqp_python.gates.gate_result import GateResult
from aqp_python.stages.stage_01_universe.composite_score import composite_scores
from aqp_python.stages.stage_01_universe.coverage_tier_assignment import assign_tiers
from aqp_python.stages.stage_01_universe.strategy_classification import classify
```

**File:** `src/aqp_python/stages/stage_01_universe/universe_formation.py` — component 2 of 2: form_universe

```python
def form_universe(metrics: pd.DataFrame, rules: dict,
                  tiers: dict) -> tuple[pd.DataFrame, pd.DataFrame, list[GateResult]]:
    """Return (members, decisions, gate_results). Every rejection has a reason code."""
    gates = [check_formation(sid, row, rules) for sid, row in metrics.iterrows()]
    decisions = metrics.copy()
    decisions["passed_hard_filters"] = [g.passed for g in gates]
    decisions["reason_codes"] = [";".join(map(str, g.reason_codes)) for g in gates]

    eligible = decisions.loc[decisions["passed_hard_filters"]]
    scores = composite_scores(eligible, rules["category_metrics"], rules["category_weights"])
    decisions = decisions.join(scores)
    decisions["strategy_class"] = classify(decisions, rules["strategy_rules"])

    ranked = decisions.loc[decisions["passed_hard_filters"]].sort_values("composite_score", ascending=False)
    above = ranked.loc[ranked["composite_score"] >= rules["admission_threshold"]]
    admitted = above.index[: rules["capacity"]]
    decisions["admitted"] = decisions.index.isin(admitted)
    decisions.loc[ranked.index.difference(above.index), "reason_codes"] = str(RC.BELOW_THRESHOLD)
    decisions.loc[above.index[rules["capacity"]:], "reason_codes"] = str(RC.CAPACITY)
    decisions["coverage_tier"] = assign_tiers(decisions.loc[admitted], tiers)

    members = (decisions.loc[admitted].reset_index()
               .loc[:, list(UNIVERSE_MEMBER_COLUMNS)].sort_values("security_id"))
    return members.reset_index(drop=True), decisions.reset_index(), gates
```

**What it does.** Hard filters (G1) → scores → threshold → capacity → tiers. Every candidate ends with exactly one outcome in `decisions`: rejected by a hard filter (with codes), `BELOW_THRESHOLD`, `CAPACITY`, or admitted with a tier.

## 5.6 Storing versions and exits

**File:** `src/aqp_python/storage/universe_store.py` — component 1 of 2: save_universe and load_universe: immutable versions

```python
"""Frozen, versioned universe memberships. A saved version is never edited."""
import json
from pathlib import Path

import pandas as pd


def universe_folder(universe_dir: Path, universe_id: str) -> Path:
    return Path(universe_dir) / universe_id


def save_universe(universe_dir: Path, universe_id: str, members: pd.DataFrame,
                  decisions: pd.DataFrame, metadata: dict) -> Path:
    folder = universe_folder(universe_dir, universe_id)
    if folder.exists():
        raise FileExistsError(f"universe {universe_id} already exists; versions are immutable")
    folder.mkdir(parents=True)
    members.to_csv(folder / "members.csv", index=False)
    decisions.to_csv(folder / "formation_decisions.csv", index=False)
    (folder / "metadata.json").write_text(json.dumps(metadata, indent=2, default=str),
                                          encoding="utf-8")
    return folder


def load_universe(universe_dir: Path, universe_id: str) -> tuple[pd.DataFrame, dict]:
    folder = universe_folder(universe_dir, universe_id)
    if not folder.is_dir():
        raise FileNotFoundError(f"universe {universe_id} not found in {universe_dir}")
    members = pd.read_csv(folder / "members.csv")
    metadata = json.loads((folder / "metadata.json").read_text(encoding="utf-8"))
    return members, metadata
```

**What it does.** `save_universe` raises `FileExistsError` if the version already exists: **versions are immutable**.

**File:** `src/aqp_python/storage/universe_store.py` — component 2 of 2: append_exits and load_exits: the exit log

```python
def append_exits(universe_dir: Path, universe_id: str, exits: pd.DataFrame) -> None:
    """Exits are logged next to the frozen membership; the membership file is untouched."""
    if exits.empty:
        return
    path = universe_folder(universe_dir, universe_id) / "exit_log.csv"
    previous = pd.read_csv(path) if path.is_file() else pd.DataFrame(columns=exits.columns)
    combined = pd.concat([previous, exits], ignore_index=True)
    combined = combined.drop_duplicates(["security_id"], keep="first")
    combined.to_csv(path, index=False)


def load_exits(universe_dir: Path, universe_id: str) -> pd.DataFrame:
    path = universe_folder(universe_dir, universe_id) / "exit_log.csv"
    return pd.read_csv(path) if path.is_file() else pd.DataFrame(columns=["security_id"])
```

**What it does.** Exits go to a separate `exit_log.csv` next to the membership, so `members.csv` is never edited. `drop_duplicates(keep="first")` keeps the *first* exit of a name, so its exit date never moves.

**File:** `src/aqp_python/stages/stage_01_universe/mandatory_removals.py`

```python
"""Members leave only for mandatory reasons, and never come back within a version."""
import pandas as pd

from aqp_python.gates.g1_universe import check_membership
from aqp_python.gates.gate_result import GateResult


def find_exits(members: list[str], metrics: pd.DataFrame, rules: dict,
               decision_date: pd.Timestamp) -> tuple[pd.DataFrame, list[GateResult]]:
    gates = [check_membership(sid, metrics.loc[sid], rules) for sid in members]
    exits = [{"security_id": g.subject, "exit_date": decision_date.date(),
              "reason_codes": ";".join(map(str, g.reason_codes))}
             for g in gates if not g.passed]
    return pd.DataFrame(exits, columns=["security_id", "exit_date", "reason_codes"]), gates
```

## 5.7 The Stage 01 service

**File:** `src/aqp_python/stages/stage_01_universe/service.py` — component 1 of 4: imports

```python
"""Stage 01 entry point: load (or form once) the frozen universe, then apply mandatory exits."""
import logging

import pandas as pd

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.contracts.screening_schema import ScreeningData
from aqp_python.contracts.universe_schema import UniverseSelection
from aqp_python.orchestration.run_context import RunContext, resolve_decision_time
from aqp_python.orchestration.stage_status import StageResult
from aqp_python.stages.stage_01_universe.mandatory_removals import find_exits
from aqp_python.stages.stage_01_universe.universe_formation import form_universe
from aqp_python.stages.stage_01_universe.universe_metrics import compute_metrics
from aqp_python.stages.stage_01_universe.universe_versioning import universe_id_for
from aqp_python.storage.universe_store import (
    append_exits,
    load_exits,
    load_universe,
    save_universe,
    universe_folder,
)
from aqp_python.storage.vintage_store import latest_vintage

log = logging.getLogger(__name__)
STAGE = "stage_01_universe"
```

**File:** `src/aqp_python/stages/stage_01_universe/service.py` — component 2 of 4: _form: build a version from what was known on the formation date

```python
def _form(ctx: RunContext, data: ScreeningData, rules: dict, universe_dir, universe_id: str) -> None:
    formation_date = pd.Timestamp(rules["formation_date"])
    formation_time = resolve_decision_time(formation_date.date(), ctx.config["pipeline"])
    if formation_time > ctx.decision_time:
        raise ValueError("formation_date is after the decision time")
    # Formation uses only what was known on the formation date (including reference vintages).
    reference = latest_vintage(data.reference_vintages, formation_time, keys=["security_id"])
    metrics = compute_metrics(data.prices, data.benchmark, reference, data.trading_days,
                              formation_date, rules["liquidity_lookback_days"])
    members, decisions, _ = form_universe(metrics, rules, ctx.config["coverage_tiers"]["tiers"])
    save_universe(universe_dir, universe_id, members, decisions, {
        "universe_id": universe_id, "formation_date": rules["formation_date"],
        "formed_in_run": ctx.run_id, "config_hash": ctx.config_hash, "members": len(members)})
    log.info("Formed universe %s with %d members", universe_id, len(members))
```

**What it does.** Forms a version **using only what was known on the formation date**. It rebuilds the reference table with `latest_vintage(..., formation_time)`, so SYN13's 2026 distress revision is invisible at formation: SYN13 is admitted in 2025 and must leave later through a mandatory exit.

**File:** `src/aqp_python/stages/stage_01_universe/service.py` — component 3 of 4: run_stage: load the version and apply today's mandatory exits

```python
def run_stage(ctx: RunContext, data: ScreeningData) -> tuple[StageResult, UniverseSelection | None]:
    result = StageResult(STAGE)
    rules = ctx.config["universe"]
    universe_dir = ctx.resolve(ctx.config["pipeline"]["universe_dir"])
    universe_id = universe_id_for(ctx.config["sleeve"]["market"], rules)
    if not universe_folder(universe_dir, universe_id).is_dir():
        _form(ctx, data, rules, universe_dir, universe_id)
        result.metrics["formed_now"] = True
    members, metadata = load_universe(universe_dir, universe_id)

    # Mandatory removals at today's decision date, using today's reference vintage.
    already_exited = set(load_exits(universe_dir, universe_id)["security_id"])
    candidates = [s for s in members["security_id"] if s not in already_exited]
    metrics_now = compute_metrics(data.prices, data.benchmark, data.reference, data.trading_days,
                                  ctx.decision_date, rules["liquidity_lookback_days"])
    exits, gates = find_exits(candidates, metrics_now, rules, ctx.decision_date)
    append_exits(universe_dir, universe_id, exits)
    result.gates.extend(gates)
    all_exits = load_exits(universe_dir, universe_id)
    exited = set(all_exits["security_id"])
    active = tuple(sorted(set(members["security_id"]) - exited - data.quarantined))
```

**What it does.** Every run, including the one that formed it, **loads** the version from disk; there is no code path that re-forms an existing version. Mandatory removals use *today's* reference vintage and prices. Quarantined names (from Stage 00) are excluded from `active_ids` for this run only: they are not exits.

**File:** `src/aqp_python/stages/stage_01_universe/service.py` — component 4 of 4: run_stage: write the universe evidence and hand off

```python
    decisions = pd.read_csv(universe_folder(universe_dir, universe_id) / "formation_decisions.csv")
    members.to_csv(ctx.paths.sub("universe", "selected_companies.csv"), index=False)
    decisions.loc[~decisions["admitted"]].to_csv(ctx.paths.sub("universe", "exclusions.csv"), index=False)
    all_exits.to_csv(ctx.paths.sub("universe", "exits.csv"), index=False)
    pd.Series(active, name="security_id").to_csv(ctx.paths.sub("universe", "active_members.csv"), index=False)
    result.artifacts.update({"selected_companies": "universe/selected_companies.csv",
                             "exclusions": "universe/exclusions.csv", "exits": "universe/exits.csv",
                             "active_members": "universe/active_members.csv"})
    result.metrics.update({"universe_id": universe_id, "members": len(members),
                           "exits": sorted(exited), "active": len(active),
                           "tiers": members["coverage_tier"].value_counts().sort_index().to_dict(),
                           "formed_in_run": metadata["formed_in_run"]})
    if not active:
        return result.block(RC.EXIT_LIQ, "no active members remain in the universe"), None
    if len(exits):
        result.warnings.append(f"mandatory exits today: {exits['security_id'].tolist()}")
    return result, UniverseSelection(universe_id, members, active)
```

**What it does.** Writes the universe evidence (`selected_companies.csv`, `exclusions.csv`, `exits.csv`, `active_members.csv`) and the tier counts, then hands over the `UniverseSelection`.

## 5.8 Tests

**File:** `tests/unit/test_universe_scoring.py`

```python
import pandas as pd
import pytest

from aqp_python.stages.stage_01_universe.composite_score import composite_scores, robust_z
from aqp_python.stages.stage_01_universe.strategy_classification import classify

pytestmark = pytest.mark.batch02


def test_robust_z_uses_median_and_mad_within_group():
    values = pd.Series([1.0, 2.0, 3.0, 100.0, 5.0, 5.0])
    groups = pd.Series(["a", "a", "a", "a", "b", "b"])
    z = robust_z(values, groups)
    # group a: median 2.5, MAD 1.0 -> z(2.0) = (2.0 - 2.5) / 1.4826; the outlier 100 barely matters
    assert z.iloc[1] == pytest.approx((2.0 - 2.5) / 1.4826)
    assert (z.iloc[4:] == 0).all()                 # no dispersion -> 0, never infinite


def test_direction_flips_the_score():
    metrics = pd.DataFrame({"sector": ["x"] * 3, "m": [1.0, 2.0, 3.0]}, index=["a", "b", "c"])
    up = composite_scores(metrics, {"cat": {"m": [1, 1.0]}}, {"cat": 1.0})["composite_score"]
    down = composite_scores(metrics, {"cat": {"m": [-1, 1.0]}}, {"cat": 1.0})["composite_score"]
    assert up["c"] > up["a"] and down["a"] > down["c"]
    assert up.between(0, 100).all()


def test_strategy_classification_routes_but_does_not_trade():
    metrics = pd.DataFrame({"momentum_12_1": [0.4, 0.0, 0.0], "quality_score": [0.0, 0.9, 0.1]},
                           index=["a", "b", "c"])
    rules = {"momentum_abs_threshold": 0.25, "quality_threshold": 0.5}
    assert classify(metrics, rules).tolist() == ["momentum_trend", "quality_value", "mean_reversion"]
```

**File:** `tests/unit/test_universe_removals.py`

```python
import pandas as pd
import pytest

from aqp_python.stages.stage_01_universe.mandatory_removals import find_exits
from aqp_python.storage.universe_store import append_exits, load_exits, save_universe

pytestmark = pytest.mark.batch02
RULES = {"min_median_dollar_volume": 1e6,
         "mandatory_removal": {"distress_probability": 0.2, "suspension_days": 5, "liquidity_floor_ratio": 0.5}}
GOOD = {"listed": True, "distress_probability": 0.01, "median_dollar_volume": 5e6, "days_since_last_trade": 0}


def test_find_exits_logs_only_mandatory_removals():
    metrics = pd.DataFrame([GOOD, {**GOOD, "listed": False}], index=["A", "B"])
    exits, gates = find_exits(["A", "B"], metrics, RULES, pd.Timestamp("2026-10-16"))
    assert exits["security_id"].tolist() == ["B"] and exits.loc[0, "reason_codes"] == "EXIT-DLST"
    assert len(gates) == 2


def test_versions_are_immutable_and_first_exit_date_is_kept(tmp_path):
    members = pd.DataFrame({"security_id": ["A"]})
    save_universe(tmp_path, "U1", members, members, {"members": 1})
    with pytest.raises(FileExistsError):
        save_universe(tmp_path, "U1", members, members, {"members": 1})
    for exit_date, code in (("2026-01-02", "X"), ("2026-03-02", "Y")):
        append_exits(tmp_path, "U1", pd.DataFrame({"security_id": ["A"], "exit_date": [exit_date],
                                                   "reason_codes": [code]}))
    assert load_exits(tmp_path, "U1")["exit_date"].tolist() == ["2026-01-02"]
```

**File:** `tests/unit/test_coverage_tiers.py`

```python
import pandas as pd
import pytest

from aqp_python.stages.stage_01_universe.coverage_tier_assignment import assign_tiers

pytestmark = pytest.mark.batch02
TIERS = {"T1": {"max_rank": 2}, "T2": {"max_rank": 4}, "T3": {"max_rank": None}}


def test_tiers_follow_formation_liquidity_rank():
    admitted = pd.DataFrame({"median_dollar_volume": [5e6, 9e6, 1e6, 7e6, 3e6, 2e6]},
                            index=["A", "B", "C", "D", "E", "F"])
    tiers = assign_tiers(admitted, TIERS)
    assert tiers.to_dict() == {"A": "T2", "B": "T1", "C": "T3", "D": "T1", "E": "T2", "F": "T3"}
```

Append the G1 component to `test_gates.py`:

**File:** `tests/unit/test_gates.py` — component 3 of 6: G1 universe (Part 05)

```python
G1_RULES = {"max_distress_probability": 0.1, "min_median_dollar_volume": 1e6, "max_zero_volume_ratio": 0.05,
            "min_history_days": 500, "max_missing_ratio": 0.02,
            "mandatory_removal": {"distress_probability": 0.2, "suspension_days": 5, "liquidity_floor_ratio": 0.5}}
G1_GOOD = {"listed": True, "distress_probability": 0.01, "median_dollar_volume": 5e6, "zero_volume_ratio": 0.0,
           "history_days": 800, "missing_ratio": 0.0, "days_since_last_trade": 0}


@pytest.mark.batch02
@pytest.mark.parametrize("change, code", [
    ({"distress_probability": 0.3}, RC.DIST), ({"median_dollar_volume": 1e3}, RC.LIQ),
    ({"history_days": 100}, RC.HIST), ({"missing_ratio": 0.1}, RC.DATA), ({"listed": False}, RC.NOT_LISTED)])
def test_g1_formation_rejections_have_reason_codes(change, code):
    from aqp_python.gates.g1_universe import check_formation
    result = check_formation("X", {**G1_GOOD, **change}, G1_RULES)
    assert not result.passed and code in result.reason_codes


@pytest.mark.batch02
@pytest.mark.parametrize("change, code", [
    ({"listed": False}, RC.EXIT_DLST), ({"days_since_last_trade": 9}, RC.EXIT_SUSP),
    ({"distress_probability": 0.25}, RC.EXIT_DIST), ({"median_dollar_volume": 1e5}, RC.EXIT_LIQ)])
def test_g1_mandatory_exits(change, code):
    from aqp_python.gates.g1_universe import check_membership
    assert code in check_membership("X", {**G1_GOOD, **change}, G1_RULES).reason_codes


@pytest.mark.batch02
def test_g1_a_low_score_alone_is_not_an_exit():
    from aqp_python.gates.g1_universe import check_membership
    assert check_membership("X", G1_GOOD, G1_RULES).passed
```

`test_g1_a_low_score_alone_is_not_an_exit` is the most important line in this part. It captures the design's "fixed membership" rule in a form that cannot be argued with later.

Add the Stage 01 checkpoint to `tests\integration\test_stage_handoffs.py`:

**File:** `tests/integration/test_stage_handoffs.py` — component 3 of 9: test_stage_01_universe

```python
@pytest.mark.batch02
def test_stage_01_universe(make_context):
    ctx = make_context()
    out = run_through(ctx, "stage_01_universe")
    universe, metrics = out["universe"], out["stage_01_universe"].metrics
    print("\nuniverse:", universe.universe_id, "members:", metrics["members"], "exits:", metrics["exits"],
          "\ntiers:", metrics["tiers"])
    assert {"SYN13", "SYN25", "SYN30"} <= set(metrics["exits"])   # distress, suspension, delisting
    assert not set(universe.active_ids) & set(metrics["exits"])
    assert set(universe.coverage_tiers.values()) == {"T1", "T2", "T3"}
    exclusions = pd.read_csv(ctx.paths.root / "universe" / "exclusions.csv").set_index("security_id")
    assert "DIST" in exclusions.loc["SYN07", "reason_codes"] and "LIQ" in exclusions.loc["SYN09", "reason_codes"]
```

Two invariants join the suite. The first is the survivorship rule:

**File:** `tests/invariants/test_survivorship_safety.py`

```python
"""Historical formation must evaluate companies that were later delisted (design Stage 1)."""
import pandas as pd
import pytest

from aqp_python.stages.stage_00_screening import service as stage_00
from aqp_python.stages.stage_01_universe import service as stage_01

pytestmark = pytest.mark.invariant


def test_delisted_companies_are_evaluated_not_silently_missing(make_context):
    ctx = make_context()
    _, screening = stage_00.run_stage(ctx)
    stage_01.run_stage(ctx, screening)
    decisions = pd.read_csv(ctx.paths.root / "universe" / "exclusions.csv").set_index("security_id")
    members = pd.read_csv(ctx.paths.root / "universe" / "selected_companies.csv")
    assert "NOT_LISTED" in decisions.loc["SYN05", "reason_codes"]     # delisted 2023: evaluated, rejected
    assert "SYN30" in set(members["security_id"])                     # delisted 2026: admitted in 2025
```

The second proves the v14 rule that admission never depends on news, in two independent ways:

**File:** `tests/invariants/test_news_not_used_for_admission.py`

```python
"""Universe admission may record that a news source exists, but must never depend on news."""
import ast
import shutil
from pathlib import Path

import pandas as pd
import pytest

from aqp_python.stages.stage_00_screening import service as stage_00
from aqp_python.stages.stage_01_universe import service as stage_01

pytestmark = pytest.mark.invariant
SRC = Path(__file__).resolve().parents[2] / "src" / "aqp_python"
FORBIDDEN = ("aqp_python.stages.stage_02", "aqp_python.connectors.news", "aqp_python.connectors.llm")


def test_screening_and_universe_code_never_import_news_or_forecasting():
    for package in ("stages/stage_00_screening", "stages/stage_01_universe"):
        for path in (SRC / package).rglob("*.py"):
            for node in ast.walk(ast.parse(path.read_text(encoding="utf-8"))):
                if isinstance(node, ast.ImportFrom) and node.module:
                    assert not node.module.startswith(FORBIDDEN), f"{path.name} imports {node.module}"


def test_membership_is_identical_without_any_news_metadata(make_context, project):
    first = make_context()
    stage_01.run_stage(first, stage_00.run_stage(first)[1])
    with_news = pd.read_csv(first.paths.root / "universe" / "selected_companies.csv")

    shutil.rmtree(project / "data" / "universes")                      # force a fresh formation
    (project / "data" / "samples" / "news_coverage.csv").unlink()      # no news metadata at all
    second = make_context()
    stage_01.run_stage(second, stage_00.run_stage(second)[1])
    without_news = pd.read_csv(second.paths.root / "universe" / "selected_companies.csv")
    pd.testing.assert_frame_equal(with_news, without_news)
```

- The **static** test reads every `import` in the screening and universe packages with Python's `ast` module and fails if any of them reaches Stage 2, the news connector or the LLM connector.
- The **dynamic** test forms the universe twice, the second time with the news coverage metadata deleted, and requires identical membership. If anyone ever wires coverage into scoring, this test fails.

```powershell
python -m pytest -m batch02 -q
python -m pytest tests\invariants\test_survivorship_safety.py tests\invariants\test_news_not_used_for_admission.py -q
python -m pytest tests\integration\test_stage_handoffs.py -k stage_01 -s -q --basetemp=build\pytest
```

```text
universe: ASX_20250630_v1 members: 26 exits: ['SYN13', 'SYN25', 'SYN30'] 
tiers: {'T1': 8, 'T2': 12, 'T3': 6}
.
1 passed in 2.53s
```

Look at the frozen version and the run's universe artifacts:

```python
import glob

import pandas as pd

root = "build/pytest/test_stage_01_universe0"
print(pd.read_csv(f"{root}/data/universes/ASX_20250630_v1/exit_log.csv").to_string(index=False))
run = sorted(glob.glob(f"{root}/outputs/runs/*"))[0]
exclusions = pd.read_csv(f"{run}/universe/exclusions.csv")
print(exclusions[["security_id", "reason_codes", "composite_score"]].round(1).fillna("").to_string(index=False))
members = pd.read_csv(f"{run}/universe/selected_companies.csv")
print(members.groupby("coverage_tier")["security_id"].apply(list).to_dict())
```

```text
security_id  exit_date       reason_codes
      SYN13 2026-10-16          EXIT-DIST
      SYN25 2026-10-16          EXIT-SUSP
      SYN30 2026-10-16 EXIT-DLST;EXIT-LIQ
security_id        reason_codes composite_score
      SYN02     BELOW_THRESHOLD            33.1
      SYN03     BELOW_THRESHOLD            33.7
      SYN05 NOT_LISTED;LIQ;HIST                
      SYN06     BELOW_THRESHOLD            39.1
      SYN07                DIST                
      SYN08     BELOW_THRESHOLD            38.7
      SYN09                 LIQ                
      SYN17     BELOW_THRESHOLD            36.3
      SYN18     BELOW_THRESHOLD            27.8
      SYN20     BELOW_THRESHOLD            39.7
      SYN21                HIST                
      SYN22     BELOW_THRESHOLD            35.5
      SYN23     BELOW_THRESHOLD            30.4
      SYN37     BELOW_THRESHOLD            32.1
{'T1': ['SYN01', 'SYN10', 'SYN19', 'SYN29', 'SYN34', 'SYN35', 'SYN39', 'SYN40'], 'T2': ['SYN04', 'SYN11', 'SYN12', 'SYN14', 'SYN16', 'SYN25', 'SYN27', 'SYN28', 'SYN30', 'SYN31', 'SYN36', 'SYN38'], 'T3': ['SYN13', 'SYN15', 'SYN24', 'SYN26', 'SYN32', 'SYN33']}
```

Every name that is not a member has a reason. Every exit has a code. The membership file itself never changes.

## Checkpoint

- [ ] `python -m pytest -m batch02 -q` → `17 passed`.
- [ ] SYN05, SYN07, SYN09 and SYN21 are rejected with `NOT_LISTED`, `DIST`, `LIQ` and `HIST`.
- [ ] SYN13, SYN25 and SYN30 are admitted at formation and exit later.
- [ ] Both admission invariants pass.

**Check your understanding.** (1) Why does filling a missing metric with the sector median make sense in `composite_scores`, when Part 09 will insist that missing news must never be filled? (2) What would change, and what would not, if you moved SYN10 from T1 to T3?

```powershell
git add .
git commit -m "Stage 01: universe metrics, G1, robust scoring, coverage tiers, frozen versions and mandatory exits"
```

Next: [Part 06 — Stage 2 data expansion and the core feature store](06_stage02_data_expansion_and_core_features.md).
