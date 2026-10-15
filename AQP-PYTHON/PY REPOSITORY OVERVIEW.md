# AQP-PYTHON — Repository Overview

**Project:** Advanced Quantitative Pipeline (AQP)  
**Repository:** `AQP-PYTHON`  
**Primary language:** Python  
**Environment:** Windows, Visual Studio Code, PowerShell and a project virtual environment  
**Status:** Proposed repository design; files listed in the tree are planned until implemented and tested.

## 1. Purpose and design rules

`AQP-PYTHON` contains the Python implementation for AQP: data extraction and validation, company selection, forecasting, portfolio construction, sizing, execution simulation, monitoring, plotting and run reporting.

The design follows these rules:

1. **One entry point:** `aqp_python.cli` starts commands and `aqp_orchestrator.py` coordinates a pipeline run.
2. **Configuration is not code:** YAML files define run settings; Python loads and validates them.
3. **Point-in-time integrity:** a decision may only use information available at its timestamp.
4. **Selection is not a trading signal:** Stage 1 forms a research universe; later stages evaluate opportunities.
5. **Independent risk controls:** models, optimisers and reports cannot override hard limits or the kill switch.
6. **Reproducible runs:** record the run ID, code version, configuration snapshot, data references, model versions, statuses and output paths.
7. **Research first:** research, paper trading and live execution are separate modes with separate approval gates.
8. **Consolidate until separation is useful:** start with a small number of cohesive modules; split them only when complexity, testing or ownership justifies it.

## 2. Naming conventions

| Item | Convention | Example |
|---|---|---|
| Repository | `AQP-PYTHON` | `AQP-PYTHON/` |
| Python package | `aqp_python` | `src/aqp_python/` |
| Custom module | `aqp_` prefix | `aqp_universe_builder.py` |
| Stage directory | Numbered stage | `stage_01_universe/` |
| YAML configuration | `aqp_` prefix | `aqp_risk_limits.yaml` |
| Test module | `test_*.py` | `test_aqp_safety_controls.py` |
| Mermaid diagram | `aqp_*.mmd` | `aqp_stage_01_universe.mmd` |
| Run output | Unique run ID | `outputs/runs/<run_id>/` |

Do not add `AQP` to every local variable or standard test name. The repository and package already identify the project; use clear, consistent names for custom files.

## 3. Recommended repository tree

