# AQP-PYTHON — Python Repository Overview

**Project:** AQP — Advanced Quantitative Pipeline  
**Repository:** `AQP-PYTHON`  
**Primary language:** Python  
**Companion language:** R, kept separate and called through controlled adapters where required  
**Intended environment:** Windows, Visual Studio Code, Git and a project-specific Python virtual environment  
**Status:** Proposed repository layout and implementation guide

---

## 1. Purpose

`AQP-PYTHON` contains the Python implementation of the Advanced Quantitative Pipeline (AQP). It organises the workflow from market-data extraction and company selection through forecasting, portfolio construction, position sizing, execution controls, monitoring, plotting and automated run reporting.

This document defines the **target repository structure**. A path shown below is a planned location until its script has actually been implemented and tested. The presence of a file in this tree does not imply that the corresponding model is complete, validated or approved for live trading.

### Core design rules

1. **One orchestrator:** `aqp_orchestrator.py` coordinates pipeline stages; individual stage scripts should not independently launch the entire workflow.
2. **One common data contract:** stage outputs use documented schemas, timestamps, identifiers, units and status fields.
3. **Point-in-time integrity:** features and selection decisions may use only information available at the decision timestamp.
4. **Selection is not a trading signal:** Stage 1 forms a research universe; forecasting and portfolio stages determine whether an opportunity is worth taking.
5. **Risk controls can veto:** safety checks and hard limits cannot be overridden by a model score, optimiser or narrative report.
6. **Reproducible runs:** every run records its run ID, code/configuration versions, data references, model status and outputs.
7. **Research before deployment:** backtests, paper trading and live deployment have separate readiness gates.

## 2. Naming convention

| Item | Convention | Example |
|---|---|---|
| Repository folder | `AQP-PYTHON` | `AQP-PYTHON/` |
| Python package | `aqp_python` | `src/aqp_python/` |
| Custom Python scripts | Prefix with `aqp_` | `aqp_stage_01_universe_selection.py` |
| Stage folders | Include AQP package path and stage number | `stages/stage_01_universe/` |
| Configuration files | Prefix with `aqp_`, use YAML where suitable | `aqp_risk_limits.yaml` |
| Python tests | `test_*.py` | `test_aqp_data_validator.py` |
| Test batches | Number and name by pipeline capability | `test_batch_03_universe/` |
| Mermaid diagrams | Prefix with `aqp_`, use `.mmd` | `aqp_stage_01_universe.mmd` |
| Generated run folders | Use a unique run ID, not a fixed overwritten name | `outputs/runs/<run_id>/` |

Use readable names rather than forcing `AQP` into every standard Python filename. The package and repository already identify the project; custom modules and scripts should use the `aqp_` prefix consistently.

## 3. Repository tree

