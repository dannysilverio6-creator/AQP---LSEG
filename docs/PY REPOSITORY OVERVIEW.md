# AQP-PYTHON — Python Repository Overview

**Project:** AQP — Advanced Quantitative Pipeline (Multi-Market Quantitative Investment Pipeline, design specification v12)  
**Repository:** `AQP-PYTHON`  
**Layout version:** 2 — aligned to the v12 design specification; supersedes the first proposed layout  
**Primary language:** Python  
**Companion language:** R, kept separate and called through a controlled adapter  
**Intended environment:** Windows, Visual Studio Code, Git and a project-specific Python virtual environment  
**Status:** Proposed repository layout and implementation guide. An empty shell of this layout exists; no pipeline logic has been implemented.

---

## 1. Purpose and status

`AQP-PYTHON` contains the Python implementation of the Advanced Quantitative Pipeline. It follows the design's flow: point-in-time data → fixed research universe → forecasts → validation → risk and optimisation → leverage and pre-trade gates → execution → monitoring and reporting, across four sleeves (ASX interday, US interday, ASX intraday, US intraday).

This document defines the **target repository structure**. A path shown in the tree is a planned location until its module has been implemented and tested. The presence of a file here does not imply that the corresponding model is complete, validated or approved for live trading, and a sophisticated structure does not establish an economic edge.

**Shell convention.** The shell contains folders, `__init__.py` files and zero-byte placeholders for non-Python files (configuration, documentation, R scripts). The 243 planned Python modules shown in the tree are **not** created in the shell; only the 33 `__init__.py` package markers are. Folders that would otherwise be empty hold a `.gitkeep` so Git tracks them.

### 1.1 Design rules and where the repository enforces them

| Design rule (v12) | Repository mechanism |
|---|---|
| Point-in-time data; no look-ahead | `storage/vintage_store.py` (as-of queries over valid time and availability time), `stage_00_data/point_in_time_controls.py`, gate G2, `tests/invariants/test_no_lookahead.py` |
| Fixed, versioned universe; mandatory removals only | `stage_01_universe/universe_versioning.py`, `mandatory_removals.py`, `storage/universe_store.py`, `data/universes/`, `tests/invariants/test_universe_fixed_membership.py` |
| Selection is separate from forecasting | `stage_01_universe` never imports `stage_02_forecasting`; it produces a research universe, not trades |
| One forecast schema (quantile grid, lineage, label end date) | `contracts/forecast_schema.py`, `quantile_grid.py`, `stage_02_forecasting/forecast_validator.py` |
| Validate before weighting; charge for search | `stage_02_forecasting/validation/`, `storage/trial_registry.py`, gate G3 |
| Hard limits independent of models | `safety/` plus import contracts in `.importlinter` and `tests/invariants/test_hard_limits_independence.py` |
| Relative weights, total leverage and order authorisation are separate | Optimiser in `stages_03_04_portfolio`, leverage in `stage_05_sizing`, order authority in `stage_06_execution` and `safety` |
| Infeasible optimisation never creates tradable weights | `relaxation_policy.py`, `tests/invariants/test_infeasible_means_no_trade.py` |
| Reproducible runs with manifest | `orchestration/run_manifest_writer.py`, `storage/run_store.py`, `scripts/replay_run.py` |
| Controlled model promotion (no automatic replacement) | `model_lifecycle/`, gate G7, `models/` lifecycle folders |
| Narrative is generated from structured outputs only | `reporting/report_fact_checks.py`, `tests/invariants/test_narrative_uses_structured_outputs.py` |

### 1.2 What changed from the first layout

| Area | Change | Design basis |
|---|---|---|
| Gates | New `gates/` package with one module per gate G1–G7 and a shared gate-result type | Section 2 of the design: gates, failure actions, reason codes |
| Reason codes and statuses | New `contracts/reason_codes.py` and `run_status.py` (`PASS`/`DEGRADED`/`BLOCKED`; `TRADE`/`REDUCE`/`HOLD`/`NO-TRADE`) | Sections 2 and 16 |
| Company Reference and Exposure Dataset | New `stage_00_data/reference/` (geographic revenue and cost exposures, classifications, peer clusters, relationship graph, membership history) and `data/company_reference/` | Stage 0 |
| Data foundation | Added identifier mapping, calendar management, unit/currency standardisation, borrow/margin extraction, a vintage store and a data-quality report | Stage 0 timing and G2 rules |
| Universe stage | Split into corporate-status, distress, liquidity, history and stability filters; robust standardisation and composite scoring; strategy classification; data-tier assignment; versioning | Stage 1 |
| Sleeves | One configuration file per sleeve under `configs/sleeves/`; separate calendars, costs, data depth and risk budgets | Section 1 sleeves |
| Forecast schema | Dedicated quantile grid, label builder (label end date for purging) and forecast validator | Stage 2 |
| Forecast layers | Stage 2B–2D modules now mirror the design: dynamic factors, regimes, MIDAS/BVAR nowcasts, release surprises, local projections, FX; interday and intraday micro branches; news timing checks, prompt registry, LLM extraction, graph propagation | Stages 2A–2D |
| Validation | Walk-forward folds, purging/embargo, scoring (pinball, CRPS), calibration, comparison tests (Diebold–Mariano, Giacomini–White, SPA, MCS), overfitting controls (DSR, PBO), net-of-cost evaluation, horizon selection, trial registry | Stage 2E |
| Risk and optimisation | Factor, DCC and regime covariance; tail-risk engines and backtests; stress library; Black–Litterman; relaxation policy; cross-market session alignment | Stages 3–4 |
| Sizing | Kelly cap, volatility cap, margin capacity, drawdown multiplier, overnight gap risk, financing, no-trade band, order-request builder | Stage 5 |
| Execution and safety | Order lifecycle, scheduler, implementation shortfall; safety gains reconciliation block and incident log; broker interface with a paper broker | Stages 6–7 |
| Model lifecycle | New `model_lifecycle/` package and `models/` state folders (candidate, shadow, approved, retired) | Controlled learning and G7 |
| Independence | `.importlinter` contracts and an invariants test folder | "Hard limits remain independent of model outputs" |
| Tests | Eight ordered batches aligned to the implementation roadmap, plus `tests/invariants/` | Section 17 |
| Docs | Frozen copy of the design, decision log, open-decision register, phase exit criteria | Sections 15 and 17 |
| R boundary | Adapter plus an I/O schema file; DCC-GARCH added as an R candidate | Stages 2A and 3–4 |

