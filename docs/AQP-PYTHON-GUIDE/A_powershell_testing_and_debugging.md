# Appendix A — PowerShell, testing, outputs and the error playbook

Keep this open while you work. Every command runs from the `AQP-PYTHON` folder with `.venv` active.

## A.1 Start of every session

```powershell
Set-Location $HOME\Documents\AQP-PYTHON
.\.venv\Scripts\Activate.ps1                    # blocked? Set-ExecutionPolicy -Scope Process RemoteSigned
python .\scripts\validate_environment.py        # READY?
git status --short                              # anything uncommitted from last time?
```

## A.2 The component loop

Each part shows files as numbered **components** ("component 2 of 4: …"). Work one component at a time:

```text
type one component → read its "What it does / Why" → run the try-it (if any) → run its unit test
→ when the file is complete: run the stage checkpoint → run the batch → commit
```

```powershell
python -m pytest tests\unit\test_news_timing.py -q                                              # 1. narrow
python -m pytest tests\integration\test_stage_handoffs.py -k 02d -s --basetemp=build\pytest     # 2. stage
python -m pytest -m batch05 -q                                                                  # 3. batch
python -m pytest -q; ruff check src tests scripts; lint-imports                                 # 4. everything, before commit
git add .; git commit -m "Describe the one capability you added"
```

**Try-its.** A `python` block followed by an output block is a *try-it*: save it as `build\tryit.py` (or paste it into `python` at the prompt) and run it from the project folder. Your output should match the guide's exactly, because every try-it uses fixed seeds. If it does not, the component you just typed differs from the guide's.

## A.3 pytest cheat sheet

| Goal | Command |
|---|---|
| One test function | `python -m pytest tests\unit\test_config_loader.py::test_unknown_sleeve_is_rejected` |
| Tests whose name contains a word | `python -m pytest -k "duplicate or unlinked"` |
| One roadmap phase | `python -m pytest -m batch05` (phases `batch01` … `batch09`) |
| The design invariants / the extensions | `-m invariant` / `-m extension` |
| Everything except the slower full runs | `python -m pytest -m "not batch09"` |
| Stop at the first failure | `python -m pytest -x` |
| Rerun only what failed last time | `python -m pytest --lf` |
| See `print()` output | `-s` |
| Longer / shorter tracebacks | `-vv --tb=long` / `--tb=short` |
| Drop into the debugger on failure | `--pdb` (then `p variable`, `l`, `c`, `q`) |
| Keep temporary folders where you can find them | `--basetemp=build\pytest` (create `build` once) |
| Slowest tests | `--durations=5` |
| Only list what would run | `--collect-only -q` |

**Exit codes:** `0` all passed · `1` some failed · `2` interrupted or collection error · `5` no tests collected.

| Marker | Roadmap phase | Built in |
|---|---|---|
| `batch01` | contracts, configuration, storage, data foundation, Stage 00, G2 | Parts 02–04 |
| `batch02` | Stage 01, G1, coverage tiers | Part 05 |
| `batch03` | data request, company reference, X_core | Part 06 |
| `batch04` | validation framework, forecast contract, arms, baselines | Parts 07–08 |
| `batch05` | Stage 02D news engine | Part 09 |
| `batch06` | Stage 02E paired evaluation, G3, combination | Part 10 |
| `batch07` | Stages 03–05, G4–G6 | Part 11 |
| `batch08` | safety, execution | Part 12 |
| `batch09` | monitoring, reporting, full runs, fallback | Parts 12–13 |
| `invariant` | design invariants | every part |
| `extension` | tested modules not wired into the default run | Part 14 |

## A.4 Running the pipeline

```powershell
python -m aqp_python.cli validate-config
python scripts\build_sample_data.py                        # --no-signal / --no-news-signal for the drills
python scripts\register_experiment.py                      # register experiments before any result
python scripts\run_pipeline.py --as-of 2026-10-16          # research (book untouched)
python scripts\run_pipeline.py --as-of 2026-10-16 --mode paper
python scripts\replay_run.py .\outputs\runs\<run_id>
$LASTEXITCODE                                              # 0 PASS/DEGRADED, 1 BLOCKED, 2 bad input
```

## A.5 Where everything is written

