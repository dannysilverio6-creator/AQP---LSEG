# Appendix B — Module map and glossary

**Totals:** 98 built, 56 merged into another module, 89 planned (of 243).

Every Python module planned in `PY REPOSITORY OVERVIEW.md` §3, and what this guide did with it.

- **Part NN** — built and tested in that part.
- **merged** — the responsibility exists, inside the module named. Split it out when it grows.
- **planned** — not built yet; the note says where it plugs in.

Modules this guide added that the overview does not list: `stages/*/service.py` (one entry point per stage), `stage_00_data/synthetic_market.py`, `stage_01_universe/universe_metrics.py`, `stage_06_execution/portfolio_state.py`, `contracts/portfolio_schema.py` hand-off classes, `contracts/event_schema.py` usage in Part 12, and `tests/unit/test_extensions.py`.


## package root

| Planned module | Status | Note |
|---|---|---|
| `cli.py` | Part 02 | |
| `logging_config.py` | Part 01 | |
| `version.py` | Part 01 | |

## config

| Planned module | Status | Note |
|---|---|---|
| `config/config_loader.py` | Part 02 | |
| `config/config_validation.py` | Part 02 | |
| `config/sleeve_config.py` | merged | folded into `config/config_loader.py` (`load_pipeline_config(path, sleeve)`) |

## contracts

| Planned module | Status | Note |
|---|---|---|
| `contracts/data_schemas.py` | Part 02 | |
| `contracts/quantile_grid.py` | Part 02 | |
| `contracts/forecast_schema.py` | Part 07 | |
| `contracts/event_schema.py` | Part 12 | |
| `contracts/portfolio_schema.py` | Part 09 | |
| `contracts/order_schema.py` | Part 02 | |
| `contracts/position_report_schema.py` | planned | the position report is a table built by `stage_05_sizing/position_report.py`; add a column contract here when a second consumer appears |
| `contracts/reason_codes.py` | Part 02 | |
| `contracts/run_status.py` | Part 02 | |
| `contracts/run_manifest.py` | Part 03 | |

## gates

| Planned module | Status | Note |
|---|---|---|
| `gates/gate_result.py` | Part 02 | |
| `gates/gate_registry.py` | planned | not needed yet: each stage calls its gate directly; add a registry when gates become configurable per sleeve |
| `gates/g1_universe.py` | Part 05 | |
| `gates/g2_data_integrity.py` | Part 04 | |
| `gates/g3_model_eligibility.py` | Part 08 | |
| `gates/g4_portfolio_risk.py` | Part 09 | |
| `gates/g5_leverage_margin.py` | Part 09 | |
| `gates/g6_trade_eligibility.py` | Part 09 | |
| `gates/g7_model_promotion.py` | Part 12 | |

## connectors

| Planned module | Status | Note |
|---|---|---|
| `connectors/connector_base.py` | Part 04 | |
| `connectors/lseg/lseg_session.py` | planned | Part 12 §12.1 pattern: an LSEG `MarketDataSource` returning `RawMarketData`; keep `lseg.data` imports inside this package |
| `connectors/lseg/lseg_instruments.py` | planned | LSEG instrument/RIC lists per sleeve (with `lseg_session.py`) |
| `connectors/lseg/lseg_fields.py` | planned | field-name mapping kept in config; verify names against your LSEG Workspace documentation, never guess |
| `connectors/macro/macro_provider.py` | planned | macro source with official release timestamps and vintages (Part 12 §12.3) |
| `connectors/news/news_provider.py` | planned | news source; publication and processing timestamps (Part 12 §12.5) |
| `connectors/broker/broker_interface.py` | Part 09 | |
| `connectors/broker/paper_broker.py` | Part 09 | |

## orchestration