## 2. Naming convention

| Item | Convention | Example |
|---|---|---|
| Repository folder | `AQP-PYTHON` | `AQP-PYTHON/` |
| Python package | `aqp_python` | `src/aqp_python/` |
| Custom Python modules and scripts | Descriptive `snake_case`, no project prefix | `universe_formation.py` |
| Package markers | Standard name, no prefix | `__init__.py` |
| Stage folders | Stage number and name | `stages/stage_01_universe/` |
| Configuration files | Descriptive `snake_case`, YAML | `risk_limits.yaml` |
| Sleeve configuration | `sleeve_<market>_<horizon>.yaml` | `sleeve_asx_intraday.yaml` |
| R scripts | Descriptive `snake_case` | `dcc_garch.R` |
| Python tests | `test_*.py` | `test_gates.py` |
| Test batches | Number and capability | `test_batch_03_validation_framework/` |
| Documentation | Plain names, `.md` | `forecast_contract.md` |
| Generated run folders | Unique run ID, never overwritten | `outputs/runs/<run_id>/` |

Only the repository (`AQP-PYTHON`) and the package (`aqp_python`) carry the project name; individual modules, scripts and configuration files do not repeat it.

## 3. Repository tree

Legend: `.py` files other than `__init__.py` are planned module locations and are not present in the shell. Folders without a listed file (for example `data/raw/`) contain a `.gitkeep`.