| Path | What | In Git? | Safe to delete? |
|---|---|---|---|
| `data\samples\*.csv` | synthetic market (screening fields, detailed fields, news, coverage) | yes | yes, then rebuild with `build_sample_data.py` |
| `data\universes\<id>\` | frozen universe versions, tiers and exit logs | no | only for a deliberate fresh start, because it re-forms the universe |
| `data\data_requests\<request_id>.json` | frozen Stage 2 data requests | no | yes, together with `data\universes` |
| `outputs\runs\<run_id>\` | one immutable folder per run | no | yes, old runs only |
| `outputs\state\asx_interday\paper_book.json` | paper positions, cash, NAV history | no | resets paper trading to day 0 |
| `outputs\state\trial_registry.jsonl` | every model configuration evaluated | no | **no**: deleting it hides past trials from the DSR |
| `outputs\state\experiment_registry.jsonl` | every registered experiment and its spec hash | no | **no**: deleting it lets a changed experiment pass as the same one |
| `outputs\state\KILL_SWITCH` | kill switch (exists = engaged) | no | only after reviewing `incidents.jsonl` |
| `outputs\state\incidents.jsonl` | incident log | no | no |
| `build\pytest\` | pytest temporary folders (with `--basetemp`) | no | yes |

Fresh start for the synthetic market (keeps both registries):

```powershell
Remove-Item -Recurse -Force .\outputs\runs, .\outputs\state\asx_interday, .\data\universes, .\data\data_requests -ErrorAction SilentlyContinue
```

## A.6 Inspecting a run from PowerShell

```powershell
$run = (Get-ChildItem .\outputs\runs -Directory | Sort-Object Name | Select-Object -Last 1).FullName
$m = Get-Content "$run\run_manifest.json" -Raw | ConvertFrom-Json

$m | Select-Object run_id, mode, decision_time, status, config_hash, code_revision
$m.stages | Format-Table stage, status, seconds, @{n='reasons'; e={$_.reason_codes -join ','}}
$m.gates  | Where-Object outcome -ne 'PASS' | Format-Table gate_id, outcome, subject, @{n='codes'; e={$_.reason_codes -join ','}}
$m.records.experiments | Format-List                       # paired evidence and the challenger decision
$m.records.news_features | Format-List                     # news version, coverage, latency, stories by status
$m.warnings
$m.report

Import-Csv "$run\validation\experiments.csv" | Format-List
Import-Csv "$run\features\news_story_log.csv" | Where-Object status -ne 'OK' | Format-Table story_id, security_id, status, extraction_problems
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
- **Stage 02D step by step:** each line of `run_stage` (Part 09) returns the story table with more columns. Put a breakpoint after any line and look at `stories["status"].value_counts()`.
- **More logging for one run:** in `cli.py`, `configure_logging("DEBUG")`. Every stage logs to `run.log` in its run folder.