```text
AQP-PYTHON/
├── README.md                         # Quick start and project status
├── PY REPOSITORY OVERVIEW.md         # This repository map
├── pyproject.toml                    # Package metadata, dependencies, CLI and tooling
├── .gitignore                        # Excludes secrets, environments and generated files
├── .env.example                      # Names of required environment variables; no real secrets
│
├── configs/                          # Run settings; data only, not executable code
│   ├── aqp_pipeline.yaml             # Run mode, selected stages and date range
│   ├── aqp_markets.yaml              # ASX/US calendars, time zones and sessions
│   ├── aqp_data_sources.yaml         # Provider, extraction and data-frequency settings
│   ├── aqp_universe_rules.yaml       # Stage 1 gates, scoring weights and archetypes
│   ├── aqp_forecast_models.yaml      # Enabled model families, horizons and parameters
│   ├── aqp_validation.yaml           # Walk-forward splits, metrics and eligibility criteria
│   ├── aqp_risk_limits.yaml          # Position, concentration, volatility and leverage limits
│   ├── aqp_execution.yaml            # Simulation, transaction costs and order constraints
│   └── aqp_reporting.yaml            # Plots, diagnostics and narrative-report options
│
├── docs/
│   ├── aqp_architecture.md           # Architecture and stage hand-offs
│   ├── aqp_data_dictionary.md        # Fields, units, identifiers and timestamps
│   ├── aqp_model_registry.md         # Model inventory, versions and status
│   └── diagrams/                     # Editable Mermaid diagrams (.mmd)
│       ├── aqp_pipeline_overview.mmd
│       ├── aqp_stage_00_data.mmd
│       ├── aqp_stage_01_universe.mmd
│       ├── aqp_stage_02_forecasting.mmd
│       ├── aqp_stages_03_04_portfolio.mmd
│       ├── aqp_stage_05_sizing.mmd
│       ├── aqp_stage_06_execution.mmd
│       ├── aqp_stage_07_monitoring.mmd
│       └── aqp_safety_controls.mmd
│
├── data/
│   ├── README.md                     # Data handling and licensing rules
│   └── samples/                      # Small synthetic or approved test fixtures
│
├── src/
│   └── aqp_python/
│       ├── __init__.py               # Marks the Python package; keep lightweight
│       ├── cli.py                    # The single command-line entry point
│       ├── config/
│       │   └── aqp_config.py         # Loads, validates and exposes YAML settings
│       ├── contracts/
│       │   └── aqp_contracts.py      # Shared data, forecast, portfolio and run schemas
│       ├── connectors/
│       │   └── lseg/
│       │       ├── __init__.py
│       │       └── aqp_lseg_client.py # LSEG session, instrument/field mapping and API calls
│       ├── orchestration/
│       │   └── aqp_orchestrator.py   # Run context, ordered stage execution and statuses
│       ├── storage/
│       │   └── aqp_storage.py        # Run paths, dataset references and output writes
│       ├── stages/
│       │   ├── stage_00_data/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_market_data.py # Extract/build daily and intraday datasets
│       │   │   └── aqp_data_validation.py # Quality, timestamps and point-in-time checks
│       │   ├── stage_01_universe/
│       │   │   ├── __init__.py
│       │   │   └── aqp_universe_builder.py # Gates, multi-factor score, archetypes and version
│       │   ├── stage_02_forecasting/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_forecast_runner.py # Runs model families under a common contract
│       │   │   ├── aqp_forecast_validation.py # Walk-forward tests, calibration and eligibility
│       │   │   ├── financial/
│       │   │   │   └── aqp_financial_models.py
│       │   │   ├── macro/
│       │   │   │   └── aqp_macro_models.py
│       │   │   ├── micro_market/
│       │   │   │   └── aqp_market_models.py # Interday and intraday market models
│       │   │   └── news/
│       │   │       └── aqp_news_models.py
│       │   ├── stages_03_04_portfolio/
│       │   │   ├── __init__.py
│       │   │   ├── aqp_risk_engine.py # Covariance, portfolio risk and stress scenarios
│       │   │   ├── aqp_portfolio_optimizer.py
│       │   │   └── aqp_sleeve_allocator.py
│       │   ├── stage_05_sizing/
│       │   │   ├── __init__.py
│       │   │   └── aqp_position_sizer.py # Position size, Kelly fraction and leverage constraints
│       │   ├── stage_06_execution/
│       │   │   ├── __init__.py
│       │   │   └── aqp_execution.py   # Order intents, simulation, costs and reconciliation
│       │   └── stage_07_monitoring/
│       │       ├── __init__.py
│       │       └── aqp_monitoring.py  # Performance, drift, diagnostics and readiness status
│       ├── safety/
│       │   ├── __init__.py
│       │   ├── aqp_pre_trade_checks.py
│       │   ├── aqp_hard_limits.py
│       │   └── aqp_kill_switch.py
│       └── reporting/
│           ├── __init__.py
│           ├── aqp_plot_compiler.py   # Indexes and checks model plots
│           └── aqp_run_report.py     # Metrics summary and evidence-based narrative report
│
├── r_models/                          # Optional R implementations called through an adapter
│   ├── README.md
│   └── scripts/
│       └── aqp_realised_garch.R      # Example R model; integrate only when required
│
├── tests/
│   ├── conftest.py
│   ├── unit/                          # Fast tests for individual functions/modules
│   ├── integration/                   # Stage hand-offs and run artifact tests
│   └── batches/
│       └── README.md                  # Seven ordered test batches; no duplicate test tree
│
└── outputs/                           # Generated artifacts; excluded from Git by default
    ├── README.md
    └── runs/                          # One immutable folder per run
```

