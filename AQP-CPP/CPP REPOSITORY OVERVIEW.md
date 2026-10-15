# AQP-C++ — Repository Overview

**Project:** Advanced Quantitative Pipeline (AQP)  
**Repository:** `AQP-CPP`  
**Implementation language:** C++  
**Recommended standard:** C++20  
**Primary development environment:** Windows + Visual Studio Code + Microsoft Visual C++ (MSVC)  
**Build system:** CMake  
**Status:** Target repository design — the tree below describes the intended implementation and does not mean every listed source file already exists.

---

## 1. Purpose

`AQP-CPP` is the native C++ implementation of the Advanced Quantitative Pipeline. It is designed to manage market-data ingestion, point-in-time data validation, company-universe formation, forecasts, portfolio construction, position sizing, execution simulation, monitoring, plots/data exports where supported, and run reporting.

The repository follows the same research workflow as the AQP overview, but its pipeline logic and model implementations are written in C++. It does not rely on Python or R source files to execute the pipeline.

**Language boundary:** YAML/JSON configuration, Markdown documentation, Mermaid diagrams, CMake build files, and test data are not program code. They remain appropriate in a C++ repository. This design deliberately excludes `.py` and `.R` implementation files.

> **Research and trading disclaimer:** the architecture is not evidence of profitability or production readiness. Start with synthetic/sample data and research mode. Do not enable live order submission until the data, controls, broker integration, security review, and operational procedures have been independently tested and approved.

## 2. Design principles

1. **C++ implementation:** core pipeline stages, data contracts, calculations, orchestration, and tests use C++.
2. **CMake-managed builds:** keep compiler and dependency setup reproducible rather than relying on IDE-only settings.
3. **Configuration-driven runs:** YAML settings are loaded and validated by C++ code; configuration files do not execute themselves.
4. **Point-in-time integrity:** every input should preserve event time, publication/availability time, retrieval time, source, and revision information where applicable.
5. **Separate hard gates from scores:** a high company score cannot override failed data, tradability, or safety checks.
6. **Separate forecasts from orders:** models produce forecasts; portfolio construction proposes weights; independent controls approve, constrain, or block order intents.
7. **Reproducibility:** every run records code version, configuration snapshot, input lineage, universe version, model versions, random seeds where relevant, and outcome status.
8. **Safe defaults:** research or simulation mode is the default. Live execution should require explicit deployment configuration and separate approval.
9. **No hidden side effects:** constructing an object or including a header must not log in to a provider, download data, or submit orders.
10. **Incremental implementation:** each stage has unit tests, integration tests, and a clearly defined input/output contract.

## 3. Repository tree

