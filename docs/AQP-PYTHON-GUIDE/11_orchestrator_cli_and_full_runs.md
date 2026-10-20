# Part 11 — The orchestrator, the CLI and full runs

**Goal:** connect the stages into one command, run it, read every output it produces, then use it the way the roadmap intends: research runs, deliberate failure drills, a week of paper trading, and a replay that proves reproducibility. Finish by turning the architecture rules into tests that fail the build.

**Design:** §9 of the repository overview (*"the orchestrator is the only component that launches the full flow"*), design §16 (run contract, terminal status, report rules) and §17 (roadmap phases 6–7 and the stage-completion definition).

---

## 11.1 The orchestrator

**File:** `src/aqp_python/orchestration/orchestrator.py` — top of file, `RunOutcome`, `StageRunner`

```python
"""The only component that runs the full flow. Stages never start the workflow themselves."""
import logging
from collections.abc import Callable
from dataclasses import dataclass
from datetime import UTC, date, datetime
from pathlib import Path

import pandas as pd
import yaml

from aqp_python import __version__
from aqp_python.compiler.plot_compiler import compile_plots
from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.contracts.run_manifest import initial_manifest
from aqp_python.contracts.run_status import RunStatus
from aqp_python.logging_config import configure_logging
from aqp_python.orchestration.run_context import RunContext, create_context
from aqp_python.orchestration.run_manifest_writer import ManifestWriter
from aqp_python.orchestration.stage_status import StageResult
from aqp_python.reporting.metrics_summary import build_run_summary
from aqp_python.reporting.narrative_report import render_report
from aqp_python.reporting.report_fact_checks import unsupported_numbers
from aqp_python.safety.hard_limits import HardLimits
from aqp_python.safety.kill_switch import is_engaged, kill_switch_path
from aqp_python.safety.pre_trade_checks import approve_orders
from aqp_python.stages.stage_00_data import service as stage_00
from aqp_python.stages.stage_01_universe import service as stage_01
from aqp_python.stages.stage_02_forecasting import service as stage_02
from aqp_python.stages.stage_02_forecasting.validation import service as stage_02e
from aqp_python.stages.stage_05_sizing import service as stage_05
from aqp_python.stages.stage_06_execution import service as stage_06
from aqp_python.stages.stage_06_execution.portfolio_state import load_portfolio_state
from aqp_python.stages.stage_07_monitoring import service as stage_07
from aqp_python.stages.stages_03_04_portfolio import service as stage_03_04

log = logging.getLogger(__name__)


@dataclass(frozen=True)
class RunOutcome:
    run_id: str
    status: RunStatus
    run_dir: Path


class StageRunner:
    """Times each stage, converts exceptions into BLOCKED results and records the manifest."""

    def __init__(self, writer: ManifestWriter) -> None:
        self.writer = writer
        self.blocked = False

    def __call__(self, name: str, function: Callable, *args):
        started = datetime.now(UTC)
        try:
            outcome = function(*args)
        except Exception as error:                               # fail closed, never silently
            log.exception("stage %s raised", name)
            result = StageResult(name).block(RC.STAGE_EXCEPTION, f"{type(error).__name__}: {error}")
            outcome = (result, None)
        result = outcome[0] if isinstance(outcome, tuple) else outcome
        self.writer.record_stage(result, started, datetime.now(UTC))
        log.info("%-24s %s %s", name, result.status, ",".join(dict.fromkeys(result.reason_codes)))
        if result.status == RunStatus.BLOCKED:
            self.blocked = True
        return outcome
```

`StageRunner` is a small wrapper with three jobs:

- **Fail closed.** If a stage raises an exception, the stage becomes `BLOCKED` with `STAGE_EXCEPTION` and the full traceback goes to `run.log`. The run stops cleanly with a complete manifest instead of crashing with half-written files.
- **Record.** Every stage is timed and written to the manifest as soon as it finishes.
- **Remember.** `self.blocked` stays `True` once any stage blocks, so no later stage runs.

**File:** `src/aqp_python/orchestration/orchestrator.py` — `run_pipeline`