| Planned module | Status | Note |
|---|---|---|
| `orchestration/orchestrator.py` | Part 11 | |
| `orchestration/stage_router.py` | merged | folded into `orchestrator.run_pipeline` (explicit stage sequence); split out if stages become configurable |
| `orchestration/sleeve_runner.py` | merged | `--sleeve` selects the sleeve file; a multi-sleeve runner loops sleeves then calls the sleeve allocator (Part 12 §12.7) |
| `orchestration/run_context.py` | Part 03 | |
| `orchestration/stage_status.py` | Part 03 | |
| `orchestration/fallback_registry.py` | planned | with `configs/fallbacks.yaml`; first use is the DCC→Ledoit-Wolf fallback (Part 12 §12.8) |
| `orchestration/run_manifest_writer.py` | Part 03 | |

## storage

| Planned module | Status | Note |
|---|---|---|
| `storage/storage_paths.py` | Part 03 | |
| `storage/dataset_registry.py` | planned | lineage (path, rows, SHA-256) is recorded per run in the manifest; a persistent registry is needed once a vintage database replaces CSV files |
| `storage/vintage_store.py` | Part 03 | |
| `storage/universe_store.py` | Part 05 | |
| `storage/trial_registry.py` | Part 06 | |
| `storage/model_registry.py` | planned | with model lifecycle (Part 12 §12.9) |
| `storage/run_store.py` | merged | folded into `storage/storage_paths.py` (`RunPaths`) and `orchestration/run_manifest_writer.py` |

## stages/stage_00_data

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_00_data/extractors/lseg_extractor.py` | planned | with the LSEG connector |
| `stages/stage_00_data/extractors/macro_extractor.py` | planned | with the macro connector |
| `stages/stage_00_data/extractors/news_extractor.py` | planned | with the news connector |
| `stages/stage_00_data/extractors/borrow_margin_extractor.py` | planned | needed for long-short (open decision 5) |
| `stages/stage_00_data/identifier_mapper.py` | planned | needed with real providers (RIC ↔ ticker ↔ internal ID, dated) |
| `stages/stage_00_data/calendar_manager.py` | Part 04 | |
| `stages/stage_00_data/unit_currency_standardiser.py` | planned | needed for the US sleeve (USD→AUD) and per-share units |
| `stages/stage_00_data/corporate_actions.py` | Part 04 | |
| `stages/stage_00_data/hfd_builder.py` | planned | intraday data (tiers T1/T2), Part 12 §12.4 |
| `stages/stage_00_data/lfd_builder.py` | merged | folded into `stage_00_data/service.py` (daily data) |
| `stages/stage_00_data/point_in_time_controls.py` | merged | implemented by `storage/vintage_store.py` + G2 look-ahead check |
| `stages/stage_00_data/data_validator.py` | Part 04 | |
| `stages/stage_00_data/data_quality_report.py` | Part 04 | |
| `stages/stage_00_data/reference/company_reference_builder.py` | merged | the synthetic `reference.csv` plays this role; build from provider data with vintages |
| `stages/stage_00_data/reference/geographic_exposures.py` | planned | revenue shares exist (`revenue_share_*`); add cost shares separately (design §7) |
| `stages/stage_00_data/reference/classifications.py` | planned | sector column exists; add industry/product-market |
| `stages/stage_00_data/reference/peer_clusters.py` | planned | needed by panel/hierarchical models (design §8) |
| `stages/stage_00_data/reference/relationship_graph.py` | planned | produces the `W` matrix used by `news/graph_propagation.py` |
| `stages/stage_00_data/reference/membership_history.py` | planned | listing/delisting dates exist in reference data; add index-membership history with announcement dates |

## stages/stage_01_universe

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_01_universe/candidate_universe.py` | merged | folded into `universe_metrics.py` (`compute_metrics` over all reference rows incl. delisted) |
| `stages/stage_01_universe/corporate_status_filter.py` | merged | folded into `gates/g1_universe.py::check_formation` |
| `stages/stage_01_universe/distress_screen.py` | merged | folded into `gates/g1_universe.py::check_formation` |
| `stages/stage_01_universe/liquidity_screen.py` | merged | folded into `gates/g1_universe.py::check_formation` |
| `stages/stage_01_universe/data_history_check.py` | merged | folded into `gates/g1_universe.py::check_formation` |
| `stages/stage_01_universe/stability_tests.py` | planned | beta instability metric in `universe_metrics.py`; add structural-break tests here |
| `stages/stage_01_universe/strategy_classification.py` | Part 05 | |
| `stages/stage_01_universe/score_standardisation.py` | merged | folded into `composite_score.py::robust_z` |
| `stages/stage_01_universe/composite_score.py` | Part 05 | |
| `stages/stage_01_universe/universe_formation.py` | Part 05 | |
| `stages/stage_01_universe/universe_versioning.py` | merged | folded into `universe_formation.py::universe_id_for` + `storage/universe_store.py` |
| `stages/stage_01_universe/data_tier_assignment.py` | planned | Part 12 §12.4 (tiers fixed within a universe version) |
| `stages/stage_01_universe/mandatory_removals.py` | Part 05 | |