## A.8 Error playbook

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'aqp_python'` | wrong interpreter, or the package is not installed in editable mode | `python -c "import sys; print(sys.executable)"` must point into `.venv`; then `python -m pip install -e ".[dev]"` |
| `ZoneInfoNotFoundError: 'No time zone found with key Australia/Sydney'` | `tzdata` missing (Windows has no IANA database) | `python -m pip install tzdata` |
| `Activate.ps1 cannot be loaded because running scripts is disabled` | execution policy | `Set-ExecutionPolicy -Scope Process RemoteSigned`, or call `.\.venv\Scripts\python.exe` directly |
| `Configuration INVALID: ...` | a YAML key was removed or misspelt, or a v14 rule was broken (news in screening, overlapping core/news features, `approved_arm` not an arm) | the message names the key; YAML is indentation-sensitive (spaces, never tabs) |
| `FileNotFoundError: Sample data missing ...` | `data\samples` empty | `python scripts\build_sample_data.py` |
| `PermissionError: [Errno 13] ... prices.csv` | the CSV is open in Excel, which locks files on Windows | close Excel, then rerun |
| Run `BLOCKED` at Stage 00 with `DATA_STALE` | `--as-of` is a weekend, a holiday, or after the last data date | use a trading day ≤ the last date in `benchmark.csv` |
| Run `BLOCKED` with `DATA_DUPLICATE` / `DATA_LOOKAHEAD` | the input data was edited or a source changed | check `data_lineage\data_quality_summary.csv` and the G2 gate entries |
| `DataRequestError: no G1-passed universe` in Stage 2 | Stage 2 was called without a universe that passed G1 (for example from your own script) | run Stage 01 first; detailed data is never requested before G1 |
| `stage_02d_news_features DEGRADED NEWS_UNAVAILABLE` | `news.csv` or `news_coverage.csv` missing, or feature QC failed | the approved fallback; read the stage's `records.news_features.reason` |
| `stage_02d_news_features BLOCKED NEWS_UNAVAILABLE` | the same, but `news_features_unavailable` is not in `fallbacks.yaml` | restore the feed, or approve the fallback deliberately |
| Challenger always `CHALLENGER_UNAVAILABLE` | news unavailable this run, or `extractor` set to something not wired | check `news_health.json` and `news_features.yaml` |
| `ExperimentRegistryError: ... changed after registration` | an experiment's spec (features, model, horizon) changed under the same id | give it a new id; a changed experiment is a new experiment |
| `ARM_PARITY_BROKEN` in G3 | the arms differ by more than the registered news features | fix `experiments.yaml` or `forecast_models.yaml`; never loosen the check |
| `EXPERIMENT_NOT_REGISTERED` | an enabled candidate has no experiment | register it, or set `enabled: false` |
| News features are 0 where you expected NaN | coverage mask not applied | run `tests\invariants\test_missing_news_not_zero_filled.py`; check `apply_coverage` |
| `stage_06_execution BLOCKED PAPER_OUT_OF_ORDER` | a paper run for a date not after the book's last date | paper runs must move forward; use research mode to re-examine a past day |
| `safety BLOCKED KILL_SWITCH` | `outputs\state\KILL_SWITCH` exists | read `incidents.jsonl`, fix the cause, then delete the file |
| `RECONCILIATION_FAILED` and the kill switch engaged | the book does not match the expected positions | inspect `execution\reconciliation_breaks.csv`; never delete the incident |
| Everything `PASS` but no orders | no approved slot eligible (`NO_ELIGIBLE_MODEL`), all names in the no-trade band, or below the cost threshold | read `validation\experiments.csv` and `eligibility.csv`, then `sizing\position_report.csv` |
| `OPT_INFEASIBLE` | constraints contradict each other (holdings above a new cap with turnover frozen) | read `portfolio\risk_summary.json`; change config deliberately and record it in the decision log |
| A forecast or challenger looks too good to be true | look-ahead: a negative `shift`, labels joined on the wrong date, news counted before `processed_ts`, features fitted on the full sample | run `python -m pytest -m invariant`; check `training_rows`, `label_end` and `usable_day_index` |
| pytest **collection** error | syntax error or failing import in a test or conftest | run the single file with `-x --tb=short` and fix the first traceback |
| `ImportError: cannot import name ...` | a component not typed yet, a renamed function, or a circular import | compare with the component list at the top of the part |
| `lint-imports` reports BROKEN | a forbidden import was added (e.g. Stage 01 → Stage 02D, forecasting → news engine) | move the shared type into `contracts`; never relax the contract |
| `report.status = FAILED` in the manifest | the narrative contained a number not in `run_summary.json`, or a plot input was malformed | read `run.log`; the structured outputs are still valid |
| `FileExistsError: universe ... already exists` | trying to save a version that exists | intended: versions are immutable; change `version_label` for a deliberate re-formation |
| Try-it output differs from the guide | a component typed differently, or sample data built with other flags | rebuild the sample data with no flags; compare the component line by line |
| `git` warns "LF will be replaced by CRLF" | Windows line endings | harmless; optionally `git config core.autocrlf true` |

## A.9 What "passing" means

| Evidence | Says | Does **not** say |
|---|---|---|
| A unit test passes | the component behaves as specified on that input | the model is good |
| A batch passes | that roadmap phase's capabilities work | the system is ready |
| A research run is `PASS` | all checks worked on that date's data | the strategy is profitable |
| G3 eligible | out-of-sample, after-cost evidence on *this* data | it will work tomorrow, or on real data |
| `SHADOW_CANDIDATE` | the news challenger beat its own baseline on matched rows | it may trade (that needs G7) |
| Weeks of clean paper trading | operational readiness | permission to trade live (that needs deployment sign-off) |