```text
AQP-PYTHON/
├── README.md
├── PY REPOSITORY OVERVIEW.md                                       # this navigation map (layout v2)
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── .gitignore
├── .gitattributes
├── .editorconfig
├── .env.example                                                    # placeholder names only, never real credentials
├── .pre-commit-config.yaml
├── .importlinter                                                   # import contracts: independence of safety/, read-only reporting, no order authority for model_lifecycle
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   ├── tasks.json
│   └── extensions.json
├── configs/                                                        # all limits, thresholds and approvals live here, never in code
│   ├── pipeline.yaml
│   ├── markets.yaml
│   ├── calendars.yaml
│   ├── data_sources.yaml
│   ├── data_tiers.yaml
│   ├── gates.yaml
│   ├── reason_codes.yaml
│   ├── fallbacks.yaml
│   ├── universe_rules.yaml
│   ├── forecast_models.yaml
│   ├── validation.yaml
│   ├── risk_limits.yaml
│   ├── stress_scenarios.yaml
│   ├── leverage.yaml
│   ├── execution.yaml
│   ├── monitoring_alerts.yaml
│   ├── model_lifecycle.yaml
│   ├── reporting.yaml
│   └── sleeves/                                                    # one file per sleeve: calendar, costs, data depth, risk budget
│       ├── sleeve_asx_interday.yaml
│       ├── sleeve_us_interday.yaml
│       ├── sleeve_asx_intraday.yaml
│       └── sleeve_us_intraday.yaml
├── docs/
│   ├── design/
│   │   └── AQP_OVERVIEW.md                                         # frozen copy of the v12 design specification
│   ├── architecture/
│   │   ├── system_architecture.md
│   │   ├── data_dictionary.md
│   │   ├── model_registry.md
│   │   ├── forecast_contract.md
│   │   ├── run_contract.md
│   │   ├── gates_and_reason_codes.md
│   │   └── module_dependency_rules.md
│   ├── methodology/
│   │   ├── stage_00_data.md
│   │   ├── stage_01_universe.md
│   │   ├── stage_02_forecast_schema.md
│   │   ├── stage_02a_financial.md
│   │   ├── stage_02b_macro.md
│   │   ├── stage_02c_micro_market.md
│   │   ├── stage_02d_news.md
│   │   ├── stage_02e_validation.md
│   │   ├── stages_03_04_portfolio.md
│   │   ├── stage_05_sizing.md
│   │   ├── stage_06_execution.md
│   │   ├── safety_controls.md
│   │   ├── stage_07_monitoring.md
│   │   └── model_lifecycle.md
│   ├── decisions/
│   │   ├── open_decisions.md
│   │   └── decision_log.md
│   └── readiness/
│       ├── phase_exit_criteria.md
│       └── stage_completion_checklist.md
├── data/
│   ├── README.md
│   ├── raw/                                                        # original provider inputs; excluded from Git
│   ├── vintages/                                                   # point-in-time store: valid time + availability timestamp; excluded from Git
│   ├── interim/
│   ├── processed/                                                  # validated, analysis-ready datasets
│   ├── reference/                                                  # symbol mappings, calendars, classification tables
│   ├── company_reference/                                          # versioned Company Reference and Exposure Dataset
│   ├── universes/                                                  # frozen, versioned universe memberships and exit logs
│   └── samples/                                                    # small synthetic/non-sensitive fixtures for tests
├── models/                                                         # model artefacts by lifecycle state; excluded from Git
│   ├── README.md
│   ├── registry/
│   ├── candidates/
│   ├── shadow/
│   ├── approved/
│   └── retired/
├── src/
│   └── aqp_python/
│       ├── __init__.py
│       ├── cli.py
│       ├── logging_config.py
│       ├── version.py
│       ├── config/
│       │   ├── __init__.py
│       │   ├── config_loader.py
│       │   ├── config_validation.py
│       │   └── sleeve_config.py
│       ├── contracts/                                              # shared schemas: the only layer every other package may depend on
│       │   ├── __init__.py
│       │   ├── data_schemas.py
│       │   ├── quantile_grid.py
│       │   ├── forecast_schema.py
│       │   ├── event_schema.py
│       │   ├── portfolio_schema.py
│       │   ├── order_schema.py
│       │   ├── position_report_schema.py
│       │   ├── reason_codes.py
│       │   ├── run_status.py
│       │   └── run_manifest.py
│       ├── gates/                                                  # G1-G7 as pure, testable checks returning a common gate result
│       │   ├── __init__.py
│       │   ├── gate_result.py
│       │   ├── gate_registry.py
│       │   ├── g1_universe.py
│       │   ├── g2_data_integrity.py
│       │   ├── g3_model_eligibility.py
│       │   ├── g4_portfolio_risk.py
│       │   ├── g5_leverage_margin.py
│       │   ├── g6_trade_eligibility.py
│       │   └── g7_model_promotion.py
│       ├── connectors/
│       │   ├── __init__.py
│       │   ├── connector_base.py
│       │   ├── lseg/
│       │   │   ├── __init__.py
│       │   │   ├── lseg_session.py
│       │   │   ├── lseg_instruments.py
│       │   │   └── lseg_fields.py
│       │   ├── macro/
│       │   │   ├── __init__.py
│       │   │   └── macro_provider.py
│       │   ├── news/
│       │   │   ├── __init__.py
│       │   │   └── news_provider.py
│       │   └── broker/                                             # used only by stage_06_execution and safety
│       │       ├── __init__.py
│       │       ├── broker_interface.py
│       │       └── paper_broker.py
│       ├── orchestration/
│       │   ├── __init__.py
│       │   ├── orchestrator.py
│       │   ├── stage_router.py
│       │   ├── sleeve_runner.py
│       │   ├── run_context.py
│       │   ├── stage_status.py
│       │   ├── fallback_registry.py
│       │   └── run_manifest_writer.py
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── storage_paths.py
│       │   ├── dataset_registry.py
│       │   ├── vintage_store.py
│       │   ├── universe_store.py
│       │   ├── trial_registry.py
│       │   ├── model_registry.py
│       │   └── run_store.py
│       ├── stages/
│       │   ├── __init__.py
│       │   ├── stage_00_data/
│       │   │   ├── __init__.py
│       │   │   ├── extractors/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── lseg_extractor.py
│       │   │   │   ├── macro_extractor.py
│       │   │   │   ├── news_extractor.py
│       │   │   │   └── borrow_margin_extractor.py
│       │   │   ├── identifier_mapper.py
│       │   │   ├── calendar_manager.py
│       │   │   ├── unit_currency_standardiser.py
│       │   │   ├── corporate_actions.py
│       │   │   ├── hfd_builder.py
│       │   │   ├── lfd_builder.py
│       │   │   ├── point_in_time_controls.py
│       │   │   ├── data_validator.py
│       │   │   ├── data_quality_report.py
│       │   │   └── reference/                                      # Company Reference and Exposure Dataset builders
│       │   │       ├── __init__.py
│       │   │       ├── company_reference_builder.py
│       │   │       ├── geographic_exposures.py
│       │   │       ├── classifications.py
│       │   │       ├── peer_clusters.py
│       │   │       ├── relationship_graph.py
│       │   │       └── membership_history.py
│       │   ├── stage_01_universe/
│       │   │   ├── __init__.py
│       │   │   ├── candidate_universe.py
│       │   │   ├── corporate_status_filter.py
│       │   │   ├── distress_screen.py
│       │   │   ├── liquidity_screen.py
│       │   │   ├── data_history_check.py
│       │   │   ├── stability_tests.py
│       │   │   ├── strategy_classification.py
│       │   │   ├── score_standardisation.py
│       │   │   ├── composite_score.py
│       │   │   ├── universe_formation.py
│       │   │   ├── universe_versioning.py
│       │   │   ├── data_tier_assignment.py
│       │   │   └── mandatory_removals.py
│       │   ├── stage_02_forecasting/
│       │   │   ├── __init__.py
│       │   │   ├── forecast_runner.py
│       │   │   ├── forecast_registry.py
│       │   │   ├── forecast_adapter.py
│       │   │   ├── label_builder.py
│       │   │   ├── forecast_validator.py
│       │   │   ├── financial/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── financial_features.py
│       │   │   │   ├── baselines.py
│       │   │   │   ├── return_quantiles.py
│       │   │   │   ├── distress_hazard.py
│       │   │   │   ├── volatility_garch.py
│       │   │   │   ├── volatility_har_rv.py
│       │   │   │   └── dynamic_exposures.py
│       │   │   ├── macro/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── macro_features.py
│       │   │   │   ├── vintage_alignment.py
│       │   │   │   ├── dynamic_factor_model.py
│       │   │   │   ├── regime_model.py
│       │   │   │   ├── nowcast_midas.py
│       │   │   │   ├── nowcast_bvar.py
│       │   │   │   ├── company_macro_exposure.py
│       │   │   │   ├── macro_sensitivity.py
│       │   │   │   ├── release_surprise.py
│       │   │   │   ├── transmission_local_projections.py
│       │   │   │   ├── fx_exposure.py
│       │   │   │   └── macro_forecast.py
│       │   │   ├── micro_market/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── micro_forecast.py
│       │   │   │   ├── interday/
│       │   │   │   │   ├── __init__.py
│       │   │   │   │   ├── panel_features.py
│       │   │   │   │   ├── panel_model.py
│       │   │   │   │   ├── hierarchical_shrinkage.py
│       │   │   │   │   ├── dml_effects.py
│       │   │   │   │   └── peer_earnings_response.py
│       │   │   │   └── intraday/
│       │   │   │       ├── __init__.py
│       │   │   │       ├── trade_quote_cleaning.py
│       │   │   │       ├── trade_intensity.py
│       │   │   │       ├── order_flow_imbalance.py
│       │   │   │       ├── price_impact.py
│       │   │   │       ├── realised_measures.py
│       │   │   │       ├── liquidity_state.py
│       │   │   │       └── execution_cost_estimator.py
│       │   │   ├── news/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── news_ingestion.py
│       │   │   │   ├── source_timing_check.py
│       │   │   │   ├── entity_linking.py
│       │   │   │   ├── story_deduplication.py
│       │   │   │   ├── prompt_registry.py
│       │   │   │   ├── llm_event_extractor.py
│       │   │   │   ├── event_schema_validator.py
│       │   │   │   ├── novelty_check.py
│       │   │   │   ├── graph_propagation.py
│       │   │   │   ├── event_response_model.py
│       │   │   │   └── news_event_forecast.py
│       │   │   └── validation/
│       │   │       ├── __init__.py
│       │   │       ├── walk_forward_folds.py
│       │   │       ├── purging_embargo.py
│       │   │       ├── baseline_comparison.py
│       │   │       ├── forecast_scoring.py
│       │   │       ├── forecast_calibration.py
│       │   │       ├── forecast_comparison_tests.py
│       │   │       ├── overfitting_controls.py
│       │   │       ├── stability_checks.py
│       │   │       ├── net_of_cost_evaluation.py
│       │   │       ├── model_eligibility.py
│       │   │       ├── signal_combiner.py
│       │   │       └── horizon_selection.py
│       │   ├── stages_03_04_portfolio/
│       │   │   ├── __init__.py
│       │   │   ├── covariance_estimator.py
│       │   │   ├── factor_covariance.py
│       │   │   ├── dcc_garch_adapter.py
│       │   │   ├── regime_covariance.py
│       │   │   ├── cross_market_alignment.py
│       │   │   ├── tail_risk_engines.py
│       │   │   ├── risk_backtests.py
│       │   │   ├── portfolio_risk.py
│       │   │   ├── stress_library.py
│       │   │   ├── stress_testing.py
│       │   │   ├── expected_returns.py
│       │   │   ├── portfolio_objective.py
│       │   │   ├── portfolio_constraints.py
│       │   │   ├── portfolio_optimizer.py
│       │   │   ├── relaxation_policy.py
│       │   │   └── sleeve_allocator.py
│       │   ├── stage_05_sizing/
│       │   │   ├── __init__.py
│       │   │   ├── kelly_cap.py
│       │   │   ├── volatility_cap.py
│       │   │   ├── margin_capacity.py
│       │   │   ├── drawdown_multiplier.py
│       │   │   ├── leverage_rules.py
│       │   │   ├── overnight_gap_risk.py
│       │   │   ├── financing_borrow_costs.py
│       │   │   ├── trade_decision.py
│       │   │   ├── no_trade_band.py
│       │   │   ├── invalidation_conditions.py
│       │   │   ├── order_request_builder.py
│       │   │   ├── position_sizer.py
│       │   │   └── position_report.py
│       │   ├── stage_06_execution/
│       │   │   ├── __init__.py
│       │   │   ├── order_intent.py
│       │   │   ├── execution_scheduler.py
│       │   │   ├── order_router.py
│       │   │   ├── order_lifecycle.py
│       │   │   ├── execution_simulator.py
│       │   │   ├── cost_slippage_model.py
│       │   │   ├── fill_reconciliation.py
│       │   │   └── implementation_shortfall.py
│       │   └── stage_07_monitoring/
│       │       ├── __init__.py
│       │       ├── performance_attribution.py
│       │       ├── performance_monitor.py
│       │       ├── calibration_monitor.py
│       │       ├── data_drift_monitor.py
│       │       ├── model_drift_monitor.py
│       │       ├── risk_backtest_monitor.py
│       │       ├── execution_error_monitor.py
│       │       ├── alert_rules.py
│       │       ├── run_diagnostics.py
│       │       └── readiness_status.py
│       ├── model_lifecycle/                                        # candidate -> G3 eligible -> shadow -> G7 approved -> retired; no order authority
│       │   ├── __init__.py
│       │   ├── candidate_registry.py
│       │   ├── challenger_training.py
│       │   ├── shadow_runner.py
│       │   ├── promotion_review.py
│       │   ├── release_manager.py
│       │   └── rollback.py
│       ├── safety/                                                 # independent hard limits; must not import forecast, portfolio, sizing or reporting code
│       │   ├── __init__.py
│       │   ├── pre_trade_checks.py
│       │   ├── hard_limits.py
│       │   ├── kill_switch.py
│       │   ├── reconciliation_block.py
│       │   ├── incident_log.py
│       │   └── restart_approval.py
│       ├── compiler/
│       │   ├── __init__.py
│       │   ├── plot_compiler.py
│       │   ├── report_compiler.py
│       │   └── run_artifact_index.py
│       └── reporting/                                              # read-only: reports describe recorded outputs and never issue orders
│           ├── __init__.py
│           ├── metrics_summary.py
│           ├── narrative_report.py
│           └── report_fact_checks.py
├── r_models/                                                       # R is called only through the documented adapter
│   ├── README.md
│   ├── scripts/
│   │   ├── realised_garch.R
│   │   └── dcc_garch.R
│   ├── schemas/
│   │   └── r_io_schema.json
│   └── adapters/
│       ├── __init__.py
│       └── r_model_adapter.py
├── scripts/
│   ├── run_pipeline.py
│   ├── run_sleeve.py
│   ├── validate_environment.py
│   ├── build_sample_data.py
│   └── replay_run.py
├── tests/
│   ├── conftest.py
│   ├── fixtures/
│   │   └── README.md
│   ├── unit/
│   │   ├── test_config_loader.py
│   │   ├── test_gates.py
│   │   ├── test_reason_codes.py
│   │   ├── test_data_validator.py
│   │   ├── test_vintage_store.py
│   │   ├── test_company_reference.py
│   │   ├── test_universe_scoring.py
│   │   ├── test_universe_removals.py
│   │   ├── test_forecast_schema.py
│   │   ├── test_quantile_grid.py
│   │   ├── test_walk_forward_folds.py
│   │   ├── test_purging_embargo.py
│   │   ├── test_model_eligibility.py
│   │   ├── test_signal_combiner.py
│   │   ├── test_covariance.py
│   │   ├── test_portfolio_constraints.py
│   │   ├── test_kelly_leverage.py
│   │   ├── test_no_trade_band.py
│   │   ├── test_execution_simulator.py
│   │   ├── test_safety_controls.py
│   │   └── test_run_manifest.py
│   ├── integration/
│   │   ├── test_stage_handoffs.py
│   │   ├── test_run_artifacts.py
│   │   ├── test_fallback_behaviour.py
│   │   └── test_r_adapter.py
│   ├── invariants/                                                 # design rules that must hold for every release
│   │   ├── test_no_lookahead.py
│   │   ├── test_survivorship_safety.py
│   │   ├── test_universe_fixed_membership.py
│   │   ├── test_infeasible_means_no_trade.py
│   │   ├── test_hard_limits_independence.py
│   │   ├── test_status_semantics.py
│   │   └── test_narrative_uses_structured_outputs.py
│   └── test_batches/                                               # run in order; each batch has a README
│       ├── test_batch_01_data_foundation/
│       │   └── README.md
│       ├── test_batch_02_universe_and_baselines/
│       │   └── README.md
│       ├── test_batch_03_validation_framework/
│       │   └── README.md
│       ├── test_batch_04_forecast_layers/
│       │   └── README.md
│       ├── test_batch_05_risk_optimisation_sizing/
│       │   └── README.md
│       ├── test_batch_06_execution_simulation/
│       │   └── README.md
│       ├── test_batch_07_safety_and_paper_trading/
│       │   └── README.md
│       └── test_batch_08_challenger_lifecycle/
│           └── README.md
├── outputs/                                                        # generated artefacts; excluded from Git
│   ├── runs/                                                       # one immutable folder per run_id
│   ├── plots/
│   ├── reports/
│   ├── diagnostics/
│   └── logs/
└── build/                                                          # temporary local build/export outputs; excluded from Git
```