**Files shown are planned files, not proof of implementation.** Empty directories may not appear in Git until they contain a tracked file; add a short `README.md` where an otherwise-empty directory needs a documented purpose.

## 4. Why this tree is smaller

The simplified design removes duplicate ways to start the pipeline and consolidates related functions into cohesive modules:

| Earlier separation | Simplified location | Why |
|---|---|---|
| Separate runner script and CLI | `cli.py` | One supported entry point avoids competing launch paths. |
| Config loader + validator | `config/aqp_config.py` | Loading and validation are closely related; split later if they grow substantially. |
| Paths + dataset registry + run store | `storage/aqp_storage.py` | One storage interface for paths, datasets and run artifacts. |
| LSEG session + instruments + fields | `connectors/lseg/aqp_lseg_client.py` | One client initially owns provider-specific details. |
| HFD builder + LFD builder | `stage_00_data/aqp_market_data.py` | Shared extraction/transformation logic with frequency-specific functions. |
| Candidate universe + scoring + formation | `stage_01_universe/aqp_universe_builder.py` | One stage entry point handles gates, scoring, archetypes and versioning. |
| Several tiny forecast registries/adapters | `aqp_forecast_runner.py` | Start with a simple runner and common forecast contract. |
| Walk-forward + calibration + eligibility | `aqp_forecast_validation.py` | Keep related validation steps together. |
| Covariance + portfolio risk + stress tests | `aqp_risk_engine.py` | One risk module supplies consistent estimates to the optimiser. |
| Kelly cap + leverage rules + position report | `aqp_position_sizer.py` | Keep sizing functions together but test each rule independently. |
| Separate drift monitors | `aqp_monitoring.py` | One monitoring entry point can call separately testable checks. |
| Plot compiler + report compiler + metrics summary | `reporting/` | Two outputs are enough initially: plot index and run report. |

Consolidation is about **fewer files**, not fewer controls. Hard limits and the kill switch remain separate modules so they can be reviewed and tested independently. Forecast families also remain distinct because they use different data and assumptions.

## 5. Stage responsibilities

| Stage | Main module | Responsibility | Main output |
|---|---|---|---|
| **00 — Data** | `aqp_market_data.py`, `aqp_data_validation.py` | Extract and normalise market data; check timestamps, duplicates, missing fields, corporate actions and point-in-time availability. | Validated datasets plus quality summary. |
| **01 — Universe** | `aqp_universe_builder.py` | Apply hard eligibility gates, compute category/composite scores, assign archetypes and version membership. | Eligible research universe with scores and exclusion reasons. |
| **02 — Forecasting** | `aqp_forecast_runner.py` | Run financial, macro, market/microstructure and news models through a common interface. | Forecasts with horizon, uncertainty, provenance and status. |
| **02E — Validation** | `aqp_forecast_validation.py` | Evaluate out of sample, check calibration and decide which forecasts can be combined. | Validation metrics and eligibility decisions. |
| **03–04 — Portfolio** | `aqp_risk_engine.py`, `aqp_portfolio_optimizer.py`, `aqp_sleeve_allocator.py` | Estimate risk, stress portfolios, optimise within constraints and allocate across ASX/US and interday/intraday sleeves. | Proposed weights and portfolio diagnostics. |
| **05 — Sizing** | `aqp_position_sizer.py` | Apply position, Kelly-fraction, volatility, liquidity, leverage and margin limits. | Bounded target positions and sizing report. |
| **06 — Execution** | `aqp_execution.py` | Simulate orders, estimate costs/slippage, model fills and reconcile results. | Order/fill/cost records. |
| **07 — Monitoring** | `aqp_monitoring.py` | Monitor performance, data/model drift, run health and readiness. | Diagnostics and run status. |
| **Independent safety** | `safety/` | Enforce pre-trade checks, hard limits and emergency stop conditions. | Allow/block decisions with reason codes. |
| **Reporting** | `reporting/` | Gather model plots and create a narrative based on structured results. | Indexed plots and run report. |

### Shared forecast contract