```python
def run_pipeline(config_path: Path, sleeve: str, as_of: date, project_root: Path,
                 mode: str | None = None) -> RunOutcome:
    ctx = create_context(Path(config_path), sleeve, as_of, Path(project_root), mode)
    configure_logging(log_file=ctx.paths.root / "run.log")
    log.info("run %s | sleeve %s | mode %s | decision %s", ctx.run_id, sleeve, ctx.mode,
             ctx.decision_time.isoformat())
    ctx.paths.config_snapshot.write_text(yaml.safe_dump(ctx.config, sort_keys=True), encoding="utf-8")
    writer = ManifestWriter(ctx.paths.manifest, initial_manifest(
        run_id=ctx.run_id, sleeve=sleeve, mode=ctx.mode, decision_time=ctx.decision_time.isoformat(),
        config_hash=ctx.config_hash, code_revision=ctx.code_revision, package_version=__version__,
        started_at=datetime.now(UTC).isoformat()))
    stage = StageRunner(writer)

    _, data = stage("stage_00_data", stage_00.run_stage, ctx)
    if not stage.blocked:
        _, universe = stage("stage_01_universe", stage_01.run_stage, ctx, data)
    if not stage.blocked:
        _, bundle = stage("stage_02_forecasting", stage_02.run_stage, ctx, data, universe)
    if not stage.blocked:
        _, combined = stage("stage_02e_validation", stage_02e.run_stage, ctx, bundle, data.trading_days)
    if not stage.blocked:
        state = load_portfolio_state(ctx, data)
        _, target = stage("stages_03_04_portfolio", stage_03_04.run_stage, ctx, data, combined, state)
    if not stage.blocked:
        _, orders = stage("stage_05_sizing", stage_05.run_stage, ctx, data, universe, combined, target, state)
    if not stage.blocked:
        approved = stage("safety", _safety_stage, ctx, data, orders, state)[1]
    if not stage.blocked:
        execution = stage("stage_06_execution", stage_06.run_stage, ctx, data, approved, state)
        shortfall = execution.metrics.get("shortfall_bps", 0.0)
        stage("stage_07_monitoring", stage_07.run_stage, ctx, combined, state, shortfall)

    status = writer.finish()
    _compile_outputs(ctx, writer)
    log.info("run %s finished with status %s -> %s", ctx.run_id, status, ctx.paths.root)
    return RunOutcome(ctx.run_id, status, ctx.paths.root)
```

Read `run_pipeline` against the flow in the repository overview §9. Each line is one stage, and each stage receives only typed hand-offs from earlier stages (`data`, `universe`, `bundle`, `combined`, `target`, `orders`). No stage reads another stage's files. `writer.finish()` fixes the terminal status **before** plots and the report are compiled, so a reporting failure can never change it.

**File:** `src/aqp_python/orchestration/orchestrator.py` — `_safety_stage`

```python
def _safety_stage(ctx: RunContext, data, orders, state) -> tuple[StageResult, list]:
    """Independent approval. Uses risk_limits.yaml and plain facts only."""
    result = StageResult("safety")
    limits = HardLimits.from_config(ctx.config["risk_limits"])
    state_dir = ctx.resolve(ctx.config["pipeline"]["state_dir"])
    engaged = is_engaged(kill_switch_path(state_dir, ctx.config["risk_limits"]["kill_switch_file"]))
    recent = data.prices.loc[data.prices["date"] > ctx.decision_date - pd.offsets.BDay(60)]
    equity = state.equity_history
    daily_pnl = float(equity.iloc[-1] / equity.iloc[-2] - 1) if len(equity) > 1 else 0.0
    decision = approve_orders(orders, positions=state.positions, prices=state.prices, nav=state.nav,
                              adv=recent.groupby("security_id")["dollar_volume"].median(),
                              daily_pnl_fraction=daily_pnl, limits=limits, kill_switch_engaged=engaged)
    rows = ([{"order_id": o.order_id, "decision": "APPROVED", "reason_codes": ""} for o in decision.approved]
            + [{"order_id": o.order_id, "decision": "REJECTED", "reason_codes": ";".join(c)}
               for o, c in decision.rejected])
    pd.DataFrame(rows, columns=["order_id", "decision", "reason_codes"]).to_csv(
        ctx.paths.sub("safety", "pre_trade_checks.csv"), index=False)
    result.artifacts["pre_trade_checks"] = "safety/pre_trade_checks.csv"
    result.metrics.update({"approved": len(decision.approved), "rejected": len(decision.rejected),
                           "kill_switch_engaged": engaged})
    if engaged:
        result.block(RC.KILL_SWITCH, "kill switch engaged: no orders may be sent")
    for _, codes in decision.rejected:
        result.reason_codes.extend(codes)
    return result, decision.approved
```