## 4. Stage-by-stage responsibilities

| Stage | Package | Responsibility | Principal output | Gates |
|---|---|---|---|---|
| **00 — Data** | `stages/stage_00_data` | Extract provider data; standardise identifiers, calendars, units and time zones; keep vintages; apply corporate actions; build HFD/LFD; build the Company Reference and Exposure Dataset | Validated point-in-time data, quality flags, versioned company reference data | G2 |
| **01 — Universe** | `stages/stage_01_universe` | Corporate-status and distress screens, liquidity and tradability, data depth and stability, strategy classification, scoring, admission, data tiers, versioning, mandatory removals | Frozen, versioned research universe with reason codes and exit log | G1 |
| **02 — Schema** | `contracts`, `stage_02_forecasting` | Common quantile grid, excess-return target, label end date, schema and timestamp validation | Comparable predictive distributions | — |
| **02A — Financial** | `stage_02_forecasting/financial` | Return quantiles, GJR-GARCH/EGARCH and HAR-RV volatility, distress hazard, filtered dynamic exposures, simple baselines first | Return quantiles, volatility, distress probability | — |
| **02B — Macro** | `stage_02_forecasting/macro` | Dynamic factors, regimes, MIDAS/BVAR nowcasts, exposure-weighted sensitivities with shrinkage, release surprises, local-projection transmission, FX | Macro contribution and quantiles | — |
| **02C — Micro** | `stage_02_forecasting/micro_market` | Interday firm/peer panel models; intraday trade, quote and order-book features, realised measures, liquidity state, execution-cost estimates | Firm/microstructure contribution and quantiles | — |
| **02D — News** | `stage_02_forecasting/news` | Timing checks, deduplication, entity linking, fixed-schema LLM extraction, novelty, graph propagation, calibrated event-response model | News contribution and event probabilities | — |
| **02E — Validation** | `stage_02_forecasting/validation` | Walk-forward evaluation with purging and embargo, scoring, calibration, comparison tests, overfitting controls, net-of-cost evaluation, eligibility, combination | Eligibility decisions and combined forecast distributions; `NO-TRADE` if the eligible set is empty | G3 |
| **03–04 — Portfolio** | `stages/stages_03_04_portfolio` | Covariance and tail-risk estimation with backtests, stress library, Black–Litterman, cost-aware optimisation, constrained relaxation, sleeve allocation | Target weights, risk diagnostics, binding constraints | G4 |
| **05 — Sizing** | `stages/stage_05_sizing` | Kelly and volatility caps, margin capacity, drawdown multiplier, overnight gap risk, financing, trade decision, no-trade band, order requests, position report | Bounded exposure, order requests, position report with reason codes | G5, G6 |
| **06 — Execution** | `stages/stage_06_execution` | Scheduling, routing through the broker interface, order lifecycle, simulation, cost and slippage, reconciliation, implementation shortfall | Orders, fills, costs, reconciliation records | Pre-trade checks |
| **07 — Monitoring** | `stages/stage_07_monitoring` | Attribution, calibration and drift monitors, risk backtests, execution error, alert rules, readiness | Diagnostics, alerts, readiness status | Triggers G7 review |
| **Safety** | `safety` | Hard limits, kill switch, reconciliation block, incident log, restart approval | Allow/block decisions with reason codes | Independent of models |
| **Model lifecycle** | `model_lifecycle` | Candidate registry, challenger training, shadow runs, promotion review, versioned release, rollback | Approved-model changes with audit trail | G7 |
| **Compiler / reporting** | `compiler`, `reporting` | Index plots and run artefacts; metrics summary; narrative report with fact checks | Indexed plots and a report tied to the run ID | — |