Each forecast must identify the instrument, market/sleeve, target, units, forecast horizon, creation time, information cut-off time, point estimate and available predictive quantiles or distribution parameters. Include model/version, training window, data lineage, calibration/eligibility status, warnings and failure reason codes. Forecasts with different targets, units or horizons must not be combined until explicitly aligned.

## 6. One run from start to finish

```text
CLI command + YAML settings
          ↓
Load and validate configuration
          ↓
Create run ID and immutable run manifest
          ↓
Stage 00 — Extract and validate data
          ↓
Stage 01 — Form the research universe
          ↓
Stage 02 — Run forecasting families
          ↓
Stage 02E — Validate forecasts and combine eligible signals
          ↓
Stages 03–04 — Risk, optimisation and sleeve allocation
          ↓
Stage 05 — Position sizing and leverage rules
          ↓
Independent hard-limit and pre-trade checks
          ↓
Stage 06 — Execution simulation (research mode by default)
          ↓
Stage 07 — Monitoring, plot compilation and reporting
```

The orchestrator should stop or mark a run `DEGRADED`/`BLOCKED` when a required stage fails. Optional models may be skipped only where an explicit fallback is defined and recorded. A narrative report cannot turn a failed stage into a successful run or override a safety decision.

## 7. Run outputs

Each run receives its own output directory to prevent later runs overwriting earlier evidence:

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

The manifest should record the run ID and timestamps, timezone, Git commit if available, configuration hash, data references, universe version, model versions, stage statuses, warnings, output paths and failure reasons.

### Plot and reporting requirements

- Every model should save plots into its current run's `plots/` area using predictable filenames.
- The plot compiler creates an index of expected and actual artifacts, and flags missing plots.
- The report generator uses structured run facts as its source of truth. Separate observed metrics from model interpretation and limitations.
- If narrative generation fails, retain the structured summary and mark report generation as failed; do not erase completed diagnostics.

## 8. Configuration files

`configs/` contains YAML data. `src/aqp_python/config/aqp_config.py` loads and validates the files. A YAML file does nothing until the Python code reads it and passes its values to the relevant stage.

| File | Key settings |
|---|---|
| `aqp_pipeline.yaml` | Run mode, stage selection, date range, output behaviour. |
| `aqp_markets.yaml` | Exchange calendars, time zones and trading sessions. |
| `aqp_data_sources.yaml` | LSEG/data-source settings, enabled frequencies and extraction options. |
| `aqp_universe_rules.yaml` | Eligibility gates, scoring directions/weights and archetype rules. |
| `aqp_forecast_models.yaml` | Enabled forecast families, horizons and parameters. |
| `aqp_validation.yaml` | Chronological splits, metrics, calibration and eligibility thresholds. |
| `aqp_risk_limits.yaml` | Position/concentration caps, volatility target, leverage and margin limits. |
| `aqp_execution.yaml` | Transaction costs, slippage assumptions and execution mode. |
| `aqp_reporting.yaml` | Plots, diagnostics, report format and narrative settings. |

Validate required keys, types, allowed ranges and cross-file compatibility before starting expensive work. Store the effective configuration snapshot for each run. Configuration must not be able to disable independent hard safety controls.

## 9. Seven ordered test batches

Use the same unit and integration tests in a documented progression instead of creating duplicate copies in seven test directories. `tests/batches/README.md` should specify commands, fixtures, expected outputs and pass criteria for each batch.

| Batch | Coverage | Example pass condition |
|---|---|---|
| 01 — Foundation | Package import, configuration, logging and run IDs. | Bad configuration fails early; valid run manifest is created. |
| 02 — Data | Mocked LSEG client, daily/intraday transformations and data quality. | Invalid/stale data is detected; timestamps and schemas are consistent. |
| 03 — Universe | Eligibility gates, score direction, weighting and archetypes. | Same inputs and configuration produce reproducible membership and reason codes. |
| 04 — Forecasting | Forecast contract, model families and walk-forward validation. | No future-data leakage; ineligible forecasts cannot enter combination. |
| 05 — Portfolio and safety | Risk estimates, optimiser constraints, Kelly/leverage and hard limits. | Infeasible or over-limit positions are blocked or safely constrained. |
| 06 — Execution | Order intents, costs, fills and reconciliation. | Simulated records reconcile; prohibited orders are rejected. |
| 07 — End-to-end | Orchestration, monitoring, plot index and report. | Run artifacts are traceable and statuses accurately represent failures. |