```text
AQP-PYTHON/
├── README.md
├── PY REPOSITORY OVERVIEW.md
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── .gitignore
├── .env.example
├── .pre-commit-config.yaml
│
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   └── tasks.json
│
├── configs/
│   ├── aqp_pipeline.yaml
│   ├── aqp_markets.yaml
│   ├── aqp_data_sources.yaml
│   ├── aqp_universe_rules.yaml
│   ├── aqp_forecast_models.yaml
│   ├── aqp_validation.yaml
│   ├── aqp_risk_limits.yaml
│   ├── aqp_execution.yaml
│   └── aqp_reporting.yaml
│
├── docs/
│   ├── architecture/
│   │   ├── aqp_system_architecture.md
│   │   ├── aqp_data_dictionary.md
│   │   └── aqp_model_registry.md
│   ├── methodology/
│   │   ├── aqp_stage_00_data.md
│   │   ├── aqp_stage_01_universe.md
│   │   ├── aqp_stage_02_forecasting.md
│   │   ├── aqp_stages_03_04_portfolio.md
│   │   ├── aqp_stage_05_sizing.md
│   │   ├── aqp_stage_06_execution.md
│   │   └── aqp_stage_07_monitoring.md
│   └── diagrams/
│       ├── aqp_pipeline_overview.mmd
│       ├── aqp_stage_00_data.mmd
│       ├── aqp_stage_01_universe.mmd
│       ├── aqp_stage_02_forecasting.mmd
│       ├── aqp_stage_02a_financial.mmd
│       ├── aqp_stage_02b_macro.mmd
│       ├── aqp_stage_02c_micro_market.mmd
│       ├── aqp_stage_02d_news.mmd
│       ├── aqp_stage_02e_validation.mmd
│       ├── aqp_stages_03_04_portfolio.mmd
│       ├── aqp_stage_05_sizing.mmd
│       ├── aqp_stage_06_execution.mmd
│       ├── aqp_safety_controls.mmd
│       └── aqp_stage_07_monitoring.mmd
│
├── data/
│   ├── README.md
│   ├── raw/                 # Original inputs; normally excluded from Git
│   ├── interim/             # Intermediate transformed data
│   ├── processed/           # Validated, analysis-ready datasets
│   ├── reference/           # Symbol mappings, calendars, classifications
│   └── samples/             # Small synthetic/non-sensitive fixtures for tests
│
├── src/
│   └── aqp_python/
│       ├── __init__.py
│       ├── cli.py
│       ├── logging_config.py
│       ├── config/
│       │   ├── aqp_config_loader.py
│       │   └── aqp_config_validation.py
│       ├── contracts/
│       │   ├── aqp_data_schemas.py
│       │   ├── aqp_forecast_schema.py
│       │   ├── aqp_portfolio_schema.py
│       │   └── aqp_run_manifest.py
│       ├── connectors/
│       │   └── lseg/
│       │       ├── __init__.py
│       │       ├── aqp_lseg_session.py
│       │       ├── aqp_lseg_instruments.py
│       │       └── aqp_lseg_fields.py
│       ├── orchestration/
│       │   ├── aqp_orchestrator.py
│       │   ├── aqp_stage_router.py
│       │   ├── aqp_run_context.py
│       │   └── aqp_run_manifest_writer.py
│       ├── storage/
│       │   ├── aqp_storage_paths.py
│       │   ├── aqp_dataset_registry.py
│       │   └── aqp_run_store.py
│       ├── stages/
│       │   ├── stage_00_data/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_lseg_extractor.py
│       │   │   ├── aqp_hfd_builder.py
│       │   │   ├── aqp_lfd_builder.py
│       │   │   ├── aqp_corporate_actions.py
│       │   │   ├── aqp_data_validator.py
│       │   │   └── aqp_point_in_time_controls.py
│       │   │
│       │   ├── stage_01_universe/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_candidate_universe.py
│       │   │   ├── aqp_company_screening.py
│       │   │   ├── aqp_universe_scoring.py
│       │   │   ├── aqp_archetype_assignment.py
│       │   │   ├── aqp_universe_formation.py
│       │   │   └── aqp_safety_exit_monitor.py
│       │   │
│       │   ├── stage_02_forecasting/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_forecast_runner.py
│       │   │   ├── aqp_forecast_registry.py
│       │   │   ├── aqp_forecast_adapter.py
│       │   │   ├── financial/
│       │   │   │   ├── aqp_financial_features.py
│       │   │   │   ├── aqp_return_forecast.py
│       │   │   │   └── aqp_distress_model.py
│       │   │   ├── macro/
│       │   │   │   ├── aqp_macro_features.py
│       │   │   │   ├── aqp_macro_state_model.py
│       │   │   │   └── aqp_macro_sensitivity.py
│       │   │   ├── micro_market/
│       │   │   │   ├── aqp_interday_market_model.py
│       │   │   │   ├── aqp_intraday_features.py
│       │   │   │   └── aqp_market_microstructure_model.py
│       │   │   ├── news/
│       │   │   │   ├── aqp_news_ingestion.py
│       │   │   │   ├── aqp_news_event_extractor.py
│       │   │   │   ├── aqp_news_novelty_check.py
│       │   │   │   └── aqp_news_event_forecast.py
│       │   │   └── validation/
│       │   │       ├── aqp_walk_forward_validation.py
│       │   │       ├── aqp_forecast_calibration.py
│       │   │       ├── aqp_model_eligibility.py
│       │   │       └── aqp_signal_combiner.py
│       │   │
│       │   ├── stages_03_04_portfolio/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_covariance_estimator.py
│       │   │   ├── aqp_portfolio_risk.py
│       │   │   ├── aqp_stress_testing.py
│       │   │   ├── aqp_portfolio_optimizer.py
│       │   │   └── aqp_sleeve_allocator.py
│       │   │
│       │   ├── stage_05_sizing/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_position_sizer.py
│       │   │   ├── aqp_kelly_cap.py
│       │   │   ├── aqp_leverage_rules.py
│       │   │   └── aqp_position_report.py
│       │   │
│       │   ├── stage_06_execution/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_order_intent.py
│       │   │   ├── aqp_execution_simulator.py
│       │   │   ├── aqp_cost_slippage_model.py
│       │   │   └── aqp_fill_reconciliation.py
│       │   │
│       │   └── stage_07_monitoring/
│       │       ├── __init__.py
│       │       ├── aqp_performance_monitor.py
│       │       ├── aqp_data_drift_monitor.py
│       │       ├── aqp_model_drift_monitor.py
│       │       ├── aqp_run_diagnostics.py
│       │       └── aqp_readiness_status.py
│       │
│       ├── safety/
│       │   ├── __init__.py
│       │   ├── aqp_pre_trade_checks.py
│       │   ├── aqp_hard_limits.py
│       │   ├── aqp_kill_switch.py
│       │   └── aqp_restart_approval.py
│       │
│       ├── compiler/
│       │   ├── __init__.py
│       │   ├── aqp_plot_compiler.py
│       │   ├── aqp_report_compiler.py
│       │   └── aqp_run_artifact_index.py
│       │
│       └── reporting/
│           ├── __init__.py
│           ├── aqp_metrics_summary.py
│           ├── aqp_narrative_report.py
│           └── aqp_report_fact_checks.py
│
├── r_models/               # Python package invokes R through a documented adapter
│   ├── README.md
│   ├── scripts/
│   │   └── aqp_realised_garch.R
│   └── adapters/
│       └── aqp_r_model_adapter.py
│
├── scripts/
│   ├── aqp_run_pipeline.py
│   ├── aqp_validate_environment.py
│   ├── aqp_build_sample_data.py
│   └── aqp_export_diagrams.ps1
│
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_aqp_config_loader.py
│   │   ├── test_aqp_data_validator.py
│   │   ├── test_aqp_universe_scoring.py
│   │   ├── test_aqp_forecast_schema.py
│   │   ├── test_aqp_portfolio_constraints.py
│   │   └── test_aqp_safety_controls.py
│   ├── integration/
│   │   ├── test_aqp_stage_handoffs.py
│   │   └── test_aqp_run_artifacts.py
│   └── test_batches/
│       ├── test_batch_01_foundation/
│       ├── test_batch_02_data_extraction/
│       ├── test_batch_03_universe_selection/
│       ├── test_batch_04_forecasting_validation/
│       ├── test_batch_05_portfolio_sizing_safety/
│       ├── test_batch_06_execution_simulation/
│       └── test_batch_07_monitoring_end_to_end/
│
├── outputs/                # Generated artifacts; normally excluded from Git
│   ├── runs/
│   ├── plots/
│   ├── reports/
│   ├── diagnostics/
│   └── logs/
│
└── build/                  # Temporary local build/export outputs; normally excluded from Git
```