## 5. Gates, reason codes and run status

Gates are implemented as pure checks that return a common gate result (gate ID, outcome, severity, machine-readable reason code, evidence reference). Gate thresholds are read from `configs/gates.yaml` and are provisional until approved per sleeve.

| Gate | Module | Evaluated in | Failure action |
|---|---|---|---|
| G1 Universe | `gates/g1_universe.py` | Stage 01 | Reject at formation or log a safety exit |
| G2 Data integrity | `gates/g2_data_integrity.py` | Stage 00 | Block (look-ahead, schema, units, time zone, duplicates), quarantine (stale history, unreconciled corporate actions) or warn |
| G3 Model eligibility | `gates/g3_model_eligibility.py` | Stage 02E | Exclude model; no eligible signal means no trade |
| G4 Portfolio risk | `gates/g4_portfolio_risk.py` | Stages 03–04 | Re-optimise within approved relaxations or scale toward cash |
| G5 Leverage and margin | `gates/g5_leverage_margin.py` | Stage 05 | Reduce gross exposure or block new risk |
| G6 Final trade eligibility | `gates/g6_trade_eligibility.py` | Stage 05, re-checked at Stage 06 | Block the order; risk-reducing action only if policy permits |
| G7 Model promotion | `gates/g7_model_promotion.py` | Model lifecycle | Keep the current approved model |