A test pass only establishes the behaviour covered by that test. It does not demonstrate that a trading strategy is profitable or ready for live trading.

## 10. Windows and VS Code workflow

Open the repository root in VS Code. In the PowerShell terminal:

```powershell
# Create and activate a project-specific environment
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip

# Install the package and its declared dependencies after pyproject.toml is implemented
python -m pip install -e .

# Validate configuration, run research mode and execute tests after the CLI is implemented
python -m aqp_python.cli validate-config --config configs/aqp_pipeline.yaml
python -m aqp_python.cli run --config configs/aqp_pipeline.yaml --mode research
pytest
```

`pyproject.toml` is the authoritative source for package metadata and dependencies. Do not keep a second hand-maintained `requirements.txt` unless a specific deployment/export workflow needs it. The commands above are intended interfaces and will work only after the package, CLI, configuration files and dependencies have been implemented.

If PowerShell blocks virtual-environment activation, use an approved terminal setup rather than weakening system-wide execution policy. Select `.venv\Scripts\python.exe` as the VS Code interpreter.

## 11. Data and Git hygiene

- Never commit LSEG credentials, tokens, passwords or real `.env` files. `.env.example` contains placeholder variable names only.
- Keep licensed raw data and large generated datasets out of Git unless a small, sanitised fixture is explicitly approved.
- Ignore `.venv/`, Python caches, local logs and generated run outputs by default.
- `.gitkeep` only helps retain an otherwise-empty directory; it does not override `.gitignore` or make files visible.
- Respect LSEG licensing and entitlements; successful retrieval does not imply redistribution rights.
- Record package versions and configuration hashes for reproducibility.

## 12. Python/R boundary

Python owns orchestration, configuration, data contracts, portfolio controls, reporting and the run manifest. Optional R model scripts live in `r_models/` and are invoked through a documented adapter only when required. Define data types, input/output schemas, timeout behaviour, package requirements and failure handling. A failed R call must not be converted into a valid forecast silently. If the project later becomes strictly Python-only, remove `r_models/` and replace or formally retire its functionality.

## 13. Implementation order

1. Create package metadata, the CLI, configuration loader/validator, logging and run manifests.
2. Implement Stage 00 with synthetic data and mocked LSEG responses.
3. Implement Stage 01 screening, scoring, archetypes, versioned universe and rejection reasons.
4. Define the forecast contract and implement baseline models before adding complex forecast families.
5. Add walk-forward validation and eligibility checks before forecasts reach portfolio construction.
6. Implement portfolio constraints, Kelly/leverage calculations and independent safety tests.
7. Add execution simulation, monitoring, plot compilation and evidence-based run reporting.
8. Complete the seven test batches before any transition from research to paper trading or separately approved live deployment.

## 14. Key terms

- **AQP:** Advanced Quantitative Pipeline.
- **HFD/LFD:** High-frequency and low-frequency data.
- **Point-in-time data:** Values represented according to when they were actually available to the decision process.
- **Research universe:** Versioned set of companies eligible for analysis; not an automatic list of trades.
- **Archetype:** Company/market classification used to route relevant modelling hypotheses.
- **Forecast eligibility:** Validation status required before a forecast can be combined with other signals.
- **Sleeve:** Separately configured strategy allocation, such as ASX interday or US intraday.
- **Run manifest:** Machine-readable record of versions, configuration, data lineage, stage results and artifacts for one run.
- **Hard limit:** A mandatory risk or operational constraint that a model or optimiser cannot override.

---

**Repository rule:** use this file as the navigation map; keep detailed mathematical derivations in methodology documents, executable Python in `src/aqp_python/`, tests in `tests/`, configuration data in `configs/` and generated run artifacts under `outputs/`.
