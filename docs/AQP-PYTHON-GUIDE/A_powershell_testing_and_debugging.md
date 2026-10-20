# Appendix A — PowerShell, testing, outputs and the error playbook

Keep this open while you work. Every command runs from the `AQP-PYTHON` folder with `.venv` active.

## A.1 Start of every session

```powershell
Set-Location $HOME\Documents\AQP-PYTHON
.\.venv\Scripts\Activate.ps1                    # blocked? Set-ExecutionPolicy -Scope Process RemoteSigned
python .\scripts\validate_environment.py        # READY?
git status --short                              # anything uncommitted from last time?
```

## A.2 The daily development loop

```text
edit one function → run its unit test → run the stage checkpoint → run the batch → commit
```

```powershell
python -m pytest tests\unit\test_universe.py -q                                   # 1. narrow
python -m pytest tests\integration\test_stage_handoffs.py -k stage_01 -s --basetemp=build\pytest   # 2. stage
python -m pytest -m batch02 -q                                                    # 3. batch
python -m pytest -q; ruff check src tests scripts                                 # 4. everything, before commit
git add .; git commit -m "Describe the one capability you added"
```

## A.3 pytest cheat sheet

| Goal | Command |
|---|---|
| One test function | `python -m pytest tests\unit\test_config.py::test_live_mode_is_not_accepted` |
| Tests whose name contains a word | `python -m pytest -k "split or stale"` |
| One roadmap batch / the invariants | `python -m pytest -m batch05` / `-m invariant` |
| Everything except slow end-to-end runs | `python -m pytest -m "not batch07"` |
| Stop at the first failure | `python -m pytest -x` |
| Rerun only what failed last time | `python -m pytest --lf` |
| See `print()` output | `-s` |
| Longer / shorter tracebacks | `-vv --tb=long` / `--tb=short` |
| Drop into the debugger on failure | `--pdb` (then `p variable`, `l`, `c`, `q`) |
| Keep temporary folders where you can find them | `--basetemp=build\pytest` (create `build` once) |
| Slowest tests | `--durations=5` |
| Only list what would run | `--collect-only -q` |

**Exit codes:** `0` all passed · `1` some failed · `2` interrupted or collection error · `5` no tests collected.

## A.4 Running the pipeline

```powershell
python -m aqp_python.cli validate-config
python -m aqp_python.cli build-sample-data              # add --no-signal for the NO-TRADE drill
python -m aqp_python.cli run --as-of 2026-10-16         # research (book untouched)
python -m aqp_python.cli run --as-of 2026-10-16 --mode paper
python .\scripts\replay_run.py .\outputs\runs\<run_id>
$LASTEXITCODE                                           # 0 PASS/DEGRADED, 1 BLOCKED, 2 bad input
```

## A.5 Where everything is written

| Path | What | In Git? | Safe to delete? |
|---|---|---|---|
| `data\samples\*.csv` | synthetic market (fixture) | yes | yes, then rebuild with `build-sample-data` |
| `data\universes\<id>\` | frozen universe versions + exit logs | no | only for a deliberate fresh start, because it re-forms the universe |
| `outputs\runs\<run_id>\` | one immutable folder per run | no | yes, old runs only |
| `outputs\state\asx_interday\paper_book.json` | paper positions, cash, NAV history | no | resets paper trading to day 0 |
| `outputs\state\trial_registry.jsonl` | every model configuration evaluated | no | **no**: deleting it hides past trials from the DSR |
| `outputs\state\KILL_SWITCH` | kill switch (exists = engaged) | no | only after reviewing `incidents.jsonl` |
| `outputs\state\incidents.jsonl` | incident log | no | no |
| `build\pytest\` | pytest temporary folders (with `--basetemp`) | no | yes |

Fresh start for the synthetic market (keeps the trial registry):

```powershell
Remove-Item -Recurse -Force .\outputs\runs, .\outputs\state\asx_interday, .\data\universes -ErrorAction SilentlyContinue
```

## A.6 Inspecting a run from PowerShell

```powershell
$run = (Get-ChildItem .\outputs\runs -Directory | Sort-Object Name | Select-Object -Last 1).FullName
$m = Get-Content "$run\run_manifest.json" -Raw | ConvertFrom-Json

$m | Select-Object run_id, mode, decision_time, status, config_hash, code_revision
$m.stages | Format-Table stage, status, seconds, @{n='reasons'; e={$_.reason_codes -join ','}}
$m.gates  | Where-Object outcome -ne 'PASS' | Format-Table gate_id, outcome, subject, @{n='codes'; e={$_.reason_codes -join ','}}
$m.warnings
$m.report