## stages/stage_02_forecasting

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_02_forecasting/forecast_runner.py` | Part 07 | |
| `stages/stage_02_forecasting/forecast_registry.py` | Part 07 | |
| `stages/stage_02_forecasting/forecast_adapter.py` | planned | needed when a model's native output (e.g. R, mean/variance) must be mapped onto the quantile grid |
| `stages/stage_02_forecasting/label_builder.py` | Part 06 | |
| `stages/stage_02_forecasting/forecast_validator.py` | Part 07 | |
| `stages/stage_02_forecasting/financial/financial_features.py` | Part 07 | |
| `stages/stage_02_forecasting/financial/baselines.py` | Part 07 | |
| `stages/stage_02_forecasting/financial/return_quantiles.py` | Part 07 | |
| `stages/stage_02_forecasting/financial/distress_hazard.py` | planned | Part 12 §12.2 |
| `stages/stage_02_forecasting/financial/volatility_garch.py` | Part 12 | |
| `stages/stage_02_forecasting/financial/volatility_har_rv.py` | planned | Part 12 §12.2 (needs intraday data) |
| `stages/stage_02_forecasting/financial/dynamic_exposures.py` | planned | Part 12 §12.2 (filtered Kalman only) |
| `stages/stage_02_forecasting/macro/macro_features.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/vintage_alignment.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/dynamic_factor_model.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/regime_model.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/nowcast_midas.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/nowcast_bvar.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/company_macro_exposure.py` | Part 12 | |
| `stages/stage_02_forecasting/macro/macro_sensitivity.py` | Part 12 | |
| `stages/stage_02_forecasting/macro/release_surprise.py` | Part 12 | |
| `stages/stage_02_forecasting/macro/transmission_local_projections.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/fx_exposure.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/macro/macro_forecast.py` | planned | macro layer, Part 12 §12.3 |
| `stages/stage_02_forecasting/micro_market/micro_forecast.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/interday/panel_features.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/interday/panel_model.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/interday/hierarchical_shrinkage.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/interday/dml_effects.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/interday/peer_earnings_response.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/trade_quote_cleaning.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/trade_intensity.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/order_flow_imbalance.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/price_impact.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/realised_measures.py` | Part 12 | |
| `stages/stage_02_forecasting/micro_market/intraday/liquidity_state.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/micro_market/intraday/execution_cost_estimator.py` | planned | micro layer, Part 12 §12.4 |
| `stages/stage_02_forecasting/news/news_ingestion.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/source_timing_check.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/entity_linking.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/story_deduplication.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/prompt_registry.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/llm_event_extractor.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/event_schema_validator.py` | Part 12 | |
| `stages/stage_02_forecasting/news/novelty_check.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/graph_propagation.py` | Part 12 | |
| `stages/stage_02_forecasting/news/event_response_model.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/news/news_event_forecast.py` | planned | news layer, Part 12 §12.5 |
| `stages/stage_02_forecasting/validation/walk_forward_folds.py` | Part 06 | |
| `stages/stage_02_forecasting/validation/purging_embargo.py` | merged | folded into `walk_forward_folds.py::purge_and_embargo` |
| `stages/stage_02_forecasting/validation/baseline_comparison.py` | merged | folded into `model_eligibility.py::score_models` (best-baseline DM test) |
| `stages/stage_02_forecasting/validation/forecast_scoring.py` | Part 06 | |
| `stages/stage_02_forecasting/validation/forecast_calibration.py` | planned | coverage in `forecast_scoring.py`; add PIT and conformal calibration (Part 12 §12.6) |
| `stages/stage_02_forecasting/validation/forecast_comparison_tests.py` | Part 06 | |
| `stages/stage_02_forecasting/validation/overfitting_controls.py` | Part 06 | |
| `stages/stage_02_forecasting/validation/stability_checks.py` | planned | Part 12 §12.6 (the missing G3 criterion) |
| `stages/stage_02_forecasting/validation/net_of_cost_evaluation.py` | Part 06 | |
| `stages/stage_02_forecasting/validation/model_eligibility.py` | Part 08 | |
| `stages/stage_02_forecasting/validation/signal_combiner.py` | Part 08 | |
| `stages/stage_02_forecasting/validation/horizon_selection.py` | planned | Part 12 §12.6 |

## stages/stages_03_04_portfolio

| Planned module | Status | Note |
|---|---|---|
| `stages/stages_03_04_portfolio/covariance_estimator.py` | Part 09 | |
| `stages/stages_03_04_portfolio/factor_covariance.py` | planned | alternative engine behind `covariance_estimator` interface |
| `stages/stages_03_04_portfolio/dcc_garch_adapter.py` | planned | uses `r_models/adapters/r_model_adapter.py` with a Ledoit-Wolf fallback (Part 12 §12.8) |
| `stages/stages_03_04_portfolio/regime_covariance.py` | planned | needs the macro regime model |
| `stages/stages_03_04_portfolio/cross_market_alignment.py` | planned | needed with the US sleeve (Part 12 §12.7) |
| `stages/stages_03_04_portfolio/tail_risk_engines.py` | planned | historical-simulation ES in `portfolio_risk.py`; add FHS/EVT engines here |
| `stages/stages_03_04_portfolio/risk_backtests.py` | planned | VaR coverage/independence backtests of the risk engines |
| `stages/stages_03_04_portfolio/portfolio_risk.py` | Part 09 | |
| `stages/stages_03_04_portfolio/stress_library.py` | planned | scenarios live in the sleeve YAML; applied by `portfolio_risk.py::stress_losses` |
| `stages/stages_03_04_portfolio/stress_testing.py` | merged | folded into `portfolio_risk.py::stress_losses` |
| `stages/stages_03_04_portfolio/expected_returns.py` | Part 09 | |
| `stages/stages_03_04_portfolio/portfolio_objective.py` | merged | folded into `portfolio_optimizer.py::optimise` (`negative_utility`) |
| `stages/stages_03_04_portfolio/portfolio_constraints.py` | merged | folded into `portfolio_optimizer.py::optimise` (constraints list) |
| `stages/stages_03_04_portfolio/portfolio_optimizer.py` | Part 09 | |
| `stages/stages_03_04_portfolio/relaxation_policy.py` | Part 09 | |
| `stages/stages_03_04_portfolio/sleeve_allocator.py` | planned | needed with a second sleeve (Part 12 §12.7) |

## stages/stage_05_sizing

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_05_sizing/kelly_cap.py` | merged | folded into `leverage_rules.py::compute_leverage` |
| `stages/stage_05_sizing/volatility_cap.py` | merged | folded into `leverage_rules.py::compute_leverage` |
| `stages/stage_05_sizing/margin_capacity.py` | merged | folded into `stage_05_sizing/service.py` (margin capacity and headroom) |
| `stages/stage_05_sizing/drawdown_multiplier.py` | merged | folded into `leverage_rules.py::compute_leverage` |
| `stages/stage_05_sizing/leverage_rules.py` | Part 09 | |
| `stages/stage_05_sizing/overnight_gap_risk.py` | planned | needed for the US sleeve held over Sydney night (design §12) |
| `stages/stage_05_sizing/financing_borrow_costs.py` | planned | needed with margin or long-short (open decisions 1 and 5) |
| `stages/stage_05_sizing/trade_decision.py` | Part 09 | |
| `stages/stage_05_sizing/no_trade_band.py` | merged | folded into `trade_decision.py::decide_trades` |
| `stages/stage_05_sizing/invalidation_conditions.py` | planned | add per-position invalidation rules (forecast reversal, tail breach, distress rise) to the position report |
| `stages/stage_05_sizing/order_request_builder.py` | merged | folded into `trade_decision.py::decide_trades` |
| `stages/stage_05_sizing/position_sizer.py` | merged | folded into `stage_05_sizing/service.py` |
| `stages/stage_05_sizing/position_report.py` | Part 09 | |