```text
AQP-CPP/
├── CPP REPOSITORY OVERVIEW.md
├── README.md
├── CMakeLists.txt
├── CMakePresets.json
├── CMakeUserPresets.json.example
├── vcpkg.json
├── vcpkg-configuration.json
├── .gitignore
├── .gitattributes
├── .clang-format
├── .clang-tidy
├── LICENSE
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
│   │   ├── aqp_cpp_system_overview.mmd
│   │   ├── aqp_cpp_stage_00_data.mmd
│   │   ├── aqp_cpp_stage_01_universe.mmd
│   │   ├── aqp_cpp_stage_02_forecasting.mmd
│   │   ├── aqp_cpp_stage_02e_validation.mmd
│   │   ├── aqp_cpp_stages_03_04_portfolio.mmd
│   │   ├── aqp_cpp_stage_05_sizing.mmd
│   │   ├── aqp_cpp_stage_06_execution.mmd
│   │   ├── aqp_cpp_stage_07_monitoring.mmd
│   │   └── aqp_cpp_safety_controls.mmd
│   ├── decisions/
│   │   └── README.md
│   └── testing/
│       └── aqp_cpp_test_strategy.md
│
├── include/
│   └── aqp_cpp/
│       ├── aqp_version.hpp
│       ├── core/
│       │   ├── aqp_error.hpp
│       │   ├── aqp_result.hpp
│       │   ├── aqp_types.hpp
│       │   ├── aqp_timestamp.hpp
│       │   ├── aqp_run_context.hpp
│       │   └── aqp_data_contracts.hpp
│       ├── config/
│       │   ├── aqp_config.hpp
│       │   ├── aqp_config_loader.hpp
│       │   └── aqp_config_validator.hpp
│       ├── connectors/
│       │   ├── aqp_market_data_source.hpp
│       │   ├── aqp_news_data_source.hpp
│       │   └── lseg/
│       │       ├── aqp_lseg_session.hpp
│       │       ├── aqp_lseg_instruments.hpp
│       │       └── aqp_lseg_fields.hpp
│       ├── storage/
│       │   ├── aqp_file_store.hpp
│       │   ├── aqp_parquet_store.hpp
│       │   └── aqp_run_manifest.hpp
│       ├── stages/
│       │   ├── stage_00_data/
│       │   │   ├── aqp_data_extractor.hpp
│       │   │   ├── aqp_data_normalizer.hpp
│       │   │   └── aqp_data_quality.hpp
│       │   ├── stage_01_universe/
│       │   │   ├── aqp_universe_builder.hpp
│       │   │   ├── aqp_company_filters.hpp
│       │   │   ├── aqp_factor_scorer.hpp
│       │   │   └── aqp_archetype_classifier.hpp
│       │   ├── stage_02_forecasting/
│       │   │   ├── aqp_forecast_contract.hpp
│       │   │   ├── financial/aqp_financial_models.hpp
│       │   │   ├── macro/aqp_macro_models.hpp
│       │   │   ├── micro/aqp_micro_models.hpp
│       │   │   └── news/aqp_news_models.hpp
│       │   ├── stage_02e_validation/
│       │   │   ├── aqp_walk_forward.hpp
│       │   │   ├── aqp_forecast_calibration.hpp
│       │   │   └── aqp_model_eligibility.hpp
│       │   ├── stages_03_04_portfolio/
│       │   │   ├── aqp_covariance.hpp
│       │   │   ├── aqp_stress_testing.hpp
│       │   │   ├── aqp_portfolio_optimizer.hpp
│       │   │   └── aqp_sleeve_allocator.hpp
│       │   ├── stage_05_sizing/
│       │   │   ├── aqp_position_sizer.hpp
│       │   │   ├── aqp_kelly_cap.hpp
│       │   │   └── aqp_leverage_rules.hpp
│       │   ├── stage_06_execution/
│       │   │   ├── aqp_order_intent.hpp
│       │   │   ├── aqp_execution_simulator.hpp
│       │   │   └── aqp_reconciliation.hpp
│       │   └── stage_07_monitoring/
│       │       ├── aqp_metrics.hpp
│       │       ├── aqp_plot_data.hpp
│       │       ├── aqp_report_generator.hpp
│       │       └── aqp_run_monitor.hpp
│       ├── safety/
│       │   ├── aqp_pre_trade_gate.hpp
│       │   ├── aqp_hard_limits.hpp
│       │   ├── aqp_kill_switch.hpp
│       │   └── aqp_release_approval.hpp
│       └── orchestration/
│           ├── aqp_pipeline.hpp
│           ├── aqp_stage_registry.hpp
│           └── aqp_orchestrator.hpp
│
├── src/
│   ├── main.cpp
│   ├── core/
│   ├── config/
│   │   ├── aqp_config_loader.cpp
│   │   └── aqp_config_validator.cpp
│   ├── connectors/
│   │   ├── aqp_file_market_data_source.cpp
│   │   ├── aqp_file_news_data_source.cpp
│   │   └── lseg/
│   │       ├── aqp_lseg_session.cpp
│   │       ├── aqp_lseg_instruments.cpp
│   │       └── aqp_lseg_fields.cpp
│   ├── storage/
│   ├── stages/
│   │   ├── stage_00_data/
│   │   ├── stage_01_universe/
│   │   ├── stage_02_forecasting/
│   │   │   ├── financial/
│   │   │   ├── macro/
│   │   │   ├── micro/
│   │   │   └── news/
│   │   ├── stage_02e_validation/
│   │   ├── stages_03_04_portfolio/
│   │   ├── stage_05_sizing/
│   │   ├── stage_06_execution/
│   │   └── stage_07_monitoring/
│   ├── safety/
│   └── orchestration/
│       ├── aqp_pipeline.cpp
│       ├── aqp_stage_registry.cpp
│       └── aqp_orchestrator.cpp
│
├── tests/
│   ├── CMakeLists.txt
│   ├── unit/
│   │   ├── core/
│   │   ├── config/
│   │   ├── stages/
│   │   └── safety/
│   ├── integration/
│   │   ├── aqp_config_integration_test.cpp
│   │   ├── aqp_file_data_integration_test.cpp
│   │   └── aqp_pipeline_smoke_test.cpp
│   ├── fixtures/
│   │   ├── market_data/
│   │   ├── company_fundamentals/
│   │   ├── news_events/
│   │   └── expected_outputs/
│   └── test_batches/
│       ├── batch_01_repository_and_build/
│       ├── batch_02_config_and_contracts/
│       ├── batch_03_data_and_connectors/
│       ├── batch_04_universe_selection/
│       ├── batch_05_forecasts_and_validation/
│       ├── batch_06_portfolio_sizing_safety/
│       └── batch_07_end_to_end_reporting/
│
├── data/
│   ├── README.md
│   ├── sample/
│   └── .gitkeep
│
├── outputs/
│   ├── README.md
│   ├── runs/.gitkeep
│   ├── plots/.gitkeep
│   └── reports/.gitkeep
│
└── build/                         # Generated by CMake; do not commit
```