G1–G7 are project-defined labels, not external regulatory identifiers.

**Reason codes** live in `contracts/reason_codes.py` and `configs/reason_codes.yaml`. They include the universe rejection codes (`DIST`, `LIQ`, `HIST`, `DATA`), the mandatory-exit codes (`EXIT-DIST`, `EXIT-DLST`, `EXIT-SUSP`, `EXIT-TKO`, `EXIT-LIQ`) and the decision outcomes `TRADE`, `REDUCE`, `HOLD` and `NO-TRADE`. Every failure receives a code.

**Run status** (`contracts/run_status.py`): `PASS` means required checks passed; `DEGRADED` means an approved fallback from `configs/fallbacks.yaml` is active and the degradation is recorded; `BLOCKED` means a critical check failed and no new risk may be initiated. Fallbacks must be tested and can never bypass data integrity, margin, hard exposure limits, kill switches or reconciliation. Status is recorded per sleeve. As a repository convention (to be confirmed), the run's terminal status reports the most severe sleeve status (`BLOCKED` over `DEGRADED` over `PASS`).

## 6. Module dependency and independence rules

The design requires hard limits, kill switches and reconciliation to stay independent of forecasts, optimisation and monitoring. The repository enforces this structurally, not by convention only.

| Package | May import | Must not import |
|---|---|---|
| `contracts`, `config` | Standard library and third-party libraries | Any other `aqp_python` package |
| `storage`, `gates` | `contracts`, `config` | `stages`, `safety`, `model_lifecycle`, `reporting` |
| `stages/*` | `contracts`, `config`, `storage`, `gates` | Other stages' internals; hand-offs go through contract-typed datasets |
| `stage_01_universe` | As above | `stage_02_forecasting` (selection is not a trading signal) |
| `safety` | `contracts`, `config`, read-only `storage`, `connectors/broker` interface | `stage_02_forecasting`, `stages_03_04_portfolio`, `stage_05_sizing`, `model_lifecycle`, `reporting`, `compiler` |
| `model_lifecycle` | `contracts`, `config`, `storage`, `gates` | `connectors/broker`, `stage_06_execution` (a candidate has no order authority) |
| `reporting`, `compiler` | `contracts`, `config`, read-only `storage` | `connectors/broker`, `stage_06_execution`, `safety` write paths |
| `connectors/broker` | `contracts` | Used only by `stage_06_execution` and `safety` |

`.importlinter` encodes these contracts; `tests/invariants/test_hard_limits_independence.py` runs them in the test suite. The rules are documented in `docs/architecture/module_dependency_rules.md`.

## 7. Standard forecast contract

Forecasting models return a documented object, not arbitrary files. The contract is defined in `contracts/forecast_schema.py` and the shared quantile grid in `contracts/quantile_grid.py`. At minimum, record:

- security identifier, market and sleeve;
- decision timestamp, forecast creation timestamp and information cut-off;
- forecast horizon and target definition (benchmark-relative excess return starting after the decision point);
- quantile values on the shared grid {0.01, 0.025, 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95, 0.975, 0.99};
- label end date (used for purging during validation);
- model name and version, feature version, training window and data lineage;
- validation and eligibility status;
- warnings, missing-input flags and failure reason codes.

`forecast_validator.py` rejects invalid timestamps or schema and crossing quantiles. Forecasts with different targets, horizons or units are not compared or combined until explicitly aligned. Pinball loss is the primary quantile score; interval coverage, sharpness and the probability integral transform are tracked alongside it.

## 8. Sleeves and configuration

| Sleeve | Configuration file | Key concern |
|---|---|---|
| ASX interday | `configs/sleeves/sleeve_asx_interday.yaml` | Closing-auction execution and limited short borrow |
| US interday | `configs/sleeves/sleeve_us_interday.yaml` | FX exposure and overnight gap risk |
| ASX intraday | `configs/sleeves/sleeve_asx_intraday.yaml` | Spread, tick size and available depth |
| US intraday | `configs/sleeves/sleeve_us_intraday.yaml` | Execution costs and competition for short-horizon signals |

Sleeves share infrastructure but keep separate calendars, cost assumptions, data depth and risk budgets. Gate thresholds, target volatility, drawdown limits and position limits are provisional and calibrated per sleeve, never hard-coded. `configs/data_tiers.yaml` holds the T1/T2/T3 data-coverage allocations (example sizes in the design are budget illustrations, not requirements); tier membership is fixed within a universe version. Asynchronous ASX and US sessions are handled by `stages_03_04_portfolio/cross_market_alignment.py`.

## 9. Orchestration and expected run flow

```text
Run configuration, run ID, decision cut-off and sleeve
        ↓
Stage 00: Extract, standardise, version and validate data            [G2]
        ↓
Stage 01: Load or form the fixed research universe version          [G1]
        ↓
Stage 02: Forecast layers in parallel → common forecast schema
        ↓
Stage 02E: Validate, apply eligibility, combine eligible forecasts   [G3]
        ↓
Stages 03–04: Risk estimation, optimisation, sleeve allocation       [G4]
        ↓
Stage 05: Leverage, margin, trade decision, order requests           [G5, G6]
        ↓
Independent safety approval (pre-trade checks, kill switch)
        ↓
Stage 06: Execution simulation or approved broker adapter
        ↓
Stage 07: Monitoring, attribution and readiness
        ↓
Compile plots, diagnostics, run manifest and narrative report
```