## stages/stage_06_execution

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_06_execution/order_intent.py` | merged | `contracts/order_schema.OrderRequest` plays this role |
| `stages/stage_06_execution/execution_scheduler.py` | planned | single-session fills today; add VWAP/closing-auction scheduling |
| `stages/stage_06_execution/order_router.py` | planned | needed with a live broker adapter (phase 8) |
| `stages/stage_06_execution/order_lifecycle.py` | planned | needed with a live broker adapter: submitted/partially filled/cancelled states |
| `stages/stage_06_execution/execution_simulator.py` | Part 10 | |
| `stages/stage_06_execution/cost_slippage_model.py` | merged | folded into `execution_simulator.py` |
| `stages/stage_06_execution/fill_reconciliation.py` | Part 10 | |
| `stages/stage_06_execution/implementation_shortfall.py` | merged | folded into `fill_reconciliation.py::implementation_shortfall` |

## stages/stage_07_monitoring

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_07_monitoring/performance_attribution.py` | planned | selection vs forecast vs allocation vs execution attribution (design §13) |
| `stages/stage_07_monitoring/performance_monitor.py` | Part 10 | |
| `stages/stage_07_monitoring/calibration_monitor.py` | Part 10 | |
| `stages/stage_07_monitoring/data_drift_monitor.py` | planned | feature distribution drift vs training folds |
| `stages/stage_07_monitoring/model_drift_monitor.py` | planned | rolling pinball/coverage per model across runs |
| `stages/stage_07_monitoring/risk_backtest_monitor.py` | planned | VaR exceptions vs realised P&L |
| `stages/stage_07_monitoring/execution_error_monitor.py` | planned | realised shortfall vs modelled cost (alert exists in `alert_rules.py`) |
| `stages/stage_07_monitoring/alert_rules.py` | Part 10 | |
| `stages/stage_07_monitoring/run_diagnostics.py` | planned | per-stage timings are in the manifest |
| `stages/stage_07_monitoring/readiness_status.py` | merged | folded into `stage_07_monitoring/service.py` (`readiness.json`) |