The tree is a target layout. Some modules may be combined or split during implementation; avoid creating empty source files merely to make the tree look complete.

## 4. Root-level build and development files

| File | Purpose |
|---|---|
| `CMakeLists.txt` | Declares the C++ project, source targets, compiler standard, dependencies, executable and test targets. |
| `CMakePresets.json` | Defines repeatable configure, build and test presets, including Windows/MSVC Debug and Release builds. |
| `CMakeUserPresets.json.example` | Template for local paths or machine-specific options; the actual private `CMakeUserPresets.json` should normally be ignored by Git. |
| `vcpkg.json` | Declares third-party C++ dependencies for reproducible package installation when using vcpkg. |
| `vcpkg-configuration.json` | Pins or configures the vcpkg baseline and registries where needed. |
| `.gitignore` | Excludes build output, IDE state, secrets, caches, raw licensed data and generated results. |
| `.clang-format` | Defines consistent C++ formatting. |
| `.clang-tidy` | Enables configurable static-analysis checks. |
| `README.md` | Gives the short start guide; this overview carries the detailed repository map. |
| `LICENSE` | Specifies project licensing once an appropriate licence has been chosen. |

## 5. Configuration files

Configuration files hold run settings; they are data files and do not execute by themselves. C++ code loads, validates, snapshots and passes the values to the relevant stage.

| File | Responsibility |
|---|---|
| `configs/aqp_pipeline.yaml` | Run mode, enabled stages, date range, output root and global parameters. |
| `configs/aqp_markets.yaml` | ASX/US definitions, timezone names, exchange calendars and session settings. |
| `configs/aqp_data_sources.yaml` | Data providers, file/API modes, frequencies, request controls and retry settings. Never store API secrets directly in the tracked file. |
| `configs/aqp_universe_rules.yaml` | Stage 1 exclusion gates, factor definitions, category weights and archetype rules. |
| `configs/aqp_forecast_models.yaml` | Model enablement, horizons, estimator settings and common output requirements. |
| `configs/aqp_validation.yaml` | Walk-forward design, calibration checks, out-of-sample metrics and eligibility criteria. |
| `configs/aqp_risk_limits.yaml` | Position, concentration, drawdown, exposure, leverage and portfolio-risk constraints. |
| `configs/aqp_execution.yaml` | Simulation settings, fees, spread/slippage assumptions and permitted execution mode. |
| `configs/aqp_reporting.yaml` | Output formats, diagnostics, plot requests and narrative-report controls. |

Suggested C++ configuration components:

- `aqp_config_loader`: reads the selected YAML files using a C++ YAML library.
- `aqp_config_validator`: checks schema, ranges, cross-file consistency and missing mandatory settings.
- `aqp_config`: stores typed, validated settings passed to each component.

The orchestrator should stop before any market-data requests or portfolio calculations if required configuration is missing or invalid. Each run should save a configuration snapshot and its content hash. Safety limits must also be enforced in C++ safety code; a YAML setting must not be able to bypass independent hard-coded invariants or deployment approvals.

## 6. C++ module responsibilities

### Core and contracts

