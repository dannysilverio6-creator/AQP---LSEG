# AQP-PYTHON — Python Repository Overview

**Project:** AQP — Advanced Quantitative Pipeline (Multi-Market Quantitative Investment Pipeline, design specification v14)  
**Repository:** `AQP-PYTHON`  
**Layout version:** 3 — aligned to Architecture Version 14 (filter-first data collection and paired news benchmarks); supersedes layout v2 (design v12)  
**Primary language:** Python  
**Companion language:** R, kept separate and called through a controlled adapter  
**Intended environment:** Windows, Visual Studio Code, Git and a project-specific Python virtual environment  
**Status:** Proposed repository layout and implementation guide. No pipeline logic has been implemented. The empty shell can be regenerated from `scripts/build_repository_shell.py`.

---

## 1. Purpose and status

`AQP-PYTHON` contains the Python implementation of the Advanced Quantitative Pipeline. It follows the v14 operating order: screening data for every candidate → fixed research universe (G1) → detailed data for admitted names only → shared core features $X^{core}$ and a separate NLP-derived news vector $Z^{news}$ → matched no-news baseline and news-augmented challenger forecasts → paired validation (G3) → risk and optimisation → leverage and pre-trade gates → execution → monitoring and reporting, across four sleeves (ASX interday, US interday, ASX intraday, US intraday).

This document defines the **target repository structure**. A path shown in the tree is a planned location until its module has been implemented and tested. The presence of a file here does not imply that the corresponding model is complete, validated or approved for live trading, and a sophisticated structure does not establish an economic edge.

**Shell convention.** The shell contains folders, `__init__.py` files and zero-byte placeholders for non-Python files (configuration, documentation, R scripts). The 323 planned Python modules shown in the tree (258 under `src/`, 1 R adapter, 7 scripts and 57 test modules) are **not** created in the shell; only the 37 `__init__.py` package markers are. The 25 folders that would otherwise be empty hold a `.gitkeep` so Git tracks them. The tree in Section 3 and these counts are generated from the same specification inside `scripts/build_repository_shell.py`, so the document and the shell cannot drift apart.

### 1.1 Design rules and where the repository enforces them

| Design rule (v14) | Repository mechanism |
|---|---|
| Point-in-time data; no look-ahead | `data_foundation/as_of_join.py`, `point_in_time_controls.py`, `storage/vintage_store.py` (valid time and availability time), gate G2, `tests/invariants/test_no_lookahead.py` |
| Filter before deep retrieval | `stage_00_screening` collects screen-level data only; `stage_02_data_expansion/data_request_manifest.py` refuses a request without a G1-passed universe version; `tests/invariants/test_no_deep_retrieval_before_g1.py` |
| Fixed, versioned universe; mandatory removals only | `stage_01_universe/universe_versioning.py`, `mandatory_removals.py`, `storage/universe_store.py`, `data/universes/`, `tests/invariants/test_universe_fixed_membership.py` |
| Selection is separate from forecasting and from news | `stage_00_screening` and `stage_01_universe` never import any `stage_02_*` package; `tests/invariants/test_news_not_used_for_admission.py` |
| Coverage tier governs data depth, not membership | `stage_01_universe/coverage_tier_assignment.py`, `configs/coverage_tiers.yaml`, read by `data_request_manifest.py` |
| News is a feature engine, not a fourth forecast family | `stage_02d_news_features` publishes $Z^{news}$ to `storage/feature_store.py`; it has no forecast schema output and no broker access |
| Isolate the news experiment | `contracts/experiment_schema.py`, `configs/experiments.yaml`, `storage/experiment_registry.py`, `stage_02_forecasting/arm_builder.py`, `stage_02e_validation/arm_parity_check.py`, `tests/invariants/test_arms_differ_only_by_news.py` |
| Missing news is never zero | `stage_02_data_expansion/missingness_policy.py`, `stage_02d_news_features/feature_qc.py`, `tests/invariants/test_missing_news_not_zero_filled.py` |
| One forecast schema (quantile grid, lineage, label end date) | `contracts/forecast_schema.py`, `quantile_grid.py`, `stage_02_forecasting/forecast_validator.py` |
| Validate before weighting; charge for search | `stage_02e_validation/`, `storage/trial_registry.py`, gate G3 |
| Hard limits independent of models | `safety/` plus import contracts in `.importlinter` and `tests/invariants/test_hard_limits_independence.py` |
| Relative weights, total leverage and order authorisation are separate | Optimiser in `stages_03_04_portfolio`, leverage in `stage_05_sizing`, order authority in `stage_06_execution` and `safety` |
| Infeasible optimisation never creates tradable weights | `relaxation_policy.py`, `tests/invariants/test_infeasible_means_no_trade.py` |
| Reproducible runs with manifest | `orchestration/run_manifest_writer.py`, `storage/run_store.py`, `storage/data_request_store.py`, `scripts/replay_run.py` |
| Controlled model promotion (no automatic replacement) | `model_lifecycle/`, gate G7, `models/` lifecycle folders, `tests/invariants/test_no_auto_promotion.py` |
| Narrative is generated from structured outputs only | `reporting/report_fact_checks.py`, `tests/invariants/test_narrative_uses_structured_outputs.py` |

### 1.2 What changed from layout v2

| Area | Change | Design basis (v14) |
|---|---|---|
| Stage 0 | `stage_00_data` becomes `stage_00_screening`: candidate registry, screening request, screen-level extractor, intraday-coverage and source-availability probes, screening snapshot and coverage inventory. No deep history, no NLP features | Stage 0 "Screening scope versus full data scope" |
| Shared PIT utilities | Identifier mapping, calendars, units/currency, time zones, corporate actions, as-of joins, freshness checks, validator and quality report move to a new `data_foundation/` package used by both Stage 0 and Stage 2 | "These rules apply both to Stage 0 screening inputs and to all detailed sources retrieved after Stage 1" |
| Stage 2 data expansion | New `stage_02_data_expansion/`: data-request manifest (universe version + sleeve + cutoff + tier), context-node scope, detailed extractors, harmonisation, HFD/LFD builders, missingness policy and the $X^{core}$ feature builder | Sections 2.1–2.2 |
| Company Reference and Exposure Dataset | Builders move from Stage 0 to `stage_02_data_expansion/reference/`; assembled only for admitted names and recorded context nodes | "Company Reference and Exposure data is fully assembled here" |
| Universe stage | Adds `sector_specific_metrics.py` (banks and insurers), `admission_capacity.py`; `data_tier_assignment.py` renamed `coverage_tier_assignment.py` | Stage 1 filters, admission threshold and capacity, coverage tiers |
| News | `stage_02_forecasting/news/` becomes the separate `stage_02d_news_features/` engine. `news_event_forecast.py` removed; the calibrated logistic model becomes `event_risk_calibration.py` (a feature, not a forecast). Adds evidence validation, novelty/surprise, decay, aggregation, latency admissibility, feature QC, coverage report and publisher | Stage 2D |
| Matched arms | `arm_builder.py` joins $X^{core}$ to both arms and $Z^{news}$ to the challenger only; `experiment_schema.py`, `configs/experiments.yaml`, `storage/experiment_registry.py` and `scripts/register_experiment.py` register pairs before testing | Sections 2.2 step 6 and 2.8 |
| Validation | `stage_02_forecasting/validation/` becomes `stage_02e_validation/`, adding experiment pairing, arm parity, coverage matching, paired loss differences, block bootstrap, multiple-testing control, feature ablation, incremental-value test, slot selection and conformal calibration | Stage 2E |
| Firm/microstructure | `micro_market/` renamed `firm_microstructure/` to match the design's family name | Stage 2C |
| Feature stores | `data/features/core/`, `data/features/news/`, `data/text_corpus/`, `data/screening/`, `data/data_requests/`; `storage/feature_store.py`, `text_corpus_store.py`, `screening_store.py`, `data_request_store.py` | Stage 2 data hand-offs |
| Connectors | New `connectors/llm/` (used only by Stage 2D) and `filings_transcripts_provider.py` | Stage 2D processing |
| Monitoring | Adds `news_feature_drift_monitor.py` and `news_latency_coverage_monitor.py` | "Forecast/news drift" in Stage 7 |
| Run manifest | Adds a news-feature record and the experiment ID and arm (`BASELINE` / `NEWS_CHALLENGER`) | Operating contract |
| Tests | Nine ordered batches aligned to the nine v14 roadmap phases; six new invariants (no deep retrieval before G1, news not used for admission, baseline has no news features, arms differ only by news, missing news not zero-filled, no automatic promotion) | Implementation roadmap |
| Docs | Frozen v14 overview plus the eleven split stage documents; new `news_feature_contract.md` and `paired_experiment_protocol.md` | Stage document map |