## model_lifecycle

| Planned module | Status | Note |
|---|---|---|
| `model_lifecycle/candidate_registry.py` | planned | Part 12 §12.9 (G7 gate is built) |
| `model_lifecycle/challenger_training.py` | planned | Part 12 §12.9 (G7 gate is built) |
| `model_lifecycle/shadow_runner.py` | planned | Part 12 §12.9 (G7 gate is built) |
| `model_lifecycle/promotion_review.py` | planned | Part 12 §12.9 (G7 gate is built) |
| `model_lifecycle/release_manager.py` | planned | Part 12 §12.9 (G7 gate is built) |
| `model_lifecycle/rollback.py` | planned | Part 12 §12.9 (G7 gate is built) |

## safety

| Planned module | Status | Note |
|---|---|---|
| `safety/pre_trade_checks.py` | Part 10 | |
| `safety/hard_limits.py` | Part 10 | |
| `safety/kill_switch.py` | Part 10 | |
| `safety/reconciliation_block.py` | Part 10 | |
| `safety/incident_log.py` | Part 10 | |
| `safety/restart_approval.py` | planned | restart = human removes `KILL_SWITCH` after reviewing `incidents.jsonl`; automate with a signed approval record |

## compiler

| Planned module | Status | Note |
|---|---|---|
| `compiler/plot_compiler.py` | Part 10 | |
| `compiler/report_compiler.py` | merged | folded into `orchestrator._compile_outputs` |
| `compiler/run_artifact_index.py` | planned | artifacts are indexed in `run_manifest.json → artifacts`; plots in `plot_index.csv` |

## reporting