## 4. Stage-by-stage responsibilities

| Stage | Main entry point(s) | Responsibility | Principal output |
|---|---|---|---|
| **00 — Data** | `aqp_lseg_extractor.py`, `aqp_data_validator.py` | Extract LSEG data, construct HFD/LFD datasets, handle corporate actions, validate fields and enforce point-in-time rules. | Validated historical and event datasets with data-quality flags. |
| **01 — Universe** | `aqp_universe_formation.py` | Apply hard eligibility gates, compute category and composite scores, assign archetypes and version the research universe. | Frozen, versioned universe plus exclusion reasons and score details. |
| **02 — Forecasting** | `aqp_forecast_runner.py` | Run financial, macro, interday/intraday market and news models through a shared forecast contract. | Forecast distribution/quantiles, horizon, uncertainty, provenance and status. |
| **02E — Validation** | `aqp_walk_forward_validation.py`, `aqp_model_eligibility.py` | Evaluate forecasts chronologically out of sample, assess calibration and determine whether a model is eligible for combination. | Validation metrics and model eligibility decisions. |
| **03–04 — Portfolio** | `aqp_portfolio_optimizer.py` | Estimate portfolio risk, stress scenarios, optimise within constraints and allocate risk across trading sleeves. | Proposed portfolio weights and risk diagnostics. |
| **05 — Sizing** | `aqp_position_sizer.py`, `aqp_leverage_rules.py` | Apply position, Kelly-fraction, volatility, leverage, liquidity and margin constraints. | Bounded target positions and an auditable sizing report. |
| **06 — Execution** | `aqp_execution_simulator.py` | Convert approved intents into simulated orders, model costs/slippage and reconcile fills. | Order, fill, cost and reconciliation records. |
| **07 — Monitoring** | `aqp_performance_monitor.py` | Track performance, data/model drift, run health, plots and readiness status. | Diagnostics, monitoring metrics and status. |
| **Safety — Independent** | `aqp_pre_trade_checks.py`, `aqp_kill_switch.py` | Enforce hard limits, block invalid orders and require an explicit process for restart after a stop. | Allow/block decisions with reason codes. |
| **Compiler/reporting** | `aqp_plot_compiler.py`, `aqp_narrative_report.py` | Gather model plots and structured run facts; generate an evidence-based run report. | Indexed plots and a narrative report tied to the run ID. |