Import-Csv "$run\sizing\position_report.csv" | Format-Table security_id, current_weight, target_weight, action, reason_code
Import-Csv "$run\validation\model_scores.csv" | Out-GridView        # sortable window
code "$run\reports\run_report.md"                                    # Ctrl+Shift+V to preview
explorer "$run\plots"
Get-Content "$run\run.log" -Tail 30
```

Compare two runs' decisions:

```powershell
$a, $b = (Get-ChildItem .\outputs\runs -Directory | Sort-Object Name | Select-Object -Last 2).FullName
Compare-Object (Import-Csv "$a\sizing\order_requests.csv") (Import-Csv "$b\sizing\order_requests.csv") -Property security_id, side, quantity
```

## A.7 Debugging in VS Code

- **Breakpoint + F5:** click left of a line number, choose **AQP: research run (2026-10-16)** in the Run and Debug panel, press F5. Inspect DataFrames in the Variables pane (right-click → *View Value in Data Viewer*).
- **Debug one test:** open the test file, choose **AQP: debug current test file**, press F5.
- **Quick stop without the debugger:** put `breakpoint()` on a line and run with `python -m pytest -s ...`. You get a `(Pdb)` prompt there.
- **More logging for one run:** in `cli.py`, `configure_logging("DEBUG")`. Every stage logs to `run.log` in its run folder.

## A.8 Error playbook

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'aqp_python'` | wrong interpreter, or the package is not installed in editable mode | `python -c "import sys; print(sys.executable)"` must point into `.venv`; then `python -m pip install -e ".[dev]"` |
| `ZoneInfoNotFoundError: 'No time zone found with key Australia/Sydney'` | `tzdata` missing (Windows has no IANA database) | `python -m pip install tzdata` |
| `Activate.ps1 cannot be loaded because running scripts is disabled` | execution policy | `Set-ExecutionPolicy -Scope Process RemoteSigned`, or call `.\.venv\Scripts\python.exe` directly |
| `Configuration INVALID: 'x' is missing keys` | a YAML key was removed or misspelt | restore it; YAML is indentation-sensitive (spaces, never tabs) |
| `FileNotFoundError: Sample data missing ... Run: python scripts/build_sample_data.py` | `data\samples` empty | `python -m aqp_python.cli build-sample-data` |
| `PermissionError: [Errno 13] ... prices.csv` | the CSV is open in Excel, which locks files on Windows | close Excel, then rerun |
| Run `BLOCKED` at Stage 00 with `DATA_STALE` "no benchmark data for …" | `--as-of` is a weekend, a holiday, or after the last data date | use a trading day ≤ the last date in `benchmark.csv` |
| Run `BLOCKED` with `DATA_DUPLICATE` / `DATA_LOOKAHEAD` | the input data was edited or a source changed | check `data_lineage\data_quality_summary.csv` and the G2 gate entries |
| `stage_06_execution BLOCKED PAPER_OUT_OF_ORDER` | a paper run for a date not after the book's last date | paper runs must move forward; use research mode to re-examine a past day |
| `safety BLOCKED KILL_SWITCH` | `outputs\state\KILL_SWITCH` exists | read `incidents.jsonl`, fix the cause, then delete the file |
| `RECONCILIATION_FAILED` and the kill switch engaged | the book does not match the expected positions | inspect `execution\reconciliation_breaks.csv`; never delete the incident |
| Everything `PASS` but no orders | no model passed G3 (`NO_ELIGIBLE_MODEL`), all names inside the no-trade band, or below the cost threshold | read `validation\eligibility.csv`, then `sizing\position_report.csv` reason codes |
| `OPT_INFEASIBLE` | constraints contradict each other (e.g. holdings above a new cap with turnover frozen) | read `portfolio\risk_summary.json`; adjust config deliberately and record it in the decision log |
| A forecast looks too good to be true | look-ahead: a negative `shift`, labels joined on the wrong date, features fitted on the full sample | run `python -m pytest -m invariant`; check `training_rows` and `label_end` |
| pytest **collection** error | syntax error or failing import in a test or conftest | run the single file with `-x --tb=short` and fix the first traceback |
| `ImportError: cannot import name ...` | renamed function, or a circular import from a low-level module | low-level modules (`contracts`, `config`) must never import higher ones |
| `lint-imports` reports BROKEN | a forbidden import was added (e.g. safety → stages) | move the shared type into `contracts`; never relax the contract |
| `report.status = FAILED` in the manifest | the narrative contained a number not in `run_summary.json`, or a plot input was malformed | read `run.log`; the structured outputs are still valid |
| `FileExistsError: universe ... already exists` | trying to save a version that exists | intended: versions are immutable; change `version_label` for a deliberate re-formation |
| `git` warns "LF will be replaced by CRLF" | Windows line endings | harmless; optionally `git config core.autocrlf true` |
| Plots folder empty in a test run | matplotlib backend | the code forces `Agg`; check `plot_index.csv` for the per-plot error |

## A.9 What "passing" means

| Evidence | Says | Does **not** say |
|---|---|---|
| A unit test passes | the function behaves as specified on that input | the model is good |
| A batch passes | that roadmap phase's capabilities work | the system is ready |
| A research run is `PASS` | all checks worked on that date's data | the strategy is profitable |
| G3 eligible | out-of-sample, after-cost evidence on *this* data | it will work tomorrow, or on real data |
| Weeks of clean paper trading | operational readiness | permission to trade live (that needs phase 8 sign-off) |