`include/aqp_cpp/core/` defines shared types and interfaces: timestamps, run context, errors/results, instrument identifiers, market observations, forecast records, portfolio proposals, order intents and data-lineage metadata. Put shared contracts here rather than allowing every stage to invent its own field names.

Use explicit units and semantics. A return, a price, a percentage, a basis-point value and a variance must not be represented as interchangeable unlabelled `double` values when a stronger type can prevent mistakes. Document timestamp timezone and whether intervals are open or closed.

### Connectors

`connectors/` isolates external systems from modelling code. Start with a file-backed connector using deterministic sample fixtures; this allows development without credentials or live data access. Keep any future LSEG adapter behind an interface such as `MarketDataSource`.

**LSEG note:** do not assume that a supported native C++ Workspace SDK or an entitlement is available. The concrete integration must use an officially supported interface that your account is allowed to access, or a licensed export/file workflow. If a supported HTTP/REST endpoint is used, isolate authentication, networking, rate limits, retry policy and response parsing in the connector. Do not embed credentials in source code, YAML committed to Git, or logs. A connector is not implemented just because its header exists.

### Stage 00 — Data ingestion and quality

Extract or read market/company/news data, normalise identifiers, validate timestamps, handle corporate actions, check duplicates and missing observations, and preserve source and availability metadata. Keep raw source records immutable where practical and write normalised data to a separate location.

### Stage 01 — Company universe

Apply mandatory eligibility gates first, then calculate category-level company scores, rank candidates and assign strategy archetypes. Persist the formation date, universe version, factor values, component scores and exclusion reason codes. A high composite score must never compensate for a failed hard gate.

### Stage 02 — Forecasting

Maintain one common forecast contract across four families:

- **Financial:** company fundamentals and company-level return/risk estimates.
- **Macro:** economic series, macro states and exposure/sensitivity estimates.
- **Micro/market:** interday price/volume features and intraday market-microstructure signals.
- **News:** event extraction, novelty, event classification and calibrated event-response estimates.

Each result should identify its instrument, forecast horizon, as-of time, model/version, expected return or distributional outputs where appropriate, uncertainty, data lineage, and validity status. A news model may call an externally hosted service only through an explicit C++ client and a configured provider; it must not silently add an untracked dependency.

### Stage 02E — Forecast validation

Use chronological or walk-forward splits, with no future information in training, feature generation or validation. Test calibration and economic relevance after realistic costs. Record sample sizes, folds, trial counts and model status. Only eligible forecast records should be passed to portfolio construction.

### Stages 03–04 — Portfolio and sleeve allocation

Estimate covariance and downside/stress measures, build constrained portfolio proposals, include transaction-cost/liquidity assumptions, and allocate risk budgets to ASX interday, US interday, ASX intraday and US intraday sleeves. Confirm solver status, constraint residuals and feasibility; do not assume an optimiser succeeded because it returned a vector.

### Stage 05 — Position sizing and leverage

Convert approved target weights into position proposals using volatility, uncertainty, liquidity, margin and drawdown restrictions. Keep fractional Kelly as a capped research input, not an automatic authority to use leverage. Enforce all applicable limits and prevent negative or non-finite position sizes. Record the binding constraint and calculation inputs.

### Stage 06 — Execution

The initial version should simulate orders, fees, spreads, impact, slippage and fills. Keep broker-facing order submission disabled unless an explicit, separately approved live mode is implemented. Reconcile intended orders with simulated or broker-reported fills and record discrepancies.

### Stage 07 — Monitoring and reporting

Collect run metrics, diagnostics, plots and structured outputs, then generate a narrative report from those stored results. If using a remote AI/LLM service, label it as an external dependency and supply only authorised data. The report must not invent numbers, claim that a failed run succeeded, or alter risk decisions. Store the input metrics and report text for auditability.

### Independent safety layer

`src/safety/` is designed to sit outside the forecast-to-order path as a blocking control. It should validate approved mode, data freshness, trading permissions, limits, position/exposure calculations and kill-switch state immediately before any order can be released. A failure blocks the action and records a reason. Restart after a kill-switch event should require an explicit defined reset and approval process.

## 7. File naming and C++ conventions