| Planned module | Status | Note |
|---|---|---|
| `reporting/metrics_summary.py` | Part 10 | |
| `reporting/narrative_report.py` | Part 10 | |
| `reporting/report_fact_checks.py` | Part 10 | |

## Scripts and R boundary

| Planned module | Status | Note |
|---|---|---|
| `r_models/adapters/r_model_adapter.py` | Part 12 | |
| `scripts/run_pipeline.py` | Part 11 | |
| `scripts/run_sleeve.py` | merged | `run_pipeline.py --sleeve <name>` covers it |
| `scripts/validate_environment.py` | Part 01 | |
| `scripts/build_sample_data.py` | Part 11 | |
| `scripts/replay_run.py` | Part 11 | |

## Tests

| Planned module | Status | Note |
|---|---|---|
| `tests/conftest.py` | Part 02 | |
| `tests/unit/test_config_loader.py` | merged | tests/unit/test_config.py |
| `tests/unit/test_gates.py` | merged | gate tests live with their stage: test_data_validator (G2), test_universe (G1), test_combination (G3), test_portfolio_and_sizing (G4-G6), test_extensions (G7) |
| `tests/unit/test_reason_codes.py` | merged | tests/unit/test_contracts.py |
| `tests/unit/test_data_validator.py` | Part 04 | |
| `tests/unit/test_vintage_store.py` | merged | tests/unit/test_storage.py |
| `tests/unit/test_company_reference.py` | merged | planned with the reference builders |
| `tests/unit/test_universe_scoring.py` | merged | tests/unit/test_universe.py |
| `tests/unit/test_universe_removals.py` | merged | tests/unit/test_universe.py |
| `tests/unit/test_forecast_schema.py` | merged | tests/unit/test_forecast_models.py (validator) and test_contracts.py |
| `tests/unit/test_quantile_grid.py` | merged | tests/unit/test_contracts.py |
| `tests/unit/test_walk_forward_folds.py` | merged | tests/unit/test_validation_framework.py |
| `tests/unit/test_purging_embargo.py` | merged | tests/unit/test_validation_framework.py |
| `tests/unit/test_model_eligibility.py` | merged | tests/unit/test_combination.py |
| `tests/unit/test_signal_combiner.py` | merged | tests/unit/test_combination.py |
| `tests/unit/test_covariance.py` | merged | tests/unit/test_portfolio_and_sizing.py (via the stage checkpoint) |
| `tests/unit/test_portfolio_constraints.py` | merged | tests/unit/test_portfolio_and_sizing.py |
| `tests/unit/test_kelly_leverage.py` | merged | tests/unit/test_portfolio_and_sizing.py |
| `tests/unit/test_no_trade_band.py` | merged | tests/unit/test_portfolio_and_sizing.py |
| `tests/unit/test_execution_simulator.py` | merged | tests/unit/test_execution.py |
| `tests/unit/test_safety_controls.py` | merged | tests/unit/test_safety_and_reporting.py |
| `tests/unit/test_run_manifest.py` | merged | tests/unit/test_storage.py |
| `tests/integration/test_stage_handoffs.py` | Part 04 | |
| `tests/integration/test_run_artifacts.py` | merged | tests/integration/test_end_to_end.py |
| `tests/integration/test_fallback_behaviour.py` | merged | planned with the first fallback (Part 12 §12.8) |
| `tests/integration/test_r_adapter.py` | merged | tests/unit/test_extensions.py |
| `tests/invariants/test_no_lookahead.py` | Part 07 | |
| `tests/invariants/test_survivorship_safety.py` | merged | covered by test_universe_fixed_membership (SYN05) and the survivorship-safe research panel |
| `tests/invariants/test_universe_fixed_membership.py` | Part 11 | |
| `tests/invariants/test_infeasible_means_no_trade.py` | Part 09 | |
| `tests/invariants/test_hard_limits_independence.py` | Part 11 | |
| `tests/invariants/test_status_semantics.py` | Part 03 | |
| `tests/invariants/test_narrative_uses_structured_outputs.py` | Part 10 | |

---

## Glossary