The orchestrator (`orchestration/orchestrator.py`) is the only component that launches the full flow; stage modules never start the workflow themselves. It stops, blocks or marks a run degraded when an essential stage fails. A failed model may be skipped only if `configs/fallbacks.yaml` defines an approved fallback and the degradation is recorded. The narrative report describes recorded output; it must not invent results, override a gate or issue an order.

## 10. Required run outputs

Each run is written to an immutable folder, `outputs/runs/<run_id>/`:

```text
outputs/runs/<run_id>/
├── run_manifest.json
├── config_snapshot.yaml
├── data_lineage/        # lineage, data_quality_summary.csv, quarantine list
├── universe/            # universe version, selected_companies.csv, exclusions.csv, exits.csv, data tiers
├── forecasts/
├── validation/          # fold definitions, scores, eligibility decisions, trial log
├── portfolio/           # risk estimates, stress results, target weights, constraint relaxations
├── sizing/              # leverage components, order requests, position report
├── execution/           # orders, fills, reconciliation, implementation shortfall
├── safety/              # gate results, incidents, kill-switch events
├── monitoring/          # attribution, calibration, drift, alerts
├── plots/
└── reports/
    ├── run_summary.json
    └── run_report.md
```

| Manifest record | Required fields |
|---|---|
| Run and configuration | Start/end timestamps and time zone, code revision, configuration version and hash, market, sleeve, cut-off, calendar, horizons |
| Data lineage | Provider and dataset, extraction and observation timestamps, availability-time rule, row counts, schema version |
| Universe | Universe version, eligible names, exits, reason codes |
| Models and validation | Model and feature versions, training window, fit status, fold/purge/embargo settings, benchmark, calibration and net-of-cost results, trial count |
| Portfolio and decision | Forecasts, current and target weights, costs, risk contributions, binding constraints, `TRADE`/`REDUCE`/`HOLD`/`NO-TRADE` with reason codes |
| Execution | Order requests, submissions, fills, timestamps, fees, cancellations and rejections, implementation shortfall |
| Outputs and status | Output paths, warnings, failure reasons, per-sleeve and terminal status |

### Plot and report requirements

- Each model saves traceable plots into the current run's `plots/` folder, named with stable model, instrument or market, and plot-type identifiers.
- `plot_compiler.py` builds an index of produced plots and reports missing expected outputs.
- `narrative_report.py` uses `run_summary.json` as its only source of truth. Generated prose separates measured results, model interpretations, limitations and failures.
- If report generation fails, retain the structured run summary and mark the report stage failed. Plot and report failures are logged and never conceal safety failures. A run is not marked successful merely because earlier stages completed.

## 11. Model lifecycle and registry

```text
candidate → G3 eligible → shadow mode → G7 approved production model → retired
```

Monitoring may trigger a challenger review, but a challenger never replaces the production model automatically. Artefacts are stored under `models/` by state; metadata (versions, training windows, trial counts, approvals, rollback targets) is held by `storage/model_registry.py` and `storage/trial_registry.py`. Promotion requires independent review, a pre-agreed shadow period (the design suggests at least 60 sessions), intact calibration, evidence against the current approved model and a tested rollback path. `model_lifecycle` has no order authority.

## 12. Test batches and invariants

Run batches in order. Each batch has a README describing its objective, fixtures, command, expected artefacts and pass criteria. Automated tests use synthetic or sanitised data; credentials and proprietary raw data are never committed.

| Batch | Roadmap phase | Scope | Example pass condition |
|---|---|---|---|
| `test_batch_01_data_foundation` | 1 Data foundation | Schemas, calendars, identifiers, corporate actions, lineage, vintages, as-of queries, G2 | As-of queries and schema/timestamp/unit tests pass |
| `test_batch_02_universe_and_baselines` | 2 Baseline research | Company reference dataset, hard gates, scoring direction, strategy classification, fixed membership, benchmarks and cost baselines | Survivorship and look-ahead tests pass; clean runs reproduce results |
| `test_batch_03_validation_framework` | 3 Validation | Walk-forward folds, purging and embargo, trial registry, scoring and comparison tests | Leakage tests work; trials and fold boundaries are logged |
| `test_batch_04_forecast_layers` | 4 Forecast layers | Financial, macro, micro and news modules under the common contract; failure states | Baselines, calibration and model failure states are measured |
| `test_batch_05_risk_optimisation_sizing` | 5 Risk and optimisation (including Stage 5 sizing) | Covariance and tail risk, stress library, optimiser, relaxation policy, Kelly/volatility/drawdown caps, margin | Infeasibility cannot create tradable weights; invalid inputs never raise leverage |
| `test_batch_06_execution_simulation` | 6 Execution simulation | Order intents, scheduling, costs, fills, slippage, partial fills, reconciliation | Simulated fills reconcile and prohibited orders are blocked |
| `test_batch_07_safety_and_paper_trading` | 7 Paper trading | Hard limits, kill switch, stale-data, restart, latency, orchestrator, run manifest, reports | Reconciliation, stale-data, restart and kill-switch tests pass; statuses are truthful |
| `test_batch_08_challenger_lifecycle` | 9 Challenger lifecycle | Shadow runs, G7, versioned release and rollback | Promotion requires criteria and rollback is available |

Phase 8 (controlled deployment) is a sign-off process, not a test batch: it needs written approval, limits, rollback and incident-response testing outside the automated suite.

`tests/invariants/` holds design rules that must hold for every release regardless of batch: no look-ahead, survivorship safety, fixed universe membership, infeasible optimisation means no trade, independence of hard limits, status semantics, and narrative reports built only from structured outputs. A passing batch establishes only the behaviour it covers; it does not establish profitability or live-trading readiness.

## 13. Windows and VS Code setup

From PowerShell, open the repository root and create a project virtual environment:

```powershell
cd path\to\AQP-PYTHON
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e .
pip install -r requirements-dev.txt
code .
```

If PowerShell blocks activation, use the terminal profile or activation approach permitted by your Windows policy; do not weaken system-wide execution policy for this project. Configure VS Code to use `AQP-PYTHON\.venv\Scripts\python.exe` as the interpreter (`.vscode/settings.json`).