- Repository directory: `AQP-CPP`.
- Public headers: `include/aqp_cpp/.../*.hpp`.
- Implementations: `src/.../*.cpp`.
- Project-specific file prefix: `aqp_` for named modules and components.
- Namespace: `aqp` (use sub-namespaces only when there is a clear ownership boundary).
- Types: `PascalCase`, for example `RunContext` and `MarketDataSource`.
- Functions and variables: use one consistent project convention, preferably `snake_case`.
- Constants: follow the project's documented constant naming convention consistently.
- Header guards: use `#pragma once` consistently if that is the chosen compiler/tooling policy.
- Keep public declarations in `.hpp` files and function definitions in `.cpp` files unless templates or other language rules make header definitions necessary.
- Avoid giant all-in-one source files. Keep provider-specific details and third-party headers out of public interfaces where practical.
- No Python `.py`, R `.R`, or mixed-language implementation is part of this repository design.

## 8. Dependencies and build approach

Use the smallest suitable dependency set and pin it through a reproducible dependency manager. Candidate libraries should be selected during implementation, after confirming licences and compatibility:

- **CMake + MSVC:** configure and compile the project on Windows.
- **vcpkg:** reproducible dependency management, if adopted.
- **yaml-cpp:** YAML parsing, if YAML is retained.
- **GoogleTest or Catch2:** C++ unit and integration tests; choose one initially.
- **spdlog:** structured application logging, if adopted.
- **fmt:** formatting, if adopted (coordinate version with any logging library).
- **Eigen:** matrix and vector operations, if suitable for the model workload.
- **A validated optimisation solver:** choose based on licence, numerical requirements and supported constraints; do not assume one particular solver is installed.
- **libcurl or another supported HTTP client:** only if a permitted service/API integration requires it.
- **Parquet/Arrow support:** optional if columnar storage is required and compatible library packages are available.

These are candidate dependencies, not a statement that they are already installed or selected. Review licensing, supported compilers, security updates, ABI compatibility, and package availability before adding a dependency.

## 9. Windows and VS Code build workflow

Install Visual Studio Build Tools or Visual Studio with the C++ desktop development workload and a Windows SDK, plus CMake and Git. VS Code is the editor; it does not itself supply the C++ compiler. The CMake Tools extension can help select a compiler kit and run configure/build/test tasks.

Once the corresponding `CMakePresets.json`, root `CMakeLists.txt`, and source files have been created, the intended command workflow is:

```powershell
# From the AQP-CPP repository root
cmake --list-presets
cmake --preset windows-msvc-debug
cmake --build --preset windows-msvc-debug
ctest --preset windows-msvc-debug
```

Those preset names are proposed conventions and will work only after matching presets have been implemented. If you configure with Visual Studio's generator, use a matching architecture such as x64 and choose Debug or Release intentionally.

A typical CLI design after implementation could be:

```powershell
# Check configuration only; does not extract data or place orders
.\build\windows-msvc-debug\aqp_cli.exe validate-config `
  --config configs/aqp_pipeline.yaml

# Run against synthetic/sample data in research mode
.\build\windows-msvc-debug\aqp_cli.exe run `
  --config configs/aqp_pipeline.yaml `
  --mode research `
  --data-source sample
```

These commands describe the intended interface and are not executable until the CLI and build targets exist. Actual binary paths depend on the CMake generator and runtime-output settings. Never put API passwords or tokens on the command line where they may appear in process listings or logs.

## 10. Runtime execution sequence

1. Parse CLI arguments and select the configuration path.
2. Load configuration and validate its schema and cross-file requirements.
3. Create a unique `run_id`, run directory, structured logger and manifest.
4. Record source-code revision, configuration snapshot/hash, selected market/sleeve, as-of window and data source.
5. Initialise only the connectors required for this run mode.
6. Execute Stage 00 and fail closed if mandatory data checks fail.
7. Form or load the explicitly versioned Stage 01 research universe.
8. Run enabled Stage 02 forecasting models and record model/data lineage.
9. Run Stage 02E validation and exclude ineligible outputs.
10. Construct and validate portfolio proposals in Stages 03–04.
11. Apply position sizing and limits in Stage 05.
12. Pass eligible order intents through independent safety checks.
13. Run Stage 06 execution simulation by default; live submission is unavailable unless separately implemented and approved.
14. Run Stage 07 monitoring, plots, structured metrics and the narrative report.
15. Finalise the run manifest with `PASS`, `DEGRADED` or `BLOCKED` status and clear failure reasons.