### Standard forecast contract

Forecasting models should return a documented object rather than arbitrary, inconsistent files. At minimum, record:

- instrument identifier, market and sleeve;
- forecast creation timestamp and information cut-off timestamp;
- target variable, forecast horizon and units;
- point forecast and available predictive quantiles or distribution parameters;
- uncertainty/calibration fields where available;
- model name/version, training window and data lineage;
- validation status and eligibility status;
- warnings, missing-input flags and failure reason codes.

Do not compare or combine forecasts with different targets, horizons or units until they have been explicitly aligned.

## 5. Orchestration and expected run flow

```text
Configuration and run ID
        ↓
Stage 00: Extract and validate data
        ↓
Stage 01: Form the research universe
        ↓
Stage 02: Generate forecasts in parallel
        ↓
Stage 02E: Validate and combine eligible forecasts
        ↓
Stages 03–04: Risk estimation and portfolio optimisation
        ↓
Stage 05: Position sizing and leverage constraints
        ↓
Independent safety approval
        ↓
Stage 06: Execution simulation / approved execution adapter
        ↓
Stage 07: Monitoring and readiness evaluation
        ↓
Compile plots, diagnostics, run manifest and narrative report
```

The orchestrator should stop, block, or mark a run degraded when an essential stage fails. A failed model may be skipped only if the relevant configuration defines an approved fallback and the run records the degradation. The narrative report must describe the recorded output; it must not invent results or override a safety decision.

## 6. Required run outputs

Each run should be written to `outputs/runs/<run_id>/` with a predictable layout, for example:

```text
outputs/runs/<run_id>/
├── run_manifest.json
├── config_snapshot.yaml
├── data_quality_summary.csv
├── universe/
│   ├── selected_companies.csv
│   └── exclusions.csv
├── forecasts/
├── portfolio/
├── sizing/
├── execution/
├── diagnostics/
├── plots/
└── reports/
    ├── run_summary.json
    └── aqp_run_report.md
```

The run manifest should record the run ID, timestamps and timezone, Git commit if available, configuration hash, data references, universe version, model versions, stage statuses, warnings, output paths and failure reasons. Use immutable run-specific output folders so a later run does not overwrite the evidence from an earlier run.

### Plot and report requirements

- Each model should save traceable plots into the current run's `plots/` folder.
- Plot filenames should contain stable identifiers for the model, instrument or market, and plot type. Avoid embedding all metadata only in the image.
- `aqp_plot_compiler.py` should build a manifest/index of produced plots and report missing expected outputs.
- `aqp_narrative_report.py` should use the structured run summary as its source of truth. Any AI-generated prose must distinguish measured results, model interpretations, limitations and failures.
- If report generation fails, retain the structured run summary and mark the narrative-report stage as failed; do not mark the full pipeline successful merely because earlier stages completed.

## 7. Seven ordered test batches

Run batches in order. Each batch should have a README describing its objective, fixtures, command, expected artifacts and pass criteria. Use synthetic or sanitised data in automated tests; do not commit credentials or proprietary raw data.

| Batch | Scope | Example pass condition |
|---|---|---|
| `test_batch_01_foundation` | Package import, config loading, logging, run IDs and manifest. | Configuration errors are caught and a manifest is produced. |
| `test_batch_02_data_extraction` | LSEG connector mocks, HFD/LFD transformations, missing data, timestamps and corporate actions. | Schema and point-in-time tests pass; missing or stale data is flagged. |
| `test_batch_03_universe_selection` | Hard gates, scoring direction, category weighting, archetypes and frozen membership. | Same inputs and configuration produce the same universe and reason codes. |
| `test_batch_04_forecasting_validation` | Forecast contracts, model adapters, walk-forward splits, calibration and eligibility. | No future-data leakage; invalid or unvalidated forecasts cannot enter combination. |
| `test_batch_05_portfolio_sizing_safety` | Covariance inputs, optimisation constraints, Kelly cap, leverage and hard limits. | Invalid, infeasible or over-limit portfolios are rejected or safely constrained. |
| `test_batch_06_execution_simulation` | Order intents, costs, fills, slippage, partial fills and reconciliation. | Simulated fills reconcile and prohibited orders are blocked. |
| `test_batch_07_monitoring_end_to_end` | Orchestrator, plot compiler, run manifest, diagnostics and narrative report. | A full test run produces traceable artifacts and truthful PASS/DEGRADED/BLOCKED statuses. |