Safety is its own entry in the manifest, between sizing and execution. The orchestrator gathers plain facts (positions, prices, NAV, ADV, today's P&L, the kill-switch file) and passes them to `approve_orders`. It is the only place where sizing's output meets the hard limits.

**File:** `src/aqp_python/orchestration/orchestrator.py` — `_compile_outputs`

```python
def _compile_outputs(ctx: RunContext, writer: ManifestWriter) -> None:
    """Plots and report come last and from recorded outputs; their failure is logged, not hidden."""
    try:
        index = compile_plots(ctx.paths.root)
        summary = build_run_summary(ctx.paths.root)
        narrative = render_report(summary)
        problems = unsupported_numbers(narrative, summary)
        if problems:
            raise ValueError(f"narrative contains numbers not in run_summary.json: {problems}")
        ctx.paths.sub("reports", "run_report.md").write_text(narrative, encoding="utf-8")
        missing = index.loc[index["status"] != "created", "plot"].tolist()
        writer.record_report("CREATED", f"missing plots: {missing}" if missing else "")
    except Exception as error:
        log.exception("report compilation failed; structured outputs are kept")
        writer.record_report("FAILED", f"{type(error).__name__}: {error}")
```

## 11.2 Thin script wrappers

The CLI from Part 02 already has `run` and `build-sample-data`. The overview also lists scripts; keep them as one-line wrappers so there is only one implementation:

**File:** `scripts/run_pipeline.py`

```python
"""Thin wrapper: python scripts/run_pipeline.py --as-of 2026-10-16 [--mode paper] [--sleeve asx_interday]"""
import sys

from aqp_python.cli import main

if __name__ == "__main__":
    raise SystemExit(main(["run", *sys.argv[1:]]))
```

**File:** `scripts/build_sample_data.py`

```python
"""Thin wrapper: python scripts/build_sample_data.py [--seed 7] [--no-signal]"""
import sys

from aqp_python.cli import main

if __name__ == "__main__":
    raise SystemExit(main(["build-sample-data", *sys.argv[1:]]))
```

## 11.3 The first full run

```powershell
python -m aqp_python.cli run --as-of 2026-10-16
```

```text
18:44:20 INFO  orchestrator | run 20261018T074420Z_77d52a5d | sleeve asx_interday | mode research | decision 2026-10-16T17:00:00+11:00
18:44:20 INFO  stage_00_data.service | Stage 00: 40 securities, 2 quarantined
18:44:20 INFO  orchestrator | stage_00_data            PASS
18:44:21 INFO  stage_01_universe.service | Formed universe ASX_20250630_v1 with 27 members
18:44:21 INFO  orchestrator | stage_01_universe        PASS
18:44:21 INFO  orchestrator | stage_02_forecasting     PASS
18:44:22 INFO  orchestrator | stage_02e_validation     PASS
18:44:22 INFO  orchestrator | stages_03_04_portfolio   PASS
18:44:22 INFO  orchestrator | stage_05_sizing          PASS
18:44:22 INFO  orchestrator | safety                   PASS
18:44:22 INFO  orchestrator | stage_06_execution       PASS
18:44:23 INFO  orchestrator | stage_07_monitoring      PASS
18:44:24 INFO  orchestrator | run 20261018T074420Z_77d52a5d finished with status PASS -> C:\...\outputs\runs\20261018T074420Z_77d52a5d

run_id : 20261018T074420Z_77d52a5d
status : PASS
folder : C:\...\AQP-PYTHON\outputs\runs\20261018T074420Z_77d52a5d
report : C:\...\AQP-PYTHON\outputs\runs\20261018T074420Z_77d52a5d\reports\run_report.md
```

The first run forms the universe ("Formed universe ... with 27 members"). Every later run loads it.

## 11.4 Reading a run, in the right order

Always read the **manifest first**: it is the record. The narrative is a convenience built from it.

```powershell
$run = (Get-ChildItem .\outputs\runs -Directory | Sort-Object Name | Select-Object -Last 1).FullName
$m = Get-Content "$run\run_manifest.json" -Raw | ConvertFrom-Json
$m | Select-Object run_id, mode, decision_time, status, code_revision
$m.stages | Format-Table stage, status, seconds, @{n='reasons'; e={$_.reason_codes -join ','}}
$m.gates | Where-Object outcome -ne 'PASS' | Format-Table gate_id, outcome, subject, @{n='codes'; e={$_.reason_codes -join ','}}
```

```text
stage                  status seconds reasons
-----                  ------ ------- -------
stage_00_data          PASS      0.21
stage_01_universe      PASS     0.468
stage_02_forecasting   PASS      0.15
stage_02e_validation   PASS     1.248
stages_03_04_portfolio PASS     0.259
stage_05_sizing        PASS     0.013
safety                 PASS     0.005
stage_06_execution     PASS      0.02
stage_07_monitoring    PASS     0.054

gate_id outcome    subject              codes
------- -------    -------              -----
G2      QUARANTINE SYN17                DATA_EXTREME_RETURN
G2      QUARANTINE SYN25                DATA_STALE
G1      BLOCK      SYN13                EXIT-DIST
G1      BLOCK      SYN25                EXIT-SUSP
G1      BLOCK      SYN30                EXIT-DLST,EXIT-LIQ
G3      BLOCK      rolling_mean         MODEL_IS_BASELINE
G3      BLOCK      zero_excess          MODEL_IS_BASELINE
```

Then the report:

```powershell
code "$run\reports\run_report.md"          # Ctrl+Shift+V for the formatted preview
```

```markdown
## Measured results
- Data: 40 securities, 48219 price rows; quarantined this run: SYN17, SYN25.
- Universe ASX_20250630_v1: 27 members, 24 active; mandatory exits: SYN13, SYN25, SYN30.
- Model ridge_location_scale: pinball 0.0057, 90% coverage 88.15%, DM p-value 0.0217, net Sharpe 2.80.
- Model rolling_mean: pinball 0.0057, 90% coverage 89.71%, DM p-value 0.7979, net Sharpe -0.49.
- Model zero_excess: pinball 0.0057, 90% coverage 89.73%, DM p-value 1.0000, net Sharpe -0.49.
- Eligible after G3: ridge_location_scale.
- Leverage 0.800 (Kelly cap 2.587, volatility cap 1.250); gross exposure 40.17%; orders 7.
- Decisions: NO-TRADE 17, TRADE 7.
- Implementation shortfall 16.57 bps on 401179.53 traded.
```

And the plots (`explorer "$run\plots"`): the out-of-sample long-short curve per model, rolling 90% coverage against its band, the combined forecast fan (median, 50% and 90% intervals per security) and current-versus-target weights.

Where to find each design §16 record:

| Design record | Location in the run folder |
|---|---|
| Run/config | `run_manifest.json` (IDs, times, hash, revision) + `config_snapshot.yaml` |
| Data lineage | `run_manifest.json → lineage` (file hashes) + `data_lineage\` |
| Universe | `universe\selected_companies.csv`, `exclusions.csv`, `exits.csv`, `active_members.csv` |
| Models/validation | `validation\folds.csv`, `model_scores.csv`, `eligibility.csv`, `oos_forecasts.csv`, `combination_weights.json` |
| Portfolio/decision | `portfolio\target_weights.csv`, `risk_summary.json`, `sizing\position_report.csv`, `sizing_summary.json`, `order_requests.csv` |
| Safety | `safety\pre_trade_checks.csv` + gates in the manifest |
| Execution | `execution\fills.csv`, `reconciliation_breaks.csv`, `implementation_shortfall.json` |
| Outputs | `plots\*.png` + `plot_index.csv`, `monitoring\*`, `reports\run_summary.json`, `run_report.md`, `run.log` |

## 11.5 Failure drills: prove the controls work

Run each drill and confirm that what happens matches the design. These are the operational tests of roadmap phase 7.

**Drill 1: no edge means no trade.**

```powershell
python -m aqp_python.cli build-sample-data --no-signal
Remove-Item -Recurse -Force .\data\universes        # the market changed, so form a new universe
python -m aqp_python.cli run --as-of 2026-10-16
```

Expect `status : PASS`, with `stage_02e_validation PASS NO_ELIGIBLE_MODEL`. The report says *"Eligible after G3: none - the sleeve does not trade"*, the candidate is excluded with `MODEL_NO_SKILL, MODEL_NO_NET_VALUE, MODEL_OVERFIT_RISK`, and `order_requests.csv` is empty. **A healthy pipeline that refuses to trade is the most important result in this guide.** Restore the data:

```powershell
python -m aqp_python.cli build-sample-data
Remove-Item -Recurse -Force .\data\universes
```

**Drill 2: kill switch.**

```powershell
New-Item -ItemType Directory -Force .\outputs\state | Out-Null
'{"reason": "drill"}' | Set-Content .\outputs\state\KILL_SWITCH
python -m aqp_python.cli run --as-of 2026-10-16          # status : BLOCKED, safety BLOCKED KILL_SWITCH
Remove-Item .\outputs\state\KILL_SWITCH
```

There is no `stage_06_execution` entry in the manifest: nothing reached the broker. `$LASTEXITCODE` is `1`.

**Drill 3: broken data.** Duplicate one line of `data\samples\prices.csv` (copy a row in VS Code), then run. Expect `BLOCKED` at Stage 00 with `DATA_DUPLICATE`, and a manifest with exactly one stage. Rebuild the sample data afterwards.

**Drill 4: a stage that crashes.** Temporarily add `raise RuntimeError("drill")` as the first line of `stage_05_sizing\service.py: run_stage`. Run it. The run is `BLOCKED` with `STAGE_EXCEPTION`, the traceback is in `run.log`, every earlier artifact exists, and the report still appears, with the line *"Stage stage_05_sizing ended BLOCKED"* under failures. Remove the line.

## 11.6 Paper trading, one day at a time

`--mode paper` updates the paper book (`outputs\state\asx_interday\paper_book.json`), and runs must move forward in time. Simulate a week:

```powershell
Remove-Item -Recurse -Force .\outputs, .\data\universes -ErrorAction SilentlyContinue
foreach ($day in '2026-10-12','2026-10-13','2026-10-14','2026-10-15','2026-10-16') {
    python -m aqp_python.cli run --as-of $day --mode paper | Select-String '^status'
}
Get-Content .\outputs\state\asx_interday\paper_book.json
```

```text
status : PASS   (x5)
{
  "cash": 566522.98,
  "positions": { "SYN10": 2269, "SYN27": 2774, "SYN33": 1580, "SYN40": 3886, ... },
  "equity": { "2026-10-12": 999202.59, "2026-10-13": 1001950.80, "2026-10-14": 995839.88,
              "2026-10-15": 997814.75, "2026-10-16": 987334.65 }
}
```

Watch the reports change from day to day. From day 2 there are `HOLD` and `REDUCE` decisions because the book is no longer empty. On day 4 the leverage drops below 0.8 because the **drawdown multiplier** reacts to the NAV history (design §12). Running `2026-10-16` again is refused with `PAPER_OUT_OF_ORDER`.

Each run uses only data available at 17:00 on its own date (Stage 00's `as_of`), so this loop is an honest walk-forward of the whole system, not just the model.

## 11.7 Replay: prove a run reproduces

**File:** `scripts/replay_run.py`

```python
"""Replay a recorded run and check that it reproduces.

    python scripts/replay_run.py outputs/runs/<run_id>

1. Verifies the input files still have the SHA-256 hashes recorded in the manifest.
2. Re-runs the same sleeve and decision date in research mode (a NEW run folder;
   the original is never touched).
3. Compares book-independent outputs: model scores and combined forecasts.
"""
import json
import sys
from datetime import date
from hashlib import sha256
from pathlib import Path

import pandas as pd

from aqp_python.orchestration.orchestrator import run_pipeline

COMPARE = ("validation/model_scores.csv", "forecasts/combined_forecasts.csv")


def main(argv: list[str]) -> int:
    if len(argv) != 1:
        print(__doc__)
        return 2
    original = Path(argv[0])
    manifest = json.loads((original / "run_manifest.json").read_text(encoding="utf-8"))
    lineage = manifest["lineage"].get("stage_00_data", {})
    changed = [name for name, item in lineage.items()
               if not Path(item["path"]).is_file()
               or sha256(Path(item["path"]).read_bytes()).hexdigest() != item["sha256"]]
    if changed:
        print(f"CANNOT REPLAY: input files changed or missing since the run: {changed}")
        return 1
    as_of = date.fromisoformat(manifest["decision_time"][:10])
    replay = run_pipeline(Path("configs/pipeline.yaml"), manifest["sleeve"], as_of, Path.cwd(), "research")
    print(f"replay run: {replay.run_id} ({replay.status})")
    same = True
    for relative in COMPARE:
        left, right = original / relative, replay.run_dir / relative
        if not (left.is_file() and right.is_file()):
            print(f"  MISSING  {relative}")
            same = False
            continue
        a, b = pd.read_csv(left), pd.read_csv(right)
        drop = [c for c in ("decision_time", "model_version") if c in a]
        try:
            pd.testing.assert_frame_equal(a.drop(columns=drop), b.drop(columns=drop),
                                          check_exact=False, rtol=1e-9)
            print(f"  MATCH    {relative}")
        except AssertionError:
            print(f"  DIFFERS  {relative}")
            same = False
    if manifest["config_hash"] != json.loads((replay.run_dir / "run_manifest.json").read_text())["config_hash"]:
        print("  NOTE     configuration hash differs (YAML or --mode changed since the original run)")
    return 0 if same else 1


if __name__ == "__main__":
    raise SystemExit(main(sys.argv[1:]))
```

```powershell
python .\scripts\replay_run.py $run
```

```text
replay run: 20261018T073109Z_dc6ee40f (PASS)
  MATCH    validation/model_scores.csv
  MATCH    forecasts/combined_forecasts.csv
```

The replay first checks the input hashes recorded in the manifest. If the sample data has changed, it refuses to replay instead of pretending, because today's data under yesterday's run ID would be a false record. It then creates a **new** run and never writes into the original folder. Only book-independent outputs are compared, because target weights also depend on the paper book at the time.

## 11.8 Turn the architecture rules into tests

Two more permanent invariants:

**File:** `tests/invariants/test_hard_limits_independence.py`

```python
"""Architecture rules enforced by reading import statements (no extra tools needed)."""
import ast
from pathlib import Path

import pytest

pytestmark = pytest.mark.invariant
SRC = Path(__file__).resolve().parents[2] / "src" / "aqp_python"

RULES = {
    "safety": ["aqp_python.stages", "aqp_python.reporting", "aqp_python.compiler", "aqp_python.orchestration"],
    "contracts": ["aqp_python.gates", "aqp_python.stages", "aqp_python.safety", "aqp_python.storage",
                  "aqp_python.orchestration", "aqp_python.connectors"],
    "config": ["aqp_python.gates", "aqp_python.stages", "aqp_python.safety", "aqp_python.storage"],
    "gates": ["aqp_python.stages", "aqp_python.safety", "aqp_python.orchestration", "aqp_python.storage"],
    "stages/stage_01_universe": ["aqp_python.stages.stage_02_forecasting"],
    "reporting": ["aqp_python.connectors.broker", "aqp_python.stages.stage_06_execution", "aqp_python.safety"],
    "compiler": ["aqp_python.connectors.broker", "aqp_python.stages.stage_06_execution", "aqp_python.safety"],
}


def imports_of(path: Path) -> set[str]:
    tree = ast.parse(path.read_text(encoding="utf-8"))
    names = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            names.update(alias.name for alias in node.names)
        elif isinstance(node, ast.ImportFrom) and node.module:
            names.add(node.module)
    return names


@pytest.mark.parametrize("package", sorted(RULES))
def test_forbidden_imports(package):
    violations = []
    for path in (SRC / package).rglob("*.py"):
        for name in imports_of(path):
            if any(name == f or name.startswith(f + ".") for f in RULES[package]):
                violations.append(f"{path.relative_to(SRC)} imports {name}")
    assert not violations, "\n".join(violations)


def test_only_stage_06_reaches_the_broker():
    allowed = {"stages/stage_06_execution", "connectors/broker"}
    offenders = [str(p.relative_to(SRC)) for p in SRC.rglob("*.py")
                 if any(n.startswith("aqp_python.connectors.broker") for n in imports_of(p))
                 and not any(str(p.relative_to(SRC)).startswith(a) for a in allowed)]
    assert not offenders, offenders
```

This test reads the `import` statements of every file with Python's `ast` module, so it needs no extra tools. It fails the build if `safety` imports a stage, if `contracts` imports anything else from the package, if Stage 01 imports forecasting, or if anything other than Stage 06 touches the broker. The same rules are written for the `import-linter` tool:

**File:** `.importlinter`

```ini
[importlinter]
root_package = aqp_python

[importlinter:contract:safety-independent]
name = Safety must not depend on models, portfolio, sizing or reporting
type = forbidden
source_modules =
    aqp_python.safety
forbidden_modules =
    aqp_python.stages
    aqp_python.reporting
    aqp_python.compiler
    aqp_python.orchestration

[importlinter:contract:contracts-are-leaves]
name = Contracts and config import nothing else from aqp_python
type = forbidden
source_modules =
    aqp_python.contracts
    aqp_python.config
forbidden_modules =
    aqp_python.gates
    aqp_python.storage
    aqp_python.stages
    aqp_python.safety
    aqp_python.orchestration
    aqp_python.reporting
    aqp_python.compiler
    aqp_python.connectors

[importlinter:contract:universe-not-forecasting]
name = Universe selection must not use forecasts
type = forbidden
source_modules =
    aqp_python.stages.stage_01_universe
forbidden_modules =
    aqp_python.stages.stage_02_forecasting

[importlinter:contract:reporting-read-only]
name = Reporting never touches the broker or execution
type = forbidden
source_modules =
    aqp_python.reporting
    aqp_python.compiler
forbidden_modules =
    aqp_python.connectors.broker
    aqp_python.stages.stage_06_execution
    aqp_python.safety
```

```powershell
lint-imports
```

```text
Safety must not depend on models, portfolio, sizing or reporting KEPT
Contracts and config import nothing else from aqp_python KEPT
Universe selection must not use forecasts KEPT
Reporting never touches the broker or execution KEPT
Contracts: 4 kept, 0 broken.
```

**File:** `tests/invariants/test_universe_fixed_membership.py`

```python
import pandas as pd
import pytest

from aqp_python.orchestration.orchestrator import run_pipeline
from tests.conftest import AS_OF

pytestmark = pytest.mark.invariant


def test_membership_is_frozen_and_exits_never_re_enter(project):
    config = project / "configs" / "pipeline.yaml"
    first = run_pipeline(config, "asx_interday", AS_OF, project, "research")
    second = run_pipeline(config, "asx_interday", AS_OF, project, "research")
    members = [pd.read_csv(r.run_dir / "universe" / "selected_companies.csv") for r in (first, second)]
    pd.testing.assert_frame_equal(*members)
    active = set(pd.read_csv(second.run_dir / "universe" / "active_members.csv")["security_id"])
    exits = set(pd.read_csv(second.run_dir / "universe" / "exits.csv")["security_id"])
    assert exits and not active & exits
    # The delisted-before-formation name is in the history but was never admitted.
    decisions = pd.read_csv(second.run_dir / "universe" / "exclusions.csv").set_index("security_id")
    assert "NOT_LISTED" in decisions.loc["SYN05", "reason_codes"]
```

And the end-to-end tests, which are the drills from 11.5 written as code:

**File:** `tests/integration/test_end_to_end.py`

```python
"""Full synthetic runs through the orchestrator."""
import json

import pandas as pd
import pytest

from aqp_python.orchestration.orchestrator import run_pipeline
from aqp_python.safety.kill_switch import engage
from aqp_python.stages.stage_00_data.synthetic_market import write_sample_data
from tests.conftest import AS_OF

pytestmark = pytest.mark.batch07
EXPECTED = ["run_manifest.json", "config_snapshot.yaml", "data_lineage/data_quality_summary.csv",
            "universe/selected_companies.csv", "forecasts/combined_forecasts.csv",
            "validation/model_scores.csv", "portfolio/target_weights.csv", "sizing/position_report.csv",
            "safety/pre_trade_checks.csv", "execution/fills.csv", "monitoring/alerts.json",
            "plots/plot_index.csv", "reports/run_summary.json", "reports/run_report.md"]


def run(project, mode="research", as_of=AS_OF):
    return run_pipeline(project / "configs" / "pipeline.yaml", "asx_interday", as_of, project, mode)


def test_research_run_passes_and_writes_every_artifact(project):
    outcome = run(project)
    assert outcome.status == "PASS"
    for relative in EXPECTED:
        assert (outcome.run_dir / relative).is_file(), relative
    manifest = json.loads((outcome.run_dir / "run_manifest.json").read_text())
    assert [s["stage"] for s in manifest["stages"]][0] == "stage_00_data"
    assert manifest["report"]["status"] == "CREATED"
    assert not (project / "outputs" / "state" / "asx_interday" / "paper_book.json").exists()


def test_paper_runs_update_the_book_and_must_move_forward(project):
    first = run(project, "paper", pd.Timestamp("2026-10-15").date())
    second = run(project, "paper", AS_OF)
    again = run(project, "paper", AS_OF)
    book = json.loads((project / "outputs/state/asx_interday/paper_book.json").read_text())
    assert first.status == second.status == "PASS"
    assert list(book["equity"]) == ["2026-10-15", "2026-10-16"]
    assert again.status == "BLOCKED"


def test_no_signal_means_no_trade(project):
    write_sample_data(project / "data" / "samples", quality_alpha=0.0, reversal_strength=0.0)
    outcome = run(project)
    scores = pd.read_csv(outcome.run_dir / "validation" / "eligibility.csv")
    orders = (outcome.run_dir / "sizing" / "order_requests.csv").read_text().strip()
    assert outcome.status == "PASS"                     # healthy pipeline, no edge
    assert (scores["outcome"] == "BLOCK").all()
    assert orders == ""


def test_kill_switch_blocks_the_run_before_execution(project):
    engage(project / "outputs" / "state" / "KILL_SWITCH", "test")
    outcome = run(project)
    manifest = json.loads((outcome.run_dir / "run_manifest.json").read_text())
    stages = {s["stage"]: s["status"] for s in manifest["stages"]}
    assert outcome.status == "BLOCKED" and stages["safety"] == "BLOCKED"
    assert "stage_06_execution" not in stages


def test_broken_data_blocks_at_stage_00(project):
    prices = project / "data" / "samples" / "prices.csv"
    frame = pd.read_csv(prices)
    pd.concat([frame, frame.head(1)]).to_csv(prices, index=False)    # duplicate key
    outcome = run(project)
    manifest = json.loads((outcome.run_dir / "run_manifest.json").read_text())
    assert outcome.status == "BLOCKED" and len(manifest["stages"]) == 1
    assert any("DATA_DUPLICATE" in g["reason_codes"] for g in manifest["gates"])
```

## 11.9 The whole suite, and code style

```powershell
python -m pytest -q
ruff check src tests scripts
```

```text
........................................................................ [ 69%]
...............................                                          [100%]
103 passed in 45s
All checks passed!
```

| Command | Runs |
|---|---|
| `python -m pytest -m batch01` … `-m batch07` | one roadmap batch |
| `python -m pytest -m invariant` | the 15 design invariants |
| `python -m pytest tests\integration\test_stage_handoffs.py -s` | every stage checkpoint, with output |
| `python -m pytest -m "not batch07"` | everything except the slower end-to-end runs |

## 11.10 Is a stage "complete"?

Design §17: *"A stage is complete only when its input/output schema, unit and failure tests, point-in-time handling, versioned configuration, logs and reason codes, reproducible outputs, baseline/invariants and downstream failure actions are documented."* For the spine you have built:

| Criterion | Where it is satisfied |
|---|---|
| Input/output schema | `contracts/*` hand-off objects; `FORECAST_COLUMNS`; `require_columns` |
| Unit and failure tests | `tests\unit\*` (every gate has failing-case tests) |
| Point-in-time handling | `as_of`, `latest_vintage`, `merge_asof`, `training_rows`, purged folds; `test_no_lookahead` |
| Versioned configuration | YAML + `config_hash` + `config_snapshot.yaml` |
| Logs and reason codes | `run.log`, `ReasonCode`, gate results in the manifest |
| Reproducible outputs | immutable run folders, input hashes, `replay_run.py` |
| Baselines/invariants | `zero_excess`, `rolling_mean`; `tests\invariants\*` |
| Downstream failure actions | `StageRunner` stop-on-block, `_hold`, NO-TRADE path, kill switch, reconciliation block |

Copy this table into `docs\readiness\stage_completion_checklist.md` and add a row each time a stage is extended.

## Checkpoint

- [ ] A research run is `PASS` with 7 orders; the four drills behave as described.
- [ ] Five consecutive paper days run; re-running a day is refused.
- [ ] `replay_run.py` prints `MATCH` twice.
- [ ] `python -m pytest -q` → 103 passed; `lint-imports` → 4 kept; `ruff check` passes.

```powershell
git add .
git commit -m "Orchestrator, full runs, paper loop, replay, architecture invariants and end-to-end tests"
git tag v0.1-spine
```

You now have a working spine. Next: [Part 12 — extending it toward production](12_extending_to_production.md).