A later stage must not run when a required predecessor fails. Define an explicit degraded-mode policy for optional failures; never silently treat missing data or an unavailable model as a normal forecast.

## 11. Output conventions

Every run should write to a unique directory, for example:

```text
outputs/runs/<run_id>/
├── run_manifest.json
├── config_snapshot/
├── logs/
├── data_quality/
├── universe/
├── forecasts/
├── validation/
├── portfolio/
├── sizing/
├── execution_simulation/
├── diagnostics/
├── plots/
└── reports/
    ├── run_summary.json
    └── aqp_run_report.md
```

The exact formats should be chosen per output contract. Keep large licensed datasets and secrets outside source control. Commit only small synthetic fixtures that are redistributable and sufficient for tests. The run manifest should reference file paths, hashes, versions, status and timestamps rather than attempting to embed all outputs into one file.

## 12. Testing strategy: seven ordered batches

| Batch | Main objective | Pass condition |
|---|---|---|
| `batch_01_repository_and_build` | CMake configure/build, compiler flags, CLI starts | Clean Debug build and basic executable smoke test succeed. |
| `batch_02_config_and_contracts` | YAML loading, schema checks, timestamps and common data/forecast types | Valid configs load; invalid or conflicting configs fail with actionable errors. |
| `batch_03_data_and_connectors` | File/sample ingestion, normalisation, identifier handling and data-quality gates | Fixtures produce deterministic outputs; missing/corrupt input is rejected or explicitly classified. |
| `batch_04_universe_selection` | Hard filters, factor transformations, category score, archetypes and frozen universe records | Expected fixture companies and reason codes match independent test expectations. |
| `batch_05_forecasts_and_validation` | Baseline models, time-aware splits, calibration and eligibility | No future-data leakage in tests; forecast contracts and eligibility decisions are reproducible. |
| `batch_06_portfolio_sizing_safety` | Risk calculations, optimiser feasibility, Kelly/leverage caps and kill switch | Constraint violations or safety failures reliably block the proposed action. |
| `batch_07_end_to_end_reporting` | Whole sample-data run, run manifest, plots/data exports and narrative report | End-to-end run is reproducible and report values trace back to structured run artifacts. |

Use unit tests for local mathematics and validation logic, integration tests for interactions between modules, and end-to-end smoke tests for the orchestrator. Do not use live market data or broker credentials in ordinary unit tests.

## 13. Minimum first milestone

The first deliverable should not be an advanced trading model. It should be a small but working C++ program that can:

1. Configure and build from a clean checkout using CMake.
2. Load `configs/aqp_pipeline.yaml` and reject invalid values.
3. Run with synthetic data and no external credentials.
4. Create a unique run directory and manifest.
5. Exercise one placeholder stage through the same stage interface the full pipeline will use.
6. Emit a summary and pass the first test batch.

After this foundation works, implement the stages in order and replace each placeholder only when its unit and integration tests pass.

## 14. What is not included yet

This overview defines the target repository and component responsibilities; it does not include the full implementations of the listed `.cpp` and `.hpp` files, the actual CMake presets, dependency lock/baseline, YAML schemas, compiled CLI, models, API authentication, broker integration, or a validated execution system. Those must be built and tested separately.

The first LSEG milestone should be a file-backed/sample connector with the same interface intended for any eventual provider connector. This allows the data contracts and Stage 00 to be tested before committing to an API integration that may depend on current product support, account permissions and entitlements.

## 15. Reference concepts

Use official documentation during implementation to verify compiler/tool versions and APIs rather than copying commands blindly:

- CMake documentation: <https://cmake.org/documentation/>
- CMake Presets manual: <https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html>
- Microsoft C++ documentation: <https://learn.microsoft.com/en-us/cpp/>
- VS Code C++ documentation: <https://code.visualstudio.com/docs/languages/cpp>
- vcpkg documentation: <https://learn.microsoft.com/en-us/vcpkg/>
- yaml-cpp project: <https://github.com/jbeder/yaml-cpp>
- GoogleTest documentation: <https://google.github.io/googletest/>

---

**Repository rule:** `AQP-CPP` contains the native C++ implementation; `aqp_` names the project's custom modules; configurations and documentation are kept separate from source; and every run begins in research/sample mode until all required tests and approvals explicitly permit anything more consequential.