A passing test batch establishes only the behaviour covered by that test. It does not establish that a strategy is profitable or ready for live trading.

## 8. Windows and VS Code setup

From PowerShell, open the repository root in VS Code and create a project virtual environment:

```powershell
cd path\to\AQP-PYTHON
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e .
pip install -r requirements-dev.txt
code .
```

If PowerShell blocks activation, use the terminal profile or activation approach permitted by your Windows policy; do not weaken system-wide execution policy just for this project. Configure VS Code to use `AQP-PYTHON\.venv\Scripts\python.exe` as the interpreter.

Example run commands once the CLI has been implemented:

```powershell
python -m aqp_python.cli validate-config --config configs/aqp_pipeline.yaml
python -m aqp_python.cli run --config configs/aqp_pipeline.yaml --mode research
pytest
```

These are the intended command interfaces; they will work only after the package, CLI and dependencies have been implemented.

## 9. Data, credentials and Git hygiene

- Keep LSEG credentials, API secrets and private tokens outside source control. Use environment variables or an approved secret store; `.env.example` contains placeholder names only.
- Add raw, intermediate and large generated datasets to `.gitignore` unless a specific small, licensed, sanitised fixture is deliberately approved for version control.
- Do not commit `.venv/`, caches, logs, large plot collections or run outputs by default.
- Use `.gitkeep` only when an otherwise-empty directory must appear in Git; it does not make hidden files visible or cause Git to track ignored files.
- Preserve data provider licences and terms. A successful LSEG connection does not itself permit redistribution of downloaded data.
- Pin and record dependency versions for reproducibility. Use lock files if the project adopts a dependency manager that supports them.

## 10. Python and R boundary

Python coordinates the pipeline and owns the standard schemas, data contracts, run manifest, portfolio controls, compiler and reporting. R implementations belong under `r_models/` and should be called through `aqp_r_model_adapter.py` or another documented interface. Define input/output types, timeout behaviour, error handling and package requirements before integrating an R model. Do not silently translate a failed R model into a successful forecast.

## 11. Implementation order

1. Establish repository tooling, configuration validation, logging and run manifests.
2. Implement Stage 00 with mocked and small approved data fixtures before relying on full LSEG extraction.
3. Implement Stage 01 screening, score normalisation, universe versioning and rejection reasons.
4. Define and test the forecast contract before adding the individual Stage 02 models.
5. Add walk-forward validation and model-eligibility rules before allowing forecasts into portfolio construction.
6. Implement portfolio constraints, sizing and independent safety tests before execution integration.
7. Add execution simulation, monitoring, plots and structured reports.
8. Run the seven test batches in order; only then progress from research to paper trading and any separately approved controlled deployment.

## 12. Key definitions

- **AQP:** Advanced Quantitative Pipeline.
- **HFD:** High-frequency data, such as intraday observations or market data sampled at short intervals.
- **LFD:** Low-frequency data, such as daily or other less frequent observations.
- **Point-in-time data:** Data represented according to when it was actually available to the decision process.
- **Research universe:** Versioned set of companies eligible for downstream analysis; not a list of automatic trades.
- **Archetype:** A classification that routes an eligible company to relevant modelling hypotheses.
- **Forecast eligibility:** A validated status allowing a model output to be considered by the signal-combination stage.
- **Sleeve:** A separately configured strategy allocation, such as ASX interday or US intraday.
- **Run manifest:** Machine-readable record of configuration, data, versions, stage statuses and output artifacts for one pipeline run.
- **Hard limit:** A mandatory risk or operational constraint that a forecast or optimiser cannot override.

---

**Repository rule:** keep this file as the navigation map. Put extended mathematics and model-specific derivations in the relevant methodology document; put executable code in `src/aqp_python/`; put tests in `tests/`; and keep generated run artifacts under `outputs/`.