## 2. Naming convention

| Item | Convention | Example |
|---|---|---|
| Repository folder | `AQP-PYTHON` | `AQP-PYTHON/` |
| Python package | `aqp_python` | `src/aqp_python/` |
| Custom Python modules and scripts | Descriptive `snake_case`, no project prefix | `universe_formation.py` |
| Package markers | Standard name, no prefix | `__init__.py` |
| Stage folders | Stage number (with sub-stage letter where the design has one) and name | `stages/stage_02d_news_features/` |
| Configuration files | Descriptive `snake_case`, YAML | `risk_limits.yaml` |
| Sleeve configuration | `sleeve_<market>_<horizon>.yaml` | `sleeve_asx_intraday.yaml` |
| Experiment IDs | `<family>__<target>__h<horizon>__<sleeve>` | `financial__xret__h5__asx_interday` |
| Experiment arms | Upper-case enum values | `BASELINE`, `NEWS_CHALLENGER` |
| R scripts | Descriptive `snake_case` | `dcc_garch.R` |
| Python tests | `test_*.py` | `test_gates.py` |
| Test batches | Number and roadmap phase | `test_batch_06_paired_evaluation/` |
| Documentation | Plain names, `.md` | `forecast_contract.md` |
| Generated run folders | Unique run ID, never overwritten | `outputs/runs/<run_id>/` |

Only the repository (`AQP-PYTHON`) and the package (`aqp_python`) carry the project name; individual modules, scripts and configuration files do not repeat it. The experiment-ID pattern is a repository convention to be confirmed in `docs/decisions/decision_log.md`.

## 3. Repository tree

Legend: `.py` files other than `__init__.py` are planned module locations and are not present in the shell. Folders without a listed file (for example `data/raw/screening/`) contain a `.gitkeep`.