Intended command interfaces once the CLI exists:

```powershell
python -m aqp_python.cli validate-config --config configs/pipeline.yaml
python -m aqp_python.cli run --config configs/pipeline.yaml --sleeve asx_interday --mode research
pytest tests/invariants
pytest
```

These commands work only after the package, CLI and dependencies have been implemented.

## 14. Data, credentials and Git hygiene

- Keep LSEG, news, macro and broker credentials outside source control. Use environment variables or an approved secret store; `.env.example` contains placeholder names only.
- Exclude `data/raw`, `data/vintages`, `data/interim`, `data/processed`, `models/` artefacts, `outputs/` and `build/` from Git unless a specific small, licensed, sanitised fixture is deliberately approved.
- Do not commit `.venv/`, caches, logs, large plot collections or run outputs.
- `.gitkeep` makes an otherwise-empty directory appear in Git; it does not make hidden files visible or force ignored files to be tracked.
- Respect provider licences: a successful provider connection does not permit redistribution of downloaded data.
- Pin and record dependency versions; use a lock file if the project adopts a dependency manager that supports one.
- Broker access starts with `connectors/broker/paper_broker.py`. Live broker permissions are enabled only after the controlled-deployment sign-off.

## 15. Python and R boundary

Python coordinates the pipeline and owns the schemas, data contracts, run manifest, gates, portfolio controls, compiler and reporting. R implementations live under `r_models/scripts/` and are called only through `r_models/adapters/r_model_adapter.py`. The adapter's input and output types are defined in `r_models/schemas/r_io_schema.json`, together with timeout behaviour, error handling and package requirements. A failed R model is never silently turned into a successful forecast; it becomes a recorded model failure and follows the fallback rules. Current R candidates are `realised_garch.R` and `dcc_garch.R`.

## 16. Implementation order

| Step | Roadmap phase | Build |
|---|---|---|
| 1 | 1 Data foundation | Tooling, configuration validation, contracts, reason codes, run status, run manifest; vintage store, calendars, identifiers, corporate actions, G2 |
| 2 | 2 Baseline research | Company reference dataset, universe stage with versioning and mandatory removals, benchmarks and cost baselines |
| 3 | 3 Validation | Forecast schema and validator, walk-forward folds, purging, embargo, trial registry |
| 4 | 4 Forecast layers | Stage 02 models one layer at a time, each with baselines and failure states; then eligibility and combination (G3) |
| 5 | 5 Risk and optimisation | Covariance, tail risk and backtests, stress library, optimiser, relaxation policy, sleeve allocator (G4); then Stage 5 sizing (G5, G6) |
| 6 | 6 Execution simulation | Order lifecycle, scheduling, costs, fills, shortfall; independent safety package tests before any broker adapter |
| 7 | 7 Paper trading | Monitoring, plots, narrative report, scheduled end-to-end runs with the paper broker |
| 8 | 8 Controlled deployment | Small approved allocation under written sign-off, limits, rollback and incident response |
| 9 | 9 Challenger lifecycle | Shadow runner, promotion review (G7), release and rollback |

A stage is complete only when its input/output schema, unit and failure tests, point-in-time handling, versioned configuration, logs and reason codes, reproducible outputs, baseline and invariants, and downstream failure actions are documented (`docs/readiness/stage_completion_checklist.md`).

## 17. Open design decisions that affect the repository

The design lists decisions required before implementation. Until resolved, they are tracked in `docs/decisions/open_decisions.md` and kept out of code.

| Decision | Repository impact |
|---|---|
| Instruments and capital structure (cash equities on margin, CFDs, futures) | `configs/execution.yaml`, `leverage.yaml`, margin and financing modules |
| Per-sleeve target volatility, drawdown, name/sector/gross limits, margin buffer | `configs/sleeves/*`, `risk_limits.yaml` |
| US broker and data provider; licensed intraday coverage and cost | `connectors/broker`, `connectors/lseg`, `data_sources.yaml`, `data_tiers.yaml` |
| News sources and latency; directional intraday use | `connectors/news`, `stage_02_forecasting/news`, `forecast_models.yaml` |
| Long-only versus long-short; borrow, especially ASX | `borrow_margin_extractor.py`, `financing_borrow_costs.py`, sleeve files |
| Universe re-formation: deliberate only or scheduled | `universe_versioning.py`, `universe_rules.yaml` |
| Fixed model families or an open challenger process | `model_lifecycle`, `model_lifecycle.yaml` |

## 18. Key definitions

- **AQP:** Advanced Quantitative Pipeline.
- **Sleeve:** A separately governed sub-portfolio (ASX interday, US interday, ASX intraday, US intraday) with its own calendar, costs, data depth and risk budget.
- **HFD / LFD:** High-frequency data (intraday observations) and low-frequency data (daily or less frequent).
- **Point-in-time data:** Data represented by when it could first have been used. Each fact has a valid time and an availability timestamp; restatements create new vintages.
- **Company Reference and Exposure Dataset:** Project-defined, versioned dataset of geographic revenue and cost exposures, classifications, peer clusters, the relationship graph and membership history.
- **Research universe:** Fixed, versioned set of companies eligible for analysis; not a list of automatic trades.
- **Strategy classification:** Routes an eligible company to relevant modelling hypotheses (trend, quality-value, pairs, event, intraday).
- **Gate (G1–G7):** Project-defined checkpoint with a pass condition and failure action.
- **Forecast eligibility:** Validated status allowing a model's output into the signal-combination stage.
- **Hard limit:** Mandatory risk or operational constraint that no forecast, optimiser or report can override.
- **Run manifest:** Machine-readable record of configuration, data, versions, statuses and outputs for one run.
- **Champion / challenger:** The current approved model and a candidate under independent evaluation.

---

**Repository rule:** keep this file as the navigation map. Put extended mathematics and model derivations in `docs/methodology/`, executable code in `src/aqp_python/`, tests in `tests/`, and generated run artefacts under `outputs/`. Research models propose forecasts; the validated pipeline proposes target positions; only independent risk and execution controls authorise orders.