| Term | Meaning in this project |
|---|---|
| **AQP** | Advanced Quantitative Pipeline. |
| **Sleeve** | A separately governed sub-portfolio (ASX interday, US interday, ASX intraday, US intraday) with its own calendar, costs, data depth, risk budget, paper book and run folders. |
| **Forecast layer / channel** | A family of forecasting models (financial, macro, micro, news). Every layer emits the same forecast contract. |
| **Valid time / availability time** | The period a fact describes / when the system could first have used it (`availability_ts`). |
| **As-of query** | Return only rows with `availability_ts ≤ decision time` (`storage/vintage_store.py`). |
| **Vintage** | One published version of a fact. Restatements add vintages; nothing is overwritten. |
| **Decision time** | The tz-aware moment a decision is taken (`--as-of` date at 17:00 Sydney). |
| **Label / label end** | The target `Σ_{s=2}^{h+1}(r_i − r_b)` and the date its last return is known. Used for purging. |
| **Purging / embargo** | Removing training rows whose label window overlaps the test period / a buffer after it. |
| **Walk-forward** | Train on the past, test on the next block, move forward. |
| **Pinball loss** | Proper score for quantiles: `max(τe, (τ−1)e)`. The primary forecast score. |
| **Coverage** | Share of outcomes inside the 5–95% interval (target 90%). |
| **Diebold–Mariano (DM)** | Test of whether one forecast's loss is lower than another's, beyond chance; HAC variance. |
| **Deflated Sharpe ratio (DSR)** | Probability that the true Sharpe exceeds what the best of N trials would reach by luck. |
| **Trial registry** | Append-only record of every distinct model configuration evaluated (`outputs/state/trial_registry.jsonl`). |
| **Baseline** | A simple yardstick model (zero excess, rolling mean). Never weighted. |
| **Eligible** | Passed G3. Only eligible models receive combination weight. |
| **Black–Litterman** | Blends an equilibrium prior with forecast views weighted by confidence. |
| **Ledoit–Wolf** | Shrinkage covariance estimator. |
| **ES (expected shortfall)** | Average loss beyond VaR; here 97.5%, historical simulation, 5-day. |
| **Relaxation policy** | The ordered list of constraints that may be loosened when the optimiser is infeasible. Never liquidity or tail risk. |
| **Fractional Kelly** | `c·μ/(σ² + Var μ)`, used only as a conservative cap on exposure. |
| **No-trade band** | Ignore weight changes smaller than this (0.5%). |
| **Gate (G1–G7)** | A project-defined checkpoint with a pass condition and failure action. They are not regulatory rule numbers. |
| **PASS / DEGRADED / BLOCKED** | Run status: healthy / approved fallback active / critical failure, no new risk. |
| **TRADE / REDUCE / HOLD / NO-TRADE** | Per-security decision. A `PASS` run can be all `NO-TRADE`. |
| **Reason code** | Stable machine-readable string attached to every failure, exclusion and decision. |
| **Quarantine** | Exclude one security for one run (G2), keeping an audit trail. Not a universe exit. |
| **Mandatory removal / exit** | The only way a member leaves a universe version (`EXIT-*`). No re-entry within the version. |
| **Universe version** | Frozen membership formed once at a formation date (`ASX_20250630_v1`). |
| **Hand-off object** | The typed output one stage passes to the next (`MarketData`, `UniverseSelection`, `ForecastBundle`, `CombinedForecast`, `PortfolioTarget`, `OrderRequest`). |
| **Run manifest** | `run_manifest.json`: identity, config hash, code revision, lineage, stages, gates, artifacts, warnings, status. |
| **Kill switch** | The file `outputs/state/KILL_SWITCH`. If it exists, no orders are sent. |
| **Reconciliation** | Expected positions (book + our fills) compared with the broker's positions. A mismatch blocks trading. |
| **Implementation shortfall** | Slippage against the decision price plus fees. |
| **Research / paper mode** | Research: the book is never changed. Paper: the paper book is updated, and runs must move forward in time. |
| **Champion / challenger** | The current approved model / a candidate under evaluation (G7). |
| **Spine** | The modules built in Parts 01–11: one sleeve, every stage and control, end to end. |