```text
AQP-PYTHON/
├── README.md
├── PY REPOSITORY OVERVIEW.md                                       # this navigation map (layout v3)
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── .gitignore
├── .gitattributes
├── .editorconfig
├── .env.example                                                    # placeholder names only, never real credentials
├── .pre-commit-config.yaml
├── .importlinter                                                   # import contracts: filter-first order, news-engine isolation, safety independence, read-only reporting
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   ├── tasks.json
│   └── extensions.json
├── configs/                                                        # all limits, thresholds, experiments and approvals live here, never in code
│   ├── pipeline.yaml
│   ├── markets.yaml
│   ├── calendars.yaml
│   ├── data_sources.yaml
│   ├── screening_request.yaml                                      # Stage 0: minimum screening fields, history and coverage probes
│   ├── detailed_data_request.yaml                                  # Stage 2: per-tier sources, frequency, depth and freshness
│   ├── coverage_tiers.yaml                                         # T1/T2/T3 budgets; governs the Stage 2 request only
│   ├── gates.yaml
│   ├── reason_codes.yaml
│   ├── fallbacks.yaml
│   ├── universe_rules.yaml
│   ├── forecast_models.yaml
│   ├── news_features.yaml                                          # sources, latency budgets, decay, extraction schema version, missingness policy
│   ├── experiments.yaml                                            # registered baseline/challenger pairs per family x target x horizon x sleeve
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
│   ├── design/                                                     # frozen copy of the v14 design specification
│   │   ├── AQP_OVERVIEW.md
│   │   └── stages/
│   │       ├── 00_aqp_overall_design.md
│   │       ├── 01_stage_0_screening_and_point_in_time.md
│   │       ├── 02_stage_1_universe_selection_and_classification.md
│   │       ├── 03_stage_2_data_expansion_forecasting_news_validation.md
│   │       ├── 04_stages_3_4_portfolio_risk_optimisation_sleeves.md
│   │       ├── 05_stage_5_leverage_sizing_position_report.md
│   │       ├── 06_stages_6_7_execution_safety_monitoring_learning.md
│   │       ├── 07_notation_glossary_open_decisions.md
│   │       ├── 08_operating_contract_and_run_outputs.md
│   │       ├── 09_implementation_roadmap_and_readiness_gates.md
│   │       └── 10_selected_technical_references.md
│   ├── architecture/
│   │   ├── system_architecture.md
│   │   ├── data_dictionary.md
│   │   ├── model_registry.md
│   │   ├── forecast_contract.md
│   │   ├── news_feature_contract.md
│   │   ├── paired_experiment_protocol.md
│   │   ├── run_contract.md
│   │   ├── gates_and_reason_codes.md
│   │   └── module_dependency_rules.md
│   ├── methodology/
│   │   ├── stage_00_screening.md
│   │   ├── stage_01_universe.md
│   │   ├── stage_02_data_expansion.md
│   │   ├── stage_02_forecast_contract.md
│   │   ├── stage_02a_financial.md
│   │   ├── stage_02b_macro.md
│   │   ├── stage_02c_firm_microstructure.md
│   │   ├── stage_02d_news_features.md
│   │   ├── stage_02e_paired_validation.md
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
│   │   ├── screening/                                              # Stage 0 pulls: screen-level fields for every candidate
│   │   └── detailed/                                               # Stage 2 pulls: admitted names and context nodes only
│   ├── vintages/                                                   # point-in-time store: valid time + availability timestamp; excluded from Git
│   ├── interim/
│   ├── processed/                                                  # validated, analysis-ready source tables
│   ├── screening/                                                  # versioned candidate registry, screening snapshots, coverage inventory
│   ├── universes/                                                  # frozen, versioned universe memberships, scores, tiers and exit logs
│   ├── data_requests/                                              # frozen Stage 2 data-request manifests (universe version + sleeve + cutoff + tier)
│   ├── reference/                                                  # symbol mappings, calendars, classification tables
│   ├── company_reference/                                          # versioned Company Reference and Exposure Dataset (assembled in Stage 2)
│   ├── text_corpus/                                                # licensed raw text for Stage 2D; excluded from Git; access-controlled
│   ├── features/
│   │   ├── core/                                                   # X_core feature store; no NLP-derived variables
│   │   └── news/                                                   # Z_news feature store; versioned, with lineage and missingness flags
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
│       │   ├── sleeve_config.py
│       │   └── experiment_config.py
│       ├── contracts/                                              # shared schemas: the only layer every other package may depend on
│       │   ├── __init__.py
│       │   ├── data_schemas.py
│       │   ├── screening_schema.py
│       │   ├── universe_schema.py
│       │   ├── data_request_schema.py
│       │   ├── core_feature_schema.py
│       │   ├── news_feature_schema.py
│       │   ├── event_extraction_schema.py
│       │   ├── quantile_grid.py
│       │   ├── forecast_schema.py
│       │   ├── experiment_schema.py
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
│       ├── data_foundation/                                        # shared point-in-time utilities used by Stage 0 and Stage 2
│       │   ├── __init__.py
│       │   ├── identifier_mapper.py
│       │   ├── calendar_manager.py
│       │   ├── unit_currency_standardiser.py
│       │   ├── timezone_normaliser.py
│       │   ├── corporate_actions.py
│       │   ├── as_of_join.py
│       │   ├── point_in_time_controls.py
│       │   ├── freshness_checks.py
│       │   ├── data_validator.py
│       │   └── data_quality_report.py
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
│       │   │   ├── news_provider.py
│       │   │   └── filings_transcripts_provider.py
│       │   ├── llm/                                                # used only by stage_02d_news_features
│       │   │   ├── __init__.py
│       │   │   └── llm_client.py
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
│       │   ├── screening_store.py
│       │   ├── universe_store.py
│       │   ├── data_request_store.py
│       │   ├── text_corpus_store.py
│       │   ├── feature_store.py
│       │   ├── experiment_registry.py
│       │   ├── trial_registry.py
│       │   ├── model_registry.py
│       │   └── run_store.py
│       ├── stages/
│       │   ├── __init__.py
│       │   ├── stage_00_screening/                                 # screen-level data only; no deep retrieval, no NLP features
│       │   │   ├── __init__.py
│       │   │   ├── candidate_registry.py
│       │   │   ├── screening_request.py
│       │   │   ├── extractors/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── screening_extractor.py
│       │   │   │   ├── intraday_coverage_probe.py
│       │   │   │   └── source_availability_probe.py
│       │   │   ├── screening_standardiser.py
│       │   │   ├── screening_snapshot.py
│       │   │   ├── coverage_inventory.py
│       │   │   └── screening_publisher.py
│       │   ├── stage_01_universe/
│       │   │   ├── __init__.py
│       │   │   ├── candidate_universe.py
│       │   │   ├── corporate_status_filter.py
│       │   │   ├── distress_screen.py
│       │   │   ├── sector_specific_metrics.py
│       │   │   ├── liquidity_screen.py
│       │   │   ├── data_history_check.py
│       │   │   ├── stability_tests.py
│       │   │   ├── strategy_classification.py
│       │   │   ├── score_standardisation.py
│       │   │   ├── composite_score.py
│       │   │   ├── admission_capacity.py
│       │   │   ├── universe_formation.py
│       │   │   ├── universe_versioning.py
│       │   │   ├── coverage_tier_assignment.py
│       │   │   └── mandatory_removals.py
│       │   ├── stage_02_data_expansion/                            # 2.1-2.2: detailed data for G1-admitted names only
│       │   │   ├── __init__.py
│       │   │   ├── data_request_manifest.py
│       │   │   ├── context_node_scope.py
│       │   │   ├── extractors/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── financial_extractor.py
│       │   │   │   ├── macro_extractor.py
│       │   │   │   ├── microstructure_extractor.py
│       │   │   │   ├── borrow_margin_extractor.py
│       │   │   │   └── text_corpus_extractor.py
│       │   │   ├── harmonisation.py
│       │   │   ├── hfd_builder.py
│       │   │   ├── lfd_builder.py
│       │   │   ├── missingness_policy.py
│       │   │   ├── core_feature_builder.py
│       │   │   └── reference/                                      # Company Reference and Exposure Dataset builders
│       │   │       ├── __init__.py
│       │   │       ├── company_reference_builder.py
│       │   │       ├── geographic_exposures.py
│       │   │       ├── classifications.py
│       │   │       ├── peer_clusters.py
│       │   │       ├── relationship_graph.py
│       │   │       └── membership_history.py
│       │   ├── stage_02_forecasting/                               # 2.3 contract + 2A-2C families; each slot has a baseline and a news-challenger arm
│       │   │   ├── __init__.py
│       │   │   ├── forecast_runner.py
│       │   │   ├── forecast_registry.py
│       │   │   ├── forecast_adapter.py
│       │   │   ├── arm_builder.py
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
│       │   │   └── firm_microstructure/
│       │   │       ├── __init__.py
│       │   │       ├── micro_forecast.py
│       │   │       ├── interday/
│       │   │       │   ├── __init__.py
│       │   │       │   ├── panel_features.py
│       │   │       │   ├── panel_model.py
│       │   │       │   ├── hierarchical_shrinkage.py
│       │   │       │   ├── dml_effects.py
│       │   │       │   └── peer_earnings_response.py
│       │   │       └── intraday/
│       │   │           ├── __init__.py
│       │   │           ├── trade_quote_cleaning.py
│       │   │           ├── trade_intensity.py
│       │   │           ├── order_flow_imbalance.py
│       │   │           ├── price_impact.py
│       │   │           ├── realised_measures.py
│       │   │           ├── liquidity_state.py
│       │   │           └── execution_cost_estimator.py
│       │   ├── stage_02d_news_features/                            # shared feature engine: text -> Z_news; not a forecast family, no order authority
│       │   │   ├── __init__.py
│       │   │   ├── text_ingestion.py
│       │   │   ├── source_timing_check.py
│       │   │   ├── story_deduplication.py
│       │   │   ├── entity_linking.py
│       │   │   ├── prompt_registry.py
│       │   │   ├── llm_event_extractor.py
│       │   │   ├── evidence_validator.py
│       │   │   ├── event_quantification.py
│       │   │   ├── novelty_surprise.py
│       │   │   ├── time_decay.py
│       │   │   ├── entity_aggregation.py
│       │   │   ├── graph_propagation.py
│       │   │   ├── event_risk_calibration.py
│       │   │   ├── latency_admissibility.py
│       │   │   ├── feature_qc.py
│       │   │   ├── news_coverage_report.py
│       │   │   └── news_feature_publisher.py
│       │   ├── stage_02e_validation/                               # absolute eligibility + paired incremental-value tests -> G3
│       │   │   ├── __init__.py
│       │   │   ├── walk_forward_folds.py
│       │   │   ├── purging_embargo.py
│       │   │   ├── experiment_pairing.py
│       │   │   ├── arm_parity_check.py
│       │   │   ├── coverage_matching.py
│       │   │   ├── baseline_comparison.py
│       │   │   ├── forecast_scoring.py
│       │   │   ├── forecast_calibration.py
│       │   │   ├── forecast_comparison_tests.py
│       │   │   ├── paired_loss_differences.py
│       │   │   ├── block_bootstrap.py
│       │   │   ├── multiple_testing.py
│       │   │   ├── feature_ablation.py
│       │   │   ├── overfitting_controls.py
│       │   │   ├── stability_checks.py
│       │   │   ├── net_of_cost_evaluation.py
│       │   │   ├── model_eligibility.py
│       │   │   ├── incremental_value_test.py
│       │   │   ├── slot_selection.py
│       │   │   ├── signal_combiner.py
│       │   │   ├── conformal_calibration.py
│       │   │   └── horizon_selection.py
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
│       │       ├── news_feature_drift_monitor.py
│       │       ├── news_latency_coverage_monitor.py
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
│       ├── safety/                                                 # independent hard limits; must not import forecast, news, portfolio, sizing or reporting code
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
│           ├── experiment_summary.py
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
│   ├── build_repository_shell.py
│   ├── run_pipeline.py
│   ├── run_sleeve.py
│   ├── register_experiment.py
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
│   │   ├── test_as_of_join.py
│   │   ├── test_vintage_store.py
│   │   ├── test_data_validator.py
│   │   ├── test_candidate_registry.py
│   │   ├── test_coverage_inventory.py
│   │   ├── test_universe_scoring.py
│   │   ├── test_universe_removals.py
│   │   ├── test_coverage_tiers.py
│   │   ├── test_data_request_manifest.py
│   │   ├── test_company_reference.py
│   │   ├── test_core_feature_builder.py
│   │   ├── test_forecast_schema.py
│   │   ├── test_quantile_grid.py
│   │   ├── test_arm_builder.py
│   │   ├── test_news_timing.py
│   │   ├── test_entity_linking.py
│   │   ├── test_event_extraction_schema.py
│   │   ├── test_graph_propagation.py
│   │   ├── test_event_risk_calibration.py
│   │   ├── test_walk_forward_folds.py
│   │   ├── test_purging_embargo.py
│   │   ├── test_paired_loss_differences.py
│   │   ├── test_block_bootstrap.py
│   │   ├── test_model_eligibility.py
│   │   ├── test_slot_selection.py
│   │   ├── test_signal_combiner.py
│   │   ├── test_covariance.py
│   │   ├── test_portfolio_constraints.py
│   │   ├── test_kelly_leverage.py
│   │   ├── test_no_trade_band.py
│   │   ├── test_execution_simulator.py
│   │   ├── test_safety_controls.py
│   │   └── test_run_manifest.py
│   ├── integration/
│   │   ├── test_screening_to_universe.py
│   │   ├── test_universe_to_data_request.py
│   │   ├── test_core_and_news_feature_stores.py
│   │   ├── test_stage_handoffs.py
│   │   ├── test_run_artifacts.py
│   │   ├── test_fallback_behaviour.py
│   │   └── test_r_adapter.py
│   ├── invariants/                                                 # design rules that must hold for every release
│   │   ├── test_no_lookahead.py
│   │   ├── test_survivorship_safety.py
│   │   ├── test_no_deep_retrieval_before_g1.py
│   │   ├── test_news_not_used_for_admission.py
│   │   ├── test_universe_fixed_membership.py
│   │   ├── test_baseline_has_no_news_features.py
│   │   ├── test_arms_differ_only_by_news.py
│   │   ├── test_missing_news_not_zero_filled.py
│   │   ├── test_no_auto_promotion.py
│   │   ├── test_infeasible_means_no_trade.py
│   │   ├── test_hard_limits_independence.py
│   │   ├── test_status_semantics.py
│   │   └── test_narrative_uses_structured_outputs.py
│   └── test_batches/                                               # run in order; one batch per roadmap phase; each batch has a README
│       ├── test_batch_01_screening_foundation/
│       │   └── README.md
│       ├── test_batch_02_universe_formation/
│       │   └── README.md
│       ├── test_batch_03_data_expansion/
│       │   └── README.md
│       ├── test_batch_04_baseline_models/
│       │   └── README.md
│       ├── test_batch_05_news_feature_engine/
│       │   └── README.md
│       ├── test_batch_06_paired_evaluation/
│       │   └── README.md
│       ├── test_batch_07_risk_optimisation_sizing/
│       │   └── README.md
│       ├── test_batch_08_execution_and_controls/
│       │   └── README.md
│       └── test_batch_09_monitoring_shadow_release/
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
| **00 — Screening** | `stages/stage_00_screening`, `data_foundation` | Load the candidate list; collect only screen-level identifiers, listing status, basic financial condition, descriptors, interday history and intraday-coverage metadata; standardise identifiers, calendars, units and time zones; apply as-of rules | Versioned candidate registry, screening snapshot, coverage inventory, G2 evidence | G2 |
| **01 — Universe** | `stages/stage_01_universe` | Corporate-status and distress screens (sector-appropriate for banks and insurers), liquidity and tradability, data depth and stability, strategy classification, robust scoring, admission threshold and capacity, coverage tiers, versioning, mandatory removals | Frozen, versioned research universe with tiers, reason codes and exit log | G1 |
| **02 — Data expansion** | `stages/stage_02_data_expansion`, `data_foundation` | Freeze the data request; retrieve detailed financial, macro, firm/microstructure and text data for admitted names and scoped context nodes; harmonise; build HFD/LFD; assemble the Company Reference and Exposure Dataset; build $X^{core}$ | Reproducible data-request manifest, validated PIT source tables, $X^{core}$, raw text corpus | G2 |
| **02 — Forecast contract** | `contracts`, `stage_02_forecasting` | Shared quantile grid, excess-return target, label end date, schema and timestamp validation, matched arm construction | Comparable predictive distributions for both arms | — |
| **02A — Financial** | `stage_02_forecasting/financial` | Return quantiles, GJR-GARCH/EGARCH and HAR-RV volatility, distress hazard, filtered dynamic exposures; simple baselines first | Return quantiles, volatility, distress probability | — |
| **02B — Macro** | `stage_02_forecasting/macro` | Dynamic factors, regimes, MIDAS/BVAR nowcasts, separate revenue and cost exposures, shrunk sensitivities, release surprises, local-projection transmission, FX | Macro contribution and quantiles | — |
| **02C — Firm/microstructure** | `stage_02_forecasting/firm_microstructure` | Interday firm/peer panels, hierarchical shrinkage, DML where justified; intraday cleaning, intensity, order-flow imbalance, impact, realised measures, liquidity state, execution costs | Firm/microstructure contribution and quantiles | — |
| **02D — News features** | `stages/stage_02d_news_features` | Timing and provenance checks, deduplication, entity linking, fixed-schema LLM extraction with evidence, quantification, novelty, decay, aggregation, graph propagation, calibrated event-risk feature, latency admissibility, QC | Versioned $Z^{news}$ with lineage, coverage and missingness flags (covariates, not forecasts) | G2 timing rules |
| **02E — Validation** | `stages/stage_02e_validation` | Walk-forward folds with purging and embargo; absolute skill and calibration; paired baseline-versus-challenger tests with uncertainty, ablation and multiple-testing control; net-of-cost evaluation; slot selection; combination and conformal calibration | Eligibility and incremental-value decisions; combined distributions; `NO-TRADE` if no eligible forecast | G3 |
| **03–04 — Portfolio** | `stages/stages_03_04_portfolio` | Ledoit–Wolf, factor, DCC and regime covariance; tail-risk engines with backtests; stress library; Black–Litterman; cost-aware optimisation; constrained relaxation; sleeve allocation | Target weights, risk diagnostics, binding constraints | G4 |
| **05 — Sizing** | `stages/stage_05_sizing` | Kelly and volatility caps, margin capacity, drawdown multiplier, overnight gap risk, financing, trade decision, no-trade band, invalidation conditions, order requests, position report | Bounded exposure, order requests, position report with reason codes | G5, G6 |
| **06 — Execution** | `stages/stage_06_execution` | Scheduling, routing through the broker interface, order lifecycle, simulation, cost and slippage, reconciliation, implementation shortfall | Orders, fills, costs, reconciliation records | Pre-trade checks |
| **07 — Monitoring** | `stages/stage_07_monitoring` | Attribution, calibration, data/news/model drift, news latency and coverage, risk backtests, execution error, alert rules, readiness | Diagnostics, alerts, readiness status | Triggers G7 review |
| **Safety** | `safety` | Hard limits, kill switch, reconciliation block, incident log, restart approval | Allow/block decisions with reason codes | Independent of models |
| **Model lifecycle** | `model_lifecycle` | Candidate registry, challenger training, shadow runs, promotion review, versioned release, rollback | Approved-model changes with audit trail | G7 |
| **Compiler / reporting** | `compiler`, `reporting` | Index plots and run artefacts; metrics and experiment summaries; narrative report with fact checks | Indexed plots and a report tied to the run ID | — |

### 4.1 Component reference by package

The table above gives each stage's job. The lists below say what the main modules in each package compute, so a reader can find the code for a formula or rule in the design.

**`data_foundation/`** — shared by Stage 0 and Stage 2.

- `as_of_join.py`: joins sources at decision time $t$ using only versions with availability timestamp $a \le t$.
- `point_in_time_controls.py`, `freshness_checks.py`: run the look-ahead and staleness checks behind G2.
- `corporate_actions.py`: keeps raw prices and applies split, rights and dividend factors when data is read. Level filters use the unadjusted price that was available at the time.
- `identifier_mapper.py`, `calendar_manager.py`, `unit_currency_standardiser.py`, `timezone_normaliser.py`: harmonise keys, sessions, units and clocks.
- `data_validator.py`, `data_quality_report.py`: classify each finding as Block, Quarantine or Warn and write the quality summary.

**`stage_00_screening/`**

- `candidate_registry.py`: builds the point-in-time candidate list, including names later delisted, for historical tests.
- `screening_request.py`: reads `configs/screening_request.yaml`, which lists the minimum screening fields.
- `extractors/intraday_coverage_probe.py`: checks or samples intraday availability without downloading full high-frequency history.
- `extractors/source_availability_probe.py`: records only whether an event or news source exists. It never builds features.
- `screening_snapshot.py`, `coverage_inventory.py`, `screening_publisher.py`: write the versioned Stage 0 outputs.

**`stage_01_universe/`**

- `score_standardisation.py`: computes the robust sector-relative score $z_{ij}=(x_{ij}-\operatorname{median}_{s(i)}x_j)/(1.4826\,\operatorname{MAD}_{s(i)}x_j)$.
- `composite_score.py`: computes $C_{ik}=\sum_{j\in k}\omega_{jk}\Phi(d_j z_{ij})$ and $S_i=100\sum_k w_k C_{ik}$. The weights in each category sum to one, and so do the category weights.
- `strategy_classification.py`: routes each name to trend, quality-value, pairs, event or intraday hypotheses.
- `admission_capacity.py`: applies the per-sleeve admission threshold and the research capacity limit.
- `mandatory_removals.py`: applies `EXIT-DIST`, `EXIT-DLST`, `EXIT-SUSP`, `EXIT-TKO` and `EXIT-LIQ` only. Score-based removal, replacement and re-entry are not allowed.

**`stage_02_data_expansion/`**

- `data_request_manifest.py`: freezes universe version, sleeve, cutoff, horizon, sources and tier. It refuses to run without a G1-passed version.
- `context_node_scope.py`: allows linked suppliers, customers, competitors or shared-input entities as context nodes. It records the relationship and scope and never adds them to the universe.
- `harmonisation.py`: runs the as-of, schema, unit, time-zone, corporate-action and freshness checks.
- `missingness_policy.py`: keeps missingness explicit rather than filling it.
- `core_feature_builder.py`: builds $X^{core}_{i,t,h}$ with no NLP-derived variables. Ordinary structured feeds, such as reported earnings or official macro releases, stay in the core set.
- `reference/geographic_exposures.py`: stores dated revenue shares $e^{rev}_{i,c}$ and cost shares $e^{cost}_{i,c}$ separately.
- `reference/relationship_graph.py`: holds the dated entity graph used by Stage 2B transmission and Stage 2D propagation.

**`stage_02_forecasting/`**

- `label_builder.py`: computes the target $y^{(h)}_{i,t}=\sum_{s=2}^{h+1}(r_{i,t+s}-r_{b,t+s})$ and records the label end date used for purging.
- `arm_builder.py`: builds Arm A from $X^{core}$ and Arm B from $X^{core}$ plus as-of $Z^{news}$ read from the feature store.
- `forecast_validator.py`: rejects bad timestamps, schema errors and crossing quantiles.
- `financial/distress_hazard.py`: fits the discrete-time hazard $\Lambda(\alpha(t)+\beta^\top x_{i,t})$.
- `financial/volatility_garch.py`: fits GJR-GARCH $\sigma_t^2=\omega+(\alpha+\gamma\mathbf 1\{r_{t-1}<0\})r_{t-1}^2+\beta\sigma_{t-1}^2$ and EGARCH, both with Student-t innovations and an EWMA baseline.
- `financial/volatility_har_rv.py`: fits HAR-RV on daily, weekly and monthly realised variance, optionally with jumps.
- `financial/dynamic_exposures.py`: estimates filtered Kalman betas only. Smoothed estimates are never used for live signals.
- `macro/company_macro_exposure.py`: computes $f^{rev}_{i,t}=\sum_c e^{rev}_{i,c}f_{c,t}$ and $f^{cost}_{i,t}$ separately.
- `macro/macro_sensitivity.py`: shrinks noisy sensitivities with $\hat b^{post}=\lambda\hat b^{OLS}+(1-\lambda)b^{prior}$, where $\lambda=\tau^2/(\tau^2+\mathrm{se}^2)$.
- `macro/release_surprise.py`: computes $S_{c,t}=(\text{actual}-\text{consensus})/\mathrm{sd}(\text{surprise})$.
- `firm_microstructure/interday/panel_model.py`: fits the two-way panel $r_{it}=\alpha_i+\lambda_t+\beta^\top x_{it}+\epsilon_{it}$ with firm- and date-clustered uncertainty.
- `firm_microstructure/intraday/realised_measures.py`: computes realised variance, bipower variation, jumps, semivariance and realised kernels.

**`stage_02d_news_features/`**

- `source_timing_check.py`, `latency_admissibility.py`: enforce both the publication-time and the first-seen/processing-time cut-offs. A feature generated after the decision time is unavailable even if the article was published earlier.
- `prompt_registry.py`: freezes the NLP model, prompt and schema versions for each evaluation period.
- `llm_event_extractor.py`, `evidence_validator.py`: extract fixed-schema facts with evidence spans and reject invalid structures.
- `event_quantification.py`, `novelty_surprise.py`, `time_decay.py`, `entity_aggregation.py`: build the feature groups listed in Section 7.3.
- `graph_propagation.py`: computes first-order terms proportional to $W_{ij}\delta_j z_j$. It checks that the spectral radius is below one for linear propagation and stops duplicate stories from multiplying the same shock.
- `event_risk_calibration.py`: fits $P(|AR_{i,t,h}|>c\hat\sigma_{i,t}\mid z^{news}_{i,t})=\Lambda(\theta^\top z^{news}_{i,t})$ and validates it with Brier and log loss and reliability curves. The output is a covariate, not a forecast.
- `feature_qc.py`, `news_coverage_report.py`, `news_feature_publisher.py`: publish the versioned $Z^{news}$ with missingness flags. Missing values are never set to zero.

**`stage_02e_validation/`**

- `walk_forward_folds.py`, `purging_embargo.py`: build chronological folds, purge overlapping labels and embargo at least the forecast horizon.
- `experiment_pairing.py`, `coverage_matching.py`: pair forecasts by family, target, horizon and test observation, and report the matched sample and its coverage.
- `arm_parity_check.py`: fails if the arms differ in anything other than the registered news features.
- `forecast_scoring.py`: computes pinball loss, CRPS, coverage, sharpness and PIT.
- `forecast_comparison_tests.py`: runs Diebold–Mariano, Giacomini–White, Hansen SPA and MCS.
- `paired_loss_differences.py`, `block_bootstrap.py`, `multiple_testing.py`, `feature_ablation.py`: produce the incremental-value evidence.
- `overfitting_controls.py`: computes DSR and PBO over every recorded trial.
- `model_eligibility.py`, `incremental_value_test.py`: return the two G3 decisions separately.
- `slot_selection.py`: picks one variant per slot. It never counts a baseline and its challenger as two ensemble members unless that design is registered.
- `signal_combiner.py`: computes $\hat q_\tau=\sum_k w_k\hat q_{k,\tau}$, with $w_k\ge0$ and $\sum_k w_k=1$, shrinking towards equal weights when evidence is weak.
- `conformal_calibration.py`, `horizon_selection.py`: calibrate on held-out data and choose the horizon with the best net information ratio.

**`stages_03_04_portfolio/`**

- `tail_risk_engines.py`, `risk_backtests.py`: compute VaR and ES by filtered historical simulation, Monte Carlo and EVT, and demote any engine that fails its coverage or independence backtest.
- `cross_market_alignment.py`: handles the asynchronous ASX and US sessions using multi-day returns or lead–lag terms.
- `expected_returns.py`: computes the Black–Litterman mean $\mu_{BL}$.
- `portfolio_objective.py`: maximises $\mu_{BL}^\top w-\tfrac{\gamma}{2}w^\top\hat\Sigma w-\sum_i(c_i|\Delta w_i|+\eta_i|\Delta w_i|^{3/2})-\kappa\sum_i p_i^{dist}|w_i|$.
- `relaxation_policy.py`: relaxes only the approved constraints, in order: turnover, then sector, then tracking error. It never relaxes liquidity or tail-risk limits. If the problem is still infeasible, it holds current positions and flags the sleeve.

**`stage_05_sizing/`**

- `kelly_cap.py`: computes $L^{Kelly}_t=\max(0,c\,\hat\mu/u_t)$, where $u_t=\hat\sigma_{p,t}^2+\widehat{\operatorname{Var}}(\hat\mu)$ and $0.25\le c\le 0.5$ is provisional.
- `volatility_cap.py`: computes $L^{vol}_t=\sigma_{target}/\hat\sigma_{p,t}$.
- `leverage_rules.py`: takes the minimum of the caps.
- `drawdown_multiplier.py`: computes $d^{DD}_t=\max(0,1-DD_t/DD_{max})$.

Any non-finite, stale or invalid input blocks new risk. It never increases leverage.

## 5. Gates, reason codes and run status

Gates are implemented as pure checks that return a common gate result (gate ID, outcome, severity, machine-readable reason code, evidence reference). Gate thresholds are read from `configs/gates.yaml` and are provisional until approved per sleeve.

| Gate | Module | Evaluated in | Failure action |
|---|---|---|---|
| G1 Universe | `gates/g1_universe.py` | Stage 01 | Reject at formation or log a safety exit |
| G2 Data integrity | `gates/g2_data_integrity.py` | Stage 00 (screening data) and Stage 02 (detailed data and news timing) | Block (look-ahead, schema, units, time zone, duplicate keys, corrupted timestamps), quarantine (stale history, unexplained extreme return, unreconciled corporate action) or warn (repeated closes, revision bursts, non-critical coverage gaps) |
| G3 Model eligibility | `gates/g3_model_eligibility.py` | Stage 02E | Exclude the failing candidate; retain an eligible baseline if its challenger fails; no eligible candidate means no trade |
| G4 Portfolio risk | `gates/g4_portfolio_risk.py` | Stages 03–04 | Re-optimise within approved relaxations or scale toward cash |
| G5 Leverage and margin | `gates/g5_leverage_margin.py` | Stage 05 | Reduce gross exposure or block new risk |
| G6 Final trade eligibility | `gates/g6_trade_eligibility.py` | Stage 05, re-checked at Stage 06 | Block the order; risk-reducing action only if policy permits |
| G7 Model promotion | `gates/g7_model_promotion.py` | Model lifecycle | Keep the current approved model |

G1–G7 are project-defined labels, not external regulatory identifiers. G3 reports two separate outcomes for a news challenger: **absolute eligibility** and **incremental value over its matched baseline**.

**Reason codes** live in `contracts/reason_codes.py` and `configs/reason_codes.yaml`. They include the universe rejection codes (`DIST`, `LIQ`, `HIST`, `DATA`), the mandatory-exit codes (`EXIT-DIST`, `EXIT-DLST`, `EXIT-SUSP`, `EXIT-TKO`, `EXIT-LIQ`) and the decision outcomes `TRADE`, `REDUCE`, `HOLD` and `NO-TRADE`. News-feature quarantines, contract failures and missing-news exclusions also carry codes. Every failure receives a code.

**Run status** (`contracts/run_status.py`): `PASS` means required checks passed; `DEGRADED` means an approved fallback from `configs/fallbacks.yaml` is active and the degradation is recorded; `BLOCKED` means a critical check failed and no new risk may be initiated. Fallbacks must be tested and can never bypass data integrity, margin, hard exposure limits, kill switches or reconciliation. Status is recorded per sleeve. As a repository convention (to be confirmed), the run's terminal status reports the most severe sleeve status (`BLOCKED` over `DEGRADED` over `PASS`).

## 6. Module dependency and independence rules

The design requires the screening decision to precede deep retrieval, the news experiment to stay isolated, and hard limits, kill switches and reconciliation to stay independent of forecasts, optimisation and monitoring. The repository enforces this structurally, not by convention only.

| Package | May import | Must not import |
|---|---|---|
| `contracts`, `config` | Standard library and third-party libraries | Any other `aqp_python` package |
| `storage`, `gates` | `contracts`, `config` | `stages`, `safety`, `model_lifecycle`, `reporting` |
| `data_foundation` | `contracts`, `config`, `storage` | `stages`, `safety`, `model_lifecycle`, `reporting` |
| `stages/*` (general) | `contracts`, `config`, `storage`, `gates`, `data_foundation` | Other stages' internals; hand-offs go through contract-typed datasets in `storage` |
| `stage_00_screening`, `stage_01_universe` | As above; read-only `connectors/lseg` | `stage_02_data_expansion`, `stage_02_forecasting`, `stage_02d_news_features`, `stage_02e_validation`, `connectors/news`, `connectors/llm` |
| `stage_02d_news_features` | As above; `connectors/news`, `connectors/llm` | `stage_02_forecasting`, `stage_02e_validation`, `stages_03_04_portfolio`, `stage_05_sizing`, `stage_06_execution`, `connectors/broker` |
| `stage_02_forecasting` | As above; reads $Z^{news}$ only through `storage/feature_store.py` | `stage_02d_news_features` internals, `connectors/llm`, `connectors/broker` |
| `safety` | `contracts`, `config`, read-only `storage`, `connectors/broker` interface | `stage_02_forecasting`, `stage_02d_news_features`, `stage_02e_validation`, `stages_03_04_portfolio`, `stage_05_sizing`, `model_lifecycle`, `reporting`, `compiler` |
| `model_lifecycle` | `contracts`, `config`, `storage`, `gates` | `connectors/broker`, `stage_06_execution` (a candidate has no order authority) |
| `reporting`, `compiler` | `contracts`, `config`, read-only `storage` | `connectors/broker`, `stage_06_execution`, `safety` write paths |
| `connectors/broker` | `contracts` | Used only by `stage_06_execution` and `safety` |
| `connectors/llm` | `contracts`, `config` | Used only by `stage_02d_news_features` |

`.importlinter` encodes these contracts; `tests/invariants/test_hard_limits_independence.py` and `test_no_deep_retrieval_before_g1.py` run them in the test suite. The rules are documented in `docs/architecture/module_dependency_rules.md`.

## 7. Data and forecast contracts

### 7.1 Screening and data-request contracts

- `contracts/screening_schema.py` describes the candidate registry, the screening snapshot and the coverage inventory. Every field carries valid time and availability time.
- `contracts/data_request_schema.py` describes the frozen Stage 2 request: universe version, sleeve, decision cutoff, horizon, coverage tier, sources, frequency, depth, freshness and context-node scope. `storage/data_request_store.py` writes it to `data/data_requests/`.

### 7.2 Core feature contract ($X^{core}$)

`contracts/core_feature_schema.py` keys every row by security or entity, decision timestamp, frequency or horizon, units or currency, and availability rule. It also records source lineage and missingness flags. It must not contain variables produced by the Stage 2D NLP/LLM pipeline.

### 7.3 News feature contract ($Z^{news}$)

`contracts/news_feature_schema.py` and `contracts/event_extraction_schema.py` define the published vector and the fixed LLM extraction schema. At minimum, each record has:

- the entity, the decision time and the horizon;
- publication, first-seen and processing timestamps;
- the source and its credibility, plus source spans or IDs;
- versions of the NLP/LLM model, prompt, extraction schema, entity map and relationship graph;
- feature groups: event semantics, surprise and novelty, reliability, time dynamics, exposure and network links, and calibrated event risk;
- confidence, coverage and missingness flags with reason codes.

Missing or low-confidence news is marked missing. It is never zero.

### 7.4 Standard forecast contract

Forecasting models return a documented object, not arbitrary files. The contract is defined in `contracts/forecast_schema.py` and the shared quantile grid in `contracts/quantile_grid.py`. At minimum, record:

- security identifier, market and sleeve;
- experiment ID and arm (`BASELINE` or `NEWS_CHALLENGER`);
- decision timestamp, forecast creation timestamp and information cut-off;
- forecast horizon and target definition (benchmark-relative excess return $y^{(h)}_{i,t}=\sum_{s=2}^{h+1}(r_{i,t+s}-r_{b,t+s})$, starting after the decision point);
- quantile values on the shared grid $\mathcal T=\{0.01, 0.025, 0.05, 0.10, 0.25, 0.50, 0.75, 0.90, 0.95, 0.975, 0.99\}$;
- label end date (used for purging during validation);
- model name and version, feature version (core and, for challengers, news), training window and data lineage;
- validation and eligibility status;
- warnings, missing-input flags and failure reason codes.

`forecast_validator.py` rejects invalid timestamps or schema and crossing quantiles. Forecasts with different targets, horizons or units are not compared or combined until explicitly aligned. Pinball loss is the primary quantile score; interval coverage, sharpness and the probability integral transform are tracked alongside it.

## 8. Paired news experiment

The design does not assume that news adds value. It tests that claim for every **slot**, meaning each combination of model family × target × horizon × sleeve.

| Element | Repository location |
|---|---|
| Registration of both arms before the test results are seen | `configs/experiments.yaml`, `scripts/register_experiment.py`, `storage/experiment_registry.py` |
| Arm A (baseline): $X^{core}$ only | `stage_02_forecasting/arm_builder.py` |
| Arm B (challenger): the same $X^{core}$ plus the versioned $Z^{news}$ | `stage_02_forecasting/arm_builder.py` |
| Held constant: universe version, timestamps, labels, horizons, folds, model family, tuning budget, metrics, downstream rules, execution policy and costs | `stage_02e_validation/arm_parity_check.py` |
| Matched observations; missing-news coverage reported separately | `stage_02e_validation/experiment_pairing.py`, `coverage_matching.py` |
| Paired loss and net-of-cost differences with uncertainty | `paired_loss_differences.py`, `block_bootstrap.py`, `net_of_cost_evaluation.py` |
| Multiple-testing, ablation and trial accounting | `multiple_testing.py`, `feature_ablation.py`, `overfitting_controls.py`, `storage/trial_registry.py` |
| Decision | `incremental_value_test.py`, then `slot_selection.py`. If the result is inconclusive, the eligible baseline or the current approved model is kept. A passing challenger goes to shadow evaluation and G7 |

A news-based overlay downstream must be applied identically to both arms, or it must be registered as a separate experiment. Changing anything else makes it a different experiment. The baseline stays as an explicit control throughout development. It is not deleted because a challenger looks promising in one fold or one metric.

## 9. Sleeves and configuration

| Sleeve | Configuration file | Key concern |
|---|---|---|
| ASX interday | `configs/sleeves/sleeve_asx_interday.yaml` | Closing-auction execution and limited short borrow |
| US interday | `configs/sleeves/sleeve_us_interday.yaml` | FX exposure and overnight gap risk |
| ASX intraday | `configs/sleeves/sleeve_asx_intraday.yaml` | Spread, tick size and available depth |
| US intraday | `configs/sleeves/sleeve_us_intraday.yaml` | Execution costs and competition for short-horizon signals |

Sleeves share infrastructure but keep separate calendars, cost assumptions, data depth and risk budgets. Gate thresholds, target volatility, drawdown limits and position limits are provisional and are calibrated per sleeve. They are never hard-coded. The design's illustrative values (portfolio volatility ≤ 1.1× target, single name ≤ 8%, sector ≤ 30%, exit within three days at 20% of ADV) belong in configuration as provisional entries.

`configs/coverage_tiers.yaml` holds the T1, T2 and T3 data-coverage budgets: about 40 names per market with trades and quotes, about 120 with one-minute bars, and the rest with daily data. These sizes are budget illustrations. A tier controls only the Stage 2 data request and never changes universe membership.

For intraday sleeves, news is used conservatively, for risk and volatility awareness, until a directional edge is shown after costs. `cross_market_alignment.py` handles the asynchronous ASX and US sessions.

## 10. Orchestration and expected run flow

```text
Run configuration, run ID, decision cut-off and sleeve
        ↓
Stage 00: Candidate list and screen-level data; standardise and validate     [G2]
        ↓
Stage 01: Filter, score, classify and freeze the research universe          [G1]
        ↓
Stage 02: Freeze data request → detailed data for admitted names only       [G2]
        ↓                       ↘
   X_core feature store          Raw text corpus → Stage 02D → Z_news
        ↓                       ↙
Stage 02A–2C: Baseline (X_core) and challenger (X_core + Z_news) per slot
        ↓
Common forecast contract → Stage 02E paired validation and eligibility      [G3]
        ↓
Stages 03–04: Risk estimation, optimisation, sleeve allocation              [G4]
        ↓
Stage 05: Leverage, margin, trade decision, order requests                  [G5, G6]
        ↓
Independent safety approval (pre-trade checks, kill switch)
        ↓
Stage 06: Execution simulation or approved broker adapter
        ↓
Stage 07: Monitoring, attribution, news drift/latency, readiness
        ↓
Compile plots, diagnostics, run manifest and narrative report
```

The orchestrator (`orchestration/orchestrator.py`) is the only component that launches the full flow. Stage modules never start the workflow themselves. `stage_router.py` will not dispatch any Stage 2 step until G1 has recorded a universe version for the run.

The orchestrator stops a run, blocks it, or marks it degraded when an essential stage fails. A failed model or news branch may be skipped only if `configs/fallbacks.yaml` defines an approved fallback and the degradation is recorded. The narrative report describes recorded output. It must not invent results, override a gate or issue an order.

## 11. Required run outputs

Each run is written to an immutable folder, `outputs/runs/<run_id>/`:

```text
outputs/runs/<run_id>/
├── run_manifest.json
├── config_snapshot.yaml
├── screening/           # candidate registry version, screening snapshot, coverage inventory, G2 evidence
├── universe/            # universe version, selected_companies.csv, exclusions.csv, exits.csv, coverage tiers
├── data_request/        # frozen data-request manifest and context-node scope
├── data_lineage/        # lineage, data_quality_summary.csv, quarantine list
├── features/            # X_core and Z_news versions, coverage and missingness reports
├── forecasts/           # per experiment ID and arm
├── validation/          # fold definitions, paired metrics, uncertainty, eligibility and incremental-value decisions, trial log
├── portfolio/           # risk estimates, stress results, target weights, constraint relaxations
├── sizing/              # leverage components, order requests, position report
├── execution/           # orders, fills, reconciliation, implementation shortfall
├── safety/              # gate results, incidents, kill-switch events
├── monitoring/          # attribution, calibration, drift, news latency/coverage, alerts
├── plots/
└── reports/
    ├── run_summary.json
    └── run_report.md
```

| Manifest record | Required fields |
|---|---|
| Run and configuration | Start/end timestamps and time zone, code revision, configuration version and hash, market, sleeve, cut-off, calendar, horizons |
| Data lineage | Provider and dataset, extraction and observation timestamps, availability-time rule, row counts, schema version |
| News features | Source and coverage counts, publication and first-seen times, NLP/LLM and prompt versions, feature schema, entity-map and relationship-graph versions, quality flags, missingness reasons |
| Universe | Universe version, eligible names, exits, reason codes |
| Models and validation | Registered experiment ID, `BASELINE`/`NEWS_CHALLENGER` arm, model, feature and NLP/prompt versions, training window, fold/purge/embargo settings, paired metrics, matched sample and coverage, uncertainty, calibration and net-cost results, trial count, promotion decision |
| Portfolio and decision | Forecasts, current and target weights, costs, risk contributions, binding constraints, `TRADE`/`REDUCE`/`HOLD`/`NO-TRADE` with reason codes |
| Execution | Order requests, submissions, fills, timestamps, fees, cancellations and rejections, implementation shortfall |
| Outputs and status | Per-security and per-sleeve plots, diagnostics, warnings, failure reasons, per-sleeve and terminal status |

### Plot and report requirements

- Each model saves traceable plots into the current run's `plots/` folder, named with stable model, arm, instrument or market, and plot-type identifiers.
- `plot_compiler.py` builds an index of produced plots and reports missing expected outputs.
- `experiment_summary.py` tabulates paired results per slot from the validation records only.
- `narrative_report.py` uses `run_summary.json` as its only source of truth. Generated prose separates measured results, model interpretations, limitations and failures.
- If report generation fails, retain the structured run summary and mark the report stage failed. Plot and report failures are logged and never conceal safety failures. A run is not marked successful merely because earlier stages completed.

## 12. Model lifecycle and registry

```text
candidate → G3 eligible → shadow mode → G7 approved production model → retired
```

Monitoring may trigger a challenger review, but a challenger never replaces the production model automatically. A news challenger reaches shadow mode only after passing both absolute eligibility and the paired incremental-value test. In shadow mode it is also checked for live latency, coverage, calibration and operational stability.

Artefacts are stored under `models/` by state. Metadata (versions, training windows, experiment IDs and arms, trial counts, approvals, rollback targets) is held by `storage/model_registry.py`, `experiment_registry.py` and `trial_registry.py`. Promotion requires independent review, a pre-agreed shadow period (the design suggests at least 60 sessions), intact calibration, evidence against the current approved model and a tested rollback path. `model_lifecycle` has no order authority. Attribution findings can inform research, but they change universe membership only through an approved re-formation.

## 13. Test batches and invariants

Run batches in order. Each batch has a README describing its objective, fixtures, command, expected artefacts and pass criteria. Automated tests use synthetic or sanitised data; credentials, proprietary raw data and licensed text are never committed.

| Batch | Roadmap phase | Scope | Example pass condition |
|---|---|---|---|
| `test_batch_01_screening_foundation` | 1 Screening foundation (Stage 0) | Candidate registry, screening data, calendars, identifiers, corporate status, G2 | As-of, schema, timestamp, unit and coverage tests pass; the screen uses only information available at its cutoff |
| `test_batch_02_universe_formation` | 2 Universe formation (Stage 1 / G1) | Filters, reason codes, scoring direction, strategy classification, fixed membership, coverage tiers | Survivorship and look-ahead tests pass; membership is reproducible; no deep data request is issued before G1 |
| `test_batch_03_data_expansion` | 3 Post-filter data expansion (Stage 2) | Data-request manifest, detailed extractors, company exposure graph, lineage, $X^{core}$ | Joins, vintages, freshness, units, missingness and G2 failure paths pass; only admitted names and scoped context nodes receive requests |
| `test_batch_04_baseline_models` | 4 No-news baselines (Stages 2A–2C) | Financial, macro and firm/microstructure baselines under the common contract | Leakage-safe targets, a simple comparator, stable outputs and walk-forward evaluation for each baseline |
| `test_batch_05_news_feature_engine` | 5 AI-NLP feature engine (Stage 2D) | Text ingestion, timing, deduplication, entity resolution, extraction schema, $Z^{news}$, coverage and versions | Availability, duplicate, evidence/schema, latency, missingness and feature-stability tests pass |
| `test_batch_06_paired_evaluation` | 6 Paired challenger evaluation (Stage 2E / G3) | Registered pairs, arm parity, paired losses, uncertainty, multiple testing, eligibility | Arms differ only by registered news features; eligibility and incremental value reported separately; no challenger auto-promotes |
| `test_batch_07_risk_optimisation_sizing` | 7 Risk, optimisation and sizing (Stages 3–5) | Covariance and ES, stress library, optimiser, relaxation, sleeve allocation, leverage and margin | Infeasibility and invalid inputs cannot create tradable weights or raise leverage |
| `test_batch_08_execution_and_controls` | 8 Execution simulation and hard controls (Stage 6) | Fill/cost model, order lifecycle, hard limits, kill switch, reconciliation | Cost/fill stress, duplicate/restart, stale-data, margin, reconciliation and emergency-stop tests pass |
| `test_batch_09_monitoring_shadow_release` | 9 Monitoring, shadow and controlled release (Stage 7 / G7) | Plots, structured reports, drift monitors, paper/shadow runs, approvals, rollback | Reproducible runs; stable calibration, coverage and latency; tested incident response; versioned rollback |

Phase 9's written approval, live limits and incident-response rehearsal are sign-off steps outside the automated suite. A passing batch does not by itself authorise any live allocation.

`tests/invariants/` holds design rules that must hold for every release, whichever batch is being run:

- no look-ahead;
- survivorship safety;
- no deep retrieval before G1;
- news not used for admission;
- fixed universe membership;
- the baseline has no news features;
- the arms differ only by news;
- missing news is not zero-filled;
- no automatic promotion;
- an infeasible optimisation means no trade;
- hard limits are independent;
- status semantics;
- narrative reports are built only from structured outputs.

A passing batch establishes only the behaviour it covers. It does not establish profitability or live-trading readiness.

## 14. Windows and VS Code setup

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

To build or refresh the empty shell from the layout specification (existing files are never overwritten):

```powershell
python scripts\build_repository_shell.py --counts
python scripts\build_repository_shell.py --create ..
```

Intended command interfaces once the CLI exists:

```powershell
python -m aqp_python.cli validate-config --config configs/pipeline.yaml
python -m aqp_python.cli run --config configs/pipeline.yaml --sleeve asx_interday --mode research
python scripts\register_experiment.py --config configs/experiments.yaml
pytest tests/invariants
pytest
```

These commands work only after the package, CLI and dependencies have been implemented.

## 15. Data, credentials and Git hygiene

- Keep LSEG, news, LLM, macro and broker credentials outside source control. Use environment variables or an approved secret store; `.env.example` contains placeholder names only.
- Exclude `data/raw`, `data/vintages`, `data/interim`, `data/processed`, `data/text_corpus`, `data/features`, `models/` artefacts, `outputs/` and `build/` from Git unless a specific small, licensed, sanitised fixture is deliberately approved.
- Retain licensed text according to source terms and access controls; record retention and privacy requirements in `configs/news_features.yaml`.
- Do not commit `.venv/`, caches, logs, large plot collections or run outputs.
- `.gitkeep` makes an otherwise-empty directory appear in Git; it does not make hidden files visible or force ignored files to be tracked.
- Respect provider licences: a successful provider connection does not permit redistribution of downloaded data.
- Pin and record dependency versions; use a lock file if the project adopts a dependency manager that supports one.
- Broker access starts with `connectors/broker/paper_broker.py`. Live broker permissions are enabled only after the controlled-release sign-off.

## 16. Python and R boundary

Python coordinates the pipeline and owns the schemas, data contracts, feature stores, experiment registry, run manifest, gates, portfolio controls, compiler and reporting. R implementations live under `r_models/scripts/` and are called only through `r_models/adapters/r_model_adapter.py`. The adapter's input and output types are defined in `r_models/schemas/r_io_schema.json`, together with timeout behaviour, error handling and package requirements. An R model used in a paired experiment receives its arm's features through the adapter exactly as a Python model would. A failed R model is never silently turned into a successful forecast; it becomes a recorded model failure and follows the fallback rules. Current R candidates are `realised_garch.R` and `dcc_garch.R`.

## 17. Implementation order

| Step | Roadmap phase | Build |
|---|---|---|
| 1 | 1 Screening foundation | Tooling, configuration validation, contracts, reason codes, run status, run manifest; `data_foundation`, vintage store, candidate registry, screening extractors and probes, G2 |
| 2 | 2 Universe formation | Stage 1 filters, scoring, classification, admission capacity, versioning, coverage tiers, mandatory removals (G1) |
| 3 | 3 Post-filter data expansion | Data-request manifest, context-node scope, detailed extractors, harmonisation, Company Reference and Exposure Dataset, $X^{core}$ feature store |
| 4 | 4 No-news baselines | Forecast schema and validator, label builder, Stage 2A–2C baselines with simple comparators; walk-forward folds, purging, embargo, trial registry |
| 5 | 5 AI-NLP feature engine | Stage 2D end to end, publishing $Z^{news}$ to the feature store with coverage and versions |
| 6 | 6 Paired challenger evaluation | Experiment registry, arm builder, Stage 2E paired tests, eligibility, slot selection and combination (G3) |
| 7 | 7 Risk, optimisation and sizing | Covariance, tail risk and backtests, stress library, optimiser, relaxation policy, sleeve allocator (G4); Stage 5 sizing (G5, G6) |
| 8 | 8 Execution simulation and hard controls | Order lifecycle, scheduling, costs, fills, shortfall; independent safety package before any broker adapter |
| 9 | 9 Monitoring, shadow and controlled release | Monitoring, plots, experiment summary, narrative report, paper/shadow runs, promotion review (G7), release and rollback; any live allocation only under written sign-off |

Do not implement a downstream model until the upstream contract and its failure tests pass. A stage is complete only when all of the following are documented in `docs/readiness/stage_completion_checklist.md`:

- its input and output schema;
- its positive and failure tests;
- its point-in-time handling;
- its versioned configuration;
- its logs and reason codes;
- reproducible outputs;
- its baseline and invariants;
- the downstream failure actions.

## 18. Open design decisions that affect the repository

The design lists decisions required before implementation. Until resolved, they are tracked in `docs/decisions/open_decisions.md` and kept out of code.

| Decision | Repository impact |
|---|---|
| Instruments and capital structure (cash equities on margin, CFDs, futures) | `configs/execution.yaml`, `leverage.yaml`, margin and financing modules |
| Per-sleeve target volatility, drawdown, name/sector/gross limits, margin buffer | `configs/sleeves/*`, `risk_limits.yaml` |
| US broker and data provider; licensed intraday coverage and cost | `connectors/broker`, `connectors/lseg`, `data_sources.yaml`, `coverage_tiers.yaml` |
| Licensed news sources, timestamp quality, NLP/LLM and prompt versioning, retention and privacy, minimum coverage, latency budgets, missingness policy, intraday directional use | `connectors/news`, `connectors/llm`, `stage_02d_news_features`, `news_features.yaml`, `experiments.yaml` |
| Long-only versus long-short; borrow, especially ASX | `borrow_margin_extractor.py`, `financing_borrow_costs.py`, sleeve files |
| Universe re-formation: deliberate only or scheduled | `universe_versioning.py`, `universe_rules.yaml` |
| Fixed model families or an open challenger process | `model_lifecycle`, `model_lifecycle.yaml`, `experiments.yaml` |

## 19. Key definitions

- **AQP:** Advanced Quantitative Pipeline.
- **Sleeve:** A separately governed sub-portfolio (ASX interday, US interday, ASX intraday, US intraday) with its own calendar, costs, data depth and risk budget.
- **Screening dataset:** The minimum point-in-time data needed for the universe decision; deliberately smaller than the post-filter research pull.
- **Data-request manifest:** Frozen, reproducible specification of the Stage 2 detailed pull (universe version, sleeve, cutoff, tier, sources).
- **Context node:** A non-admitted supplier, customer, competitor or shared-input entity whose linked data is retrieved only to analyse an admitted company; it never joins the universe.
- **HFD / LFD:** High-frequency data (intraday observations) and low-frequency data (daily or less frequent).
- **Point-in-time data:** Data represented by when it could first have been used. Each fact has a valid time and an availability timestamp; restatements create new vintages.
- **Company Reference and Exposure Dataset:** Project-defined, versioned dataset of dated geographic revenue and cost exposures, classifications, peer groups, the relationship graph and membership history; fully assembled in Stage 2.
- **Research universe:** Fixed, versioned set of companies eligible for analysis; not a list of automatic trades.
- **Coverage tier (T1–T3):** Data-depth allocation that governs the Stage 2 request; it does not change membership.
- **$X^{core}$:** Shared core feature set with no NLP-derived variables.
- **$Z^{news}$:** Versioned, point-in-time quantitative news covariate vector produced by Stage 2D; external relative to $X^{core}$, without any claim of causal exogeneity.
- **Slot:** One model family × target × horizon × sleeve combination; its baseline and challenger are alternative versions of the slot.
- **No-news baseline / news challenger:** Arm A with $X^{core}$ only / Arm B with the same model and $X^{core}$ plus $Z^{news}$.
- **Incremental news value:** Paired out-of-sample change in forecast and economic metrics attributable to adding $Z^{news}$ under the registered experiment.
- **Strategy classification:** Routes an eligible company to relevant modelling hypotheses (trend, quality-value, pairs, event, intraday).
- **Gate (G1–G7):** Project-defined checkpoint with a pass condition and failure action.
- **Forecast eligibility:** Validated status allowing a model's output into the signal-combination stage.
- **Hard limit:** Mandatory risk or operational constraint that no forecast, optimiser or report can override.
- **Run manifest:** Machine-readable record of configuration, data, versions, experiments, statuses and outputs for one run.
- **Champion / challenger:** The current approved model and a candidate under independent evaluation.

---

**Repository rule:** keep this file as the navigation map. Put extended mathematics and model derivations in `docs/methodology/`, executable code in `src/aqp_python/`, tests in `tests/`, and generated run artefacts under `outputs/`. Screening decides what is researched; research models propose forecasts; the news engine supplies covariates only; the validated pipeline proposes target positions; only independent risk and execution controls authorise orders.
