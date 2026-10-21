# Appendix B — Module map and glossary

**Totals:** 196 built, 58 merged into another module, 69 planned (of 323 planned Python modules in layout v3).

Every Python module planned in `PY REPOSITORY OVERVIEW.md` (layout v3), and what this guide did with it.

- **Part NN**: built and tested in that part (the part where its first component appears).
- **merged**: the responsibility exists, inside the module named. Split it out when it grows.
- **planned**: not built yet; the note says where it plugs in. The file exists as an empty placeholder only if the repository shell created it.

Modules this guide added that the layout does not list (19): `connectors/synthetic/synthetic_market.py` (Part 04), `stages/stage_00_screening/service.py` (Part 04), `stages/stage_01_universe/service.py` (Part 05), `stages/stage_01_universe/universe_metrics.py` (Part 05), `stages/stage_02_data_expansion/service.py` (Part 06), `stages/stage_02_forecasting/service.py` (Part 08), `stages/stage_02d_news_features/service.py` (Part 09), `stages/stage_02e_validation/service.py` (Part 10), `stages/stage_05_sizing/service.py` (Part 11), `stages/stage_06_execution/portfolio_state.py` (Part 11), `stages/stage_06_execution/service.py` (Part 12), `stages/stage_07_monitoring/service.py` (Part 12), `stages/stages_03_04_portfolio/service.py` (Part 11), `tests/unit/test_extensions.py` (Part 14), `tests/unit/test_forecast_models.py` (Part 08), `tests/unit/test_forecast_scoring.py` (Part 07), `tests/unit/test_monitoring.py` (Part 12), `tests/unit/test_news_aggregation.py` (Part 09), `tests/unit/test_reporting.py` (Part 12).


## package root

| Planned module | Status | Note |
|---|---|---|
| `src/aqp_python/cli.py` | Part 02 |  |
| `src/aqp_python/logging_config.py` | Part 01 |  |
| `src/aqp_python/version.py` | Part 01 |  |

## config

| Planned module | Status | Note |
|---|---|---|
| `config/config_loader.py` | Part 02 |  |
| `config/config_validation.py` | Part 02 |  |
| `config/sleeve_config.py` | merged | `config/config_loader.py` (`load_pipeline_config(path, sleeve)`) |
| `config/experiment_config.py` | merged | `config/config_validation.py` (experiment checks) and `stage_02_forecasting/arm_builder.py` (`model_slots`) |

## contracts

| Planned module | Status | Note |
|---|---|---|
| `contracts/data_schemas.py` | Part 02 |  |
| `contracts/screening_schema.py` | Part 04 |  |
| `contracts/universe_schema.py` | Part 05 |  |
| `contracts/data_request_schema.py` | Part 06 |  |
| `contracts/core_feature_schema.py` | Part 06 |  |
| `contracts/news_feature_schema.py` | Part 08 |  |
| `contracts/event_extraction_schema.py` | merged | the five-field schema is `prompt_registry.py` (`EVENT_TYPES`) + `evidence_validator.py`; split out when a second extractor needs it |
| `contracts/quantile_grid.py` | Part 02 |  |
| `contracts/forecast_schema.py` | Part 08 |  |
| `contracts/experiment_schema.py` | Part 08 |  |
| `contracts/portfolio_schema.py` | Part 11 |  |
| `contracts/order_schema.py` | Part 02 |  |
| `contracts/position_report_schema.py` | planned | the report is built by `stage_05_sizing/position_report.py`; add a column contract when a second consumer appears |
| `contracts/reason_codes.py` | Part 02 |  |
| `contracts/run_status.py` | Part 02 |  |
| `contracts/run_manifest.py` | Part 03 |  |

## gates

| Planned module | Status | Note |
|---|---|---|
| `gates/gate_result.py` | Part 02 |  |
| `gates/gate_registry.py` | planned | each stage calls its gate directly; add a registry when gates become configurable per sleeve |
| `gates/g1_universe.py` | Part 05 |  |
| `gates/g2_data_integrity.py` | Part 04 |  |
| `gates/g3_model_eligibility.py` | Part 10 |  |
| `gates/g4_portfolio_risk.py` | Part 11 |  |
| `gates/g5_leverage_margin.py` | Part 11 |  |
| `gates/g6_trade_eligibility.py` | Part 11 |  |
| `gates/g7_model_promotion.py` | Part 14 |  |

## data_foundation

| Planned module | Status | Note |
|---|---|---|
| `data_foundation/identifier_mapper.py` | planned | one identifier (`security_id`) in the sample data; needed with a licensed source (RIC ↔ ticker ↔ ISIN) |
| `data_foundation/calendar_manager.py` | Part 04 |  |
| `data_foundation/unit_currency_standardiser.py` | planned | single market and currency; needed for the US sleeve (Part 14.9) |
| `data_foundation/timezone_normaliser.py` | merged | `data_foundation/as_of_join.py` (`decision_timestamps`) and `calendar_manager.py` |
| `data_foundation/corporate_actions.py` | Part 04 |  |
| `data_foundation/as_of_join.py` | Part 04 |  |
| `data_foundation/point_in_time_controls.py` | merged | `storage/vintage_store.py` (`as_of`, `latest_vintage`) and `data_foundation/as_of_join.py` |
| `data_foundation/freshness_checks.py` | merged | `data_foundation/data_validator.py` (stale-data check, G2) and `gates/g6_trade_eligibility.py` (data age) |
| `data_foundation/data_validator.py` | Part 04 |  |
| `data_foundation/data_quality_report.py` | Part 04 |  |

## connectors

| Planned module | Status | Note |
|---|---|---|
| `connectors/connector_base.py` | Part 04 |  |

## connectors/lseg

| Planned module | Status | Note |
|---|---|---|
| `connectors/lseg/lseg_session.py` | planned | licensed source: implement the `TabularSource` protocol of `connectors/connector_base.py` |
| `connectors/lseg/lseg_instruments.py` | planned | as above |
| `connectors/lseg/lseg_fields.py` | planned | as above; field names map to `connector_base.SCHEMAS` |

## connectors/macro

| Planned module | Status | Note |
|---|---|---|
| `connectors/macro/macro_provider.py` | planned | macro vintages (Part 14.3) |

## connectors/news

| Planned module | Status | Note |
|---|---|---|
| `connectors/news/news_provider.py` | planned | open decision 4; must return the `news.csv` columns with `published_ts` and `first_seen_ts` |
| `connectors/news/filings_transcripts_provider.py` | planned | as above, for filings and transcripts |

## connectors/llm

| Planned module | Status | Note |
|---|---|---|
| `connectors/llm/llm_client.py` | Part 14 |  |

## connectors/broker

| Planned module | Status | Note |
|---|---|---|
| `connectors/broker/broker_interface.py` | Part 11 |  |
| `connectors/broker/paper_broker.py` | Part 11 |  |

## orchestration

| Planned module | Status | Note |
|---|---|---|
| `orchestration/orchestrator.py` | Part 13 |  |
| `orchestration/stage_router.py` | merged | `orchestration/orchestrator.py` (`run_pipeline`) |
| `orchestration/sleeve_runner.py` | planned | one sleeve today; loop over `pipeline.yaml → sleeves` when a second is added (Part 14.9) |
| `orchestration/run_context.py` | Part 03 |  |
| `orchestration/stage_status.py` | Part 03 |  |
| `orchestration/fallback_registry.py` | merged | `configs/fallbacks.yaml`, read by `stage_02d_news_features/service.py` (`_fallback`) |
| `orchestration/run_manifest_writer.py` | Part 03 |  |

## storage

| Planned module | Status | Note |
|---|---|---|
| `storage/storage_paths.py` | Part 03 |  |
| `storage/dataset_registry.py` | merged | `run_manifest.json → lineage` (tables, columns, hashes) via `connector_base.CsvSampleSource` |
| `storage/vintage_store.py` | Part 03 |  |
| `storage/screening_store.py` | merged | screening artifacts are written to the run folder by `stage_00_screening/service.py` |
| `storage/universe_store.py` | Part 05 |  |
| `storage/data_request_store.py` | Part 06 |  |
| `storage/text_corpus_store.py` | merged | the corpus is read through the data request (`text_corpus_extractor.py`); the story log is written by Stage 02D |
| `storage/feature_store.py` | Part 06 |  |
| `storage/experiment_registry.py` | Part 10 |  |
| `storage/trial_registry.py` | Part 07 |  |
| `storage/model_registry.py` | merged | `stage_02_forecasting/forecast_registry.py`, `storage/trial_registry.py` and `storage/experiment_registry.py` |
| `storage/run_store.py` | merged | `storage/storage_paths.py` (`RunPaths`) |

## stages/stage_00_screening

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_00_screening/candidate_registry.py` | Part 04 |  |
| `stages/stage_00_screening/screening_request.py` | merged | `configs/screening_request.yaml` read by `extractors/screening_extractor.py` |
| `stages/stage_00_screening/extractors/screening_extractor.py` | Part 04 |  |
| `stages/stage_00_screening/extractors/intraday_coverage_probe.py` | merged | `extractors/source_availability_probe.py` |
| `stages/stage_00_screening/extractors/source_availability_probe.py` | Part 04 |  |
| `stages/stage_00_screening/screening_standardiser.py` | merged | `stage_00_screening/service.py` (split adjustment via `data_foundation/corporate_actions.py`) |
| `stages/stage_00_screening/screening_snapshot.py` | merged | `contracts/screening_schema.py` (`ScreeningData`) |
| `stages/stage_00_screening/coverage_inventory.py` | Part 04 |  |
| `stages/stage_00_screening/screening_publisher.py` | merged | `stage_00_screening/service.py` |

## stages/stage_01_universe

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_01_universe/candidate_universe.py` | merged | `stage_00_screening/candidate_registry.py` |
| `stages/stage_01_universe/corporate_status_filter.py` | merged | `gates/g1_universe.py` |
| `stages/stage_01_universe/distress_screen.py` | merged | `gates/g1_universe.py` |
| `stages/stage_01_universe/sector_specific_metrics.py` | planned | sector-relative quality metrics; plug into `universe_metrics.py` |
| `stages/stage_01_universe/liquidity_screen.py` | merged | `gates/g1_universe.py` |
| `stages/stage_01_universe/data_history_check.py` | merged | `gates/g1_universe.py` |
| `stages/stage_01_universe/stability_tests.py` | planned | score stability across formation dates; add before a scheduled re-formation (open decision 6) |
| `stages/stage_01_universe/strategy_classification.py` | Part 05 |  |
| `stages/stage_01_universe/score_standardisation.py` | merged | `stage_01_universe/composite_score.py` |
| `stages/stage_01_universe/composite_score.py` | Part 05 |  |
| `stages/stage_01_universe/admission_capacity.py` | merged | `stage_01_universe/universe_formation.py` (capacity from `universe_rules.yaml`) |
| `stages/stage_01_universe/universe_formation.py` | Part 05 |  |
| `stages/stage_01_universe/universe_versioning.py` | Part 05 |  |
| `stages/stage_01_universe/coverage_tier_assignment.py` | Part 05 |  |
| `stages/stage_01_universe/mandatory_removals.py` | Part 05 |  |

## stages/stage_02_data_expansion

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_02_data_expansion/data_request_manifest.py` | Part 06 |  |
| `stages/stage_02_data_expansion/context_node_scope.py` | planned | non-admitted context nodes (suppliers, peers) for graph propagation (Part 14.6) |
| `stages/stage_02_data_expansion/extractors/financial_extractor.py` | Part 06 |  |
| `stages/stage_02_data_expansion/extractors/macro_extractor.py` | planned | Part 14.3 |
| `stages/stage_02_data_expansion/extractors/microstructure_extractor.py` | planned | T1/T2 intraday data (Part 14.4); `financial_extractor.py` already reports it unavailable |
| `stages/stage_02_data_expansion/extractors/borrow_margin_extractor.py` | planned | long-short (open decision 5) |
| `stages/stage_02_data_expansion/extractors/text_corpus_extractor.py` | Part 06 |  |
| `stages/stage_02_data_expansion/harmonisation.py` | Part 06 |  |
| `stages/stage_02_data_expansion/hfd_builder.py` | planned | high-frequency dataset from T1/T2 sources (Part 14.4) |
| `stages/stage_02_data_expansion/lfd_builder.py` | merged | `stage_02_data_expansion/harmonisation.py` |
| `stages/stage_02_data_expansion/missingness_policy.py` | Part 06 |  |
| `stages/stage_02_data_expansion/core_feature_builder.py` | Part 06 |  |
| `stages/stage_02_data_expansion/reference/company_reference_builder.py` | Part 06 |  |
| `stages/stage_02_data_expansion/reference/geographic_exposures.py` | merged | `reference/company_reference_builder.py` (revenue shares) |
| `stages/stage_02_data_expansion/reference/classifications.py` | merged | `reference/company_reference_builder.py` (sector) |
| `stages/stage_02_data_expansion/reference/peer_clusters.py` | planned | peer groups for cross-sectional features |
| `stages/stage_02_data_expansion/reference/relationship_graph.py` | planned | the weights `W` for `graph_propagation.py` (Part 14.6) |
| `stages/stage_02_data_expansion/reference/membership_history.py` | planned | index membership history (licensed source) |

## stages/stage_02_forecasting

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_02_forecasting/forecast_runner.py` | Part 08 |  |
| `stages/stage_02_forecasting/forecast_registry.py` | Part 08 |  |
| `stages/stage_02_forecasting/forecast_adapter.py` | merged | `stage_02_forecasting/forecast_runner.py` (`to_contract`) |
| `stages/stage_02_forecasting/arm_builder.py` | Part 08 |  |
| `stages/stage_02_forecasting/label_builder.py` | Part 07 |  |
| `stages/stage_02_forecasting/forecast_validator.py` | Part 08 |  |
| `stages/stage_02_forecasting/financial/financial_features.py` | merged | `stage_02_data_expansion/core_feature_builder.py` (X_core) |
| `stages/stage_02_forecasting/financial/baselines.py` | Part 08 |  |
| `stages/stage_02_forecasting/financial/return_quantiles.py` | Part 08 |  |
| `stages/stage_02_forecasting/financial/distress_hazard.py` | planned | financial layer extension; same registry + experiment route as GARCH (Part 14.2) |
| `stages/stage_02_forecasting/financial/volatility_garch.py` | Part 14 |  |
| `stages/stage_02_forecasting/financial/volatility_har_rv.py` | planned | financial layer extension; same registry + experiment route as GARCH (Part 14.2) |
| `stages/stage_02_forecasting/financial/dynamic_exposures.py` | planned | financial layer extension; same registry + experiment route as GARCH (Part 14.2) |
| `stages/stage_02_forecasting/macro/macro_features.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/vintage_alignment.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/dynamic_factor_model.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/regime_model.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/nowcast_midas.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/nowcast_bvar.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/company_macro_exposure.py` | Part 14 |  |
| `stages/stage_02_forecasting/macro/macro_sensitivity.py` | Part 14 |  |
| `stages/stage_02_forecasting/macro/release_surprise.py` | Part 14 |  |
| `stages/stage_02_forecasting/macro/transmission_local_projections.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/fx_exposure.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/macro/macro_forecast.py` | planned | macro layer (Part 14.3); new X_core features, so a new experiment |
| `stages/stage_02_forecasting/firm_microstructure/micro_forecast.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/interday/panel_features.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/interday/panel_model.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/interday/hierarchical_shrinkage.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/interday/dml_effects.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/interday/peer_earnings_response.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/trade_quote_cleaning.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/trade_intensity.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/order_flow_imbalance.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/price_impact.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/realised_measures.py` | Part 14 |  |
| `stages/stage_02_forecasting/firm_microstructure/intraday/liquidity_state.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |
| `stages/stage_02_forecasting/firm_microstructure/intraday/execution_cost_estimator.py` | planned | firm/microstructure layer (Part 14.4); needs T1/T2 data |

## stages/stage_02d_news_features

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_02d_news_features/text_ingestion.py` | Part 09 |  |
| `stages/stage_02d_news_features/source_timing_check.py` | Part 09 |  |
| `stages/stage_02d_news_features/story_deduplication.py` | Part 09 |  |
| `stages/stage_02d_news_features/entity_linking.py` | Part 09 |  |
| `stages/stage_02d_news_features/prompt_registry.py` | Part 09 |  |
| `stages/stage_02d_news_features/llm_event_extractor.py` | Part 09 |  |
| `stages/stage_02d_news_features/evidence_validator.py` | Part 09 |  |
| `stages/stage_02d_news_features/event_quantification.py` | Part 09 |  |
| `stages/stage_02d_news_features/novelty_surprise.py` | Part 09 |  |
| `stages/stage_02d_news_features/time_decay.py` | Part 09 |  |
| `stages/stage_02d_news_features/entity_aggregation.py` | Part 09 |  |
| `stages/stage_02d_news_features/graph_propagation.py` | Part 14 |  |
| `stages/stage_02d_news_features/event_risk_calibration.py` | Part 14 |  |
| `stages/stage_02d_news_features/latency_admissibility.py` | Part 09 |  |
| `stages/stage_02d_news_features/feature_qc.py` | Part 09 |  |
| `stages/stage_02d_news_features/news_coverage_report.py` | Part 09 |  |
| `stages/stage_02d_news_features/news_feature_publisher.py` | Part 09 |  |

## stages/stage_02e_validation

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_02e_validation/walk_forward_folds.py` | Part 07 |  |
| `stages/stage_02e_validation/purging_embargo.py` | Part 07 |  |
| `stages/stage_02e_validation/experiment_pairing.py` | Part 10 |  |
| `stages/stage_02e_validation/arm_parity_check.py` | Part 10 |  |
| `stages/stage_02e_validation/coverage_matching.py` | Part 10 |  |
| `stages/stage_02e_validation/baseline_comparison.py` | merged | `stage_02e_validation/model_eligibility.py` (`score_models`) |
| `stages/stage_02e_validation/forecast_scoring.py` | Part 07 |  |
| `stages/stage_02e_validation/forecast_calibration.py` | merged | `stage_02e_validation/forecast_scoring.py` (`coverage_90`) |
| `stages/stage_02e_validation/forecast_comparison_tests.py` | Part 07 |  |
| `stages/stage_02e_validation/paired_loss_differences.py` | Part 10 |  |
| `stages/stage_02e_validation/block_bootstrap.py` | Part 10 |  |
| `stages/stage_02e_validation/multiple_testing.py` | Part 10 |  |
| `stages/stage_02e_validation/feature_ablation.py` | Part 10 |  |
| `stages/stage_02e_validation/overfitting_controls.py` | Part 07 |  |
| `stages/stage_02e_validation/stability_checks.py` | planned | validation extension (Part 14.8) |
| `stages/stage_02e_validation/net_of_cost_evaluation.py` | Part 07 |  |
| `stages/stage_02e_validation/model_eligibility.py` | Part 10 |  |
| `stages/stage_02e_validation/incremental_value_test.py` | Part 10 |  |
| `stages/stage_02e_validation/slot_selection.py` | Part 10 |  |
| `stages/stage_02e_validation/signal_combiner.py` | Part 10 |  |
| `stages/stage_02e_validation/conformal_calibration.py` | planned | validation extension (Part 14.8) |
| `stages/stage_02e_validation/horizon_selection.py` | planned | validation extension (Part 14.8) |

## stages/stages_03_04_portfolio

| Planned module | Status | Note |
|---|---|---|
| `stages/stages_03_04_portfolio/covariance_estimator.py` | Part 11 |  |
| `stages/stages_03_04_portfolio/factor_covariance.py` | planned | portfolio extension: DCC via the R adapter (Part 14.7), sleeve allocator (Part 14.9) |
| `stages/stages_03_04_portfolio/dcc_garch_adapter.py` | merged | `r_models/adapters/r_model_adapter.py` (Part 14.7) |
| `stages/stages_03_04_portfolio/regime_covariance.py` | planned | portfolio extension: DCC via the R adapter (Part 14.7), sleeve allocator (Part 14.9) |
| `stages/stages_03_04_portfolio/cross_market_alignment.py` | planned | portfolio extension: DCC via the R adapter (Part 14.7), sleeve allocator (Part 14.9) |
| `stages/stages_03_04_portfolio/tail_risk_engines.py` | merged | `stages_03_04_portfolio/portfolio_risk.py` (historical ES) |
| `stages/stages_03_04_portfolio/risk_backtests.py` | planned | portfolio extension: DCC via the R adapter (Part 14.7), sleeve allocator (Part 14.9) |
| `stages/stages_03_04_portfolio/portfolio_risk.py` | Part 11 |  |
| `stages/stages_03_04_portfolio/stress_library.py` | merged | `configs/sleeves/*.yaml → stress_scenarios` |
| `stages/stages_03_04_portfolio/stress_testing.py` | merged | `stages_03_04_portfolio/portfolio_risk.py` |
| `stages/stages_03_04_portfolio/expected_returns.py` | Part 11 |  |
| `stages/stages_03_04_portfolio/portfolio_objective.py` | merged | `stages_03_04_portfolio/portfolio_optimizer.py` |
| `stages/stages_03_04_portfolio/portfolio_constraints.py` | merged | `stages_03_04_portfolio/portfolio_optimizer.py` and `service.py` (`build_problem`) |
| `stages/stages_03_04_portfolio/portfolio_optimizer.py` | Part 11 |  |
| `stages/stages_03_04_portfolio/relaxation_policy.py` | Part 11 |  |
| `stages/stages_03_04_portfolio/sleeve_allocator.py` | planned | portfolio extension: DCC via the R adapter (Part 14.7), sleeve allocator (Part 14.9) |

## stages/stage_05_sizing

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_05_sizing/kelly_cap.py` | merged | `stage_05_sizing/leverage_rules.py` |
| `stages/stage_05_sizing/volatility_cap.py` | merged | `stage_05_sizing/leverage_rules.py` |
| `stages/stage_05_sizing/margin_capacity.py` | merged | `stage_05_sizing/service.py` and `gates/g5_leverage_margin.py` |
| `stages/stage_05_sizing/drawdown_multiplier.py` | merged | `stage_05_sizing/leverage_rules.py` |
| `stages/stage_05_sizing/leverage_rules.py` | Part 11 |  |
| `stages/stage_05_sizing/overnight_gap_risk.py` | planned | intraday sleeves; add to `stress_scenarios` first |
| `stages/stage_05_sizing/financing_borrow_costs.py` | planned | long-short (open decision 5) |
| `stages/stage_05_sizing/trade_decision.py` | Part 11 |  |
| `stages/stage_05_sizing/no_trade_band.py` | merged | `stage_05_sizing/trade_decision.py` |
| `stages/stage_05_sizing/invalidation_conditions.py` | planned | per-position exit rules beyond the universe exits |
| `stages/stage_05_sizing/order_request_builder.py` | merged | `stage_05_sizing/trade_decision.py` |
| `stages/stage_05_sizing/position_sizer.py` | merged | `stage_05_sizing/service.py` |
| `stages/stage_05_sizing/position_report.py` | Part 11 |  |

## stages/stage_06_execution

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_06_execution/order_intent.py` | merged | `contracts/order_schema.py` |
| `stages/stage_06_execution/execution_scheduler.py` | planned | live execution, after sign-off (Part 14.12) |
| `stages/stage_06_execution/order_router.py` | planned | live broker adapter, after sign-off (Part 14.12) |
| `stages/stage_06_execution/order_lifecycle.py` | planned | live execution: cancel/replace, after sign-off |
| `stages/stage_06_execution/execution_simulator.py` | Part 12 |  |
| `stages/stage_06_execution/cost_slippage_model.py` | merged | `stage_06_execution/execution_simulator.py` |
| `stages/stage_06_execution/fill_reconciliation.py` | Part 12 |  |
| `stages/stage_06_execution/implementation_shortfall.py` | merged | `stage_06_execution/fill_reconciliation.py` |

## stages/stage_07_monitoring

| Planned module | Status | Note |
|---|---|---|
| `stages/stage_07_monitoring/performance_attribution.py` | planned | monitoring extension; follow the news monitors in Part 12 |
| `stages/stage_07_monitoring/performance_monitor.py` | Part 12 |  |
| `stages/stage_07_monitoring/calibration_monitor.py` | Part 12 |  |
| `stages/stage_07_monitoring/data_drift_monitor.py` | planned | monitoring extension; follow the news monitors in Part 12 |
| `stages/stage_07_monitoring/news_feature_drift_monitor.py` | Part 12 |  |
| `stages/stage_07_monitoring/news_latency_coverage_monitor.py` | Part 12 |  |
| `stages/stage_07_monitoring/model_drift_monitor.py` | merged | `stage_07_monitoring/calibration_monitor.py` |
| `stages/stage_07_monitoring/risk_backtest_monitor.py` | planned | monitoring extension; follow the news monitors in Part 12 |
| `stages/stage_07_monitoring/execution_error_monitor.py` | planned | monitoring extension; follow the news monitors in Part 12 |
| `stages/stage_07_monitoring/alert_rules.py` | Part 12 |  |
| `stages/stage_07_monitoring/run_diagnostics.py` | merged | `run_manifest.json` and `run.log` |
| `stages/stage_07_monitoring/readiness_status.py` | merged | `stage_07_monitoring/service.py` (`readiness.json`) |

## model_lifecycle

| Planned module | Status | Note |
|---|---|---|
| `model_lifecycle/candidate_registry.py` | merged | `configs/experiments.yaml` + `storage/experiment_registry.py` |
| `model_lifecycle/challenger_training.py` | planned | promotion tooling (Part 14.11); `lifecycle-no-orders` contract already enforced |
| `model_lifecycle/shadow_runner.py` | planned | promotion tooling (Part 14.11); `lifecycle-no-orders` contract already enforced |
| `model_lifecycle/promotion_review.py` | planned | promotion tooling (Part 14.11); `lifecycle-no-orders` contract already enforced |
| `model_lifecycle/release_manager.py` | planned | promotion tooling (Part 14.11); `lifecycle-no-orders` contract already enforced |
| `model_lifecycle/rollback.py` | planned | promotion tooling (Part 14.11); `lifecycle-no-orders` contract already enforced |

## safety

| Planned module | Status | Note |
|---|---|---|
| `safety/pre_trade_checks.py` | Part 12 |  |
| `safety/hard_limits.py` | Part 12 |  |
| `safety/kill_switch.py` | Part 12 |  |
| `safety/reconciliation_block.py` | Part 12 |  |
| `safety/incident_log.py` | Part 12 |  |
| `safety/restart_approval.py` | merged | the kill-switch file: deleting it after review is the restart approval |

## compiler

| Planned module | Status | Note |
|---|---|---|
| `compiler/plot_compiler.py` | Part 12 |  |
| `compiler/report_compiler.py` | merged | `orchestration/orchestrator.py` (`_compile_outputs`) |
| `compiler/run_artifact_index.py` | merged | `run_manifest.json → artifacts` and `plots/plot_index.csv` |

## reporting

| Planned module | Status | Note |
|---|---|---|
| `reporting/metrics_summary.py` | Part 12 |  |
| `reporting/experiment_summary.py` | Part 12 |  |
| `reporting/narrative_report.py` | Part 12 |  |
| `reporting/report_fact_checks.py` | Part 12 |  |

## r_models/adapters

| Planned module | Status | Note |
|---|---|---|
| `r_models/adapters/r_model_adapter.py` | Part 14 |  |

## scripts

| Planned module | Status | Note |
|---|---|---|
| `scripts/build_repository_shell.py` | Part 01 |  |
| `scripts/run_pipeline.py` | Part 13 |  |
| `scripts/run_sleeve.py` | planned | loop `run_pipeline.py` over sleeves (Part 14.9) |
| `scripts/register_experiment.py` | Part 10 |  |
| `scripts/validate_environment.py` | Part 01 |  |
| `scripts/build_sample_data.py` | Part 13 |  |
| `scripts/replay_run.py` | Part 13 |  |

## tests

| Planned module | Status | Note |
|---|---|---|
| `tests/conftest.py` | Part 02 |  |

## tests/unit

| Planned module | Status | Note |
|---|---|---|
| `tests/unit/test_config_loader.py` | Part 02 |  |
| `tests/unit/test_gates.py` | Part 02 |  |
| `tests/unit/test_reason_codes.py` | Part 02 |  |
| `tests/unit/test_as_of_join.py` | Part 04 |  |
| `tests/unit/test_vintage_store.py` | Part 03 |  |
| `tests/unit/test_data_validator.py` | Part 04 |  |
| `tests/unit/test_candidate_registry.py` | Part 04 |  |
| `tests/unit/test_coverage_inventory.py` | Part 04 |  |
| `tests/unit/test_universe_scoring.py` | Part 05 |  |
| `tests/unit/test_universe_removals.py` | Part 05 |  |
| `tests/unit/test_coverage_tiers.py` | Part 05 |  |
| `tests/unit/test_data_request_manifest.py` | Part 06 |  |
| `tests/unit/test_company_reference.py` | Part 06 |  |
| `tests/unit/test_core_feature_builder.py` | Part 06 |  |
| `tests/unit/test_forecast_schema.py` | Part 08 |  |
| `tests/unit/test_quantile_grid.py` | Part 02 |  |
| `tests/unit/test_arm_builder.py` | Part 08 |  |
| `tests/unit/test_news_timing.py` | Part 09 |  |
| `tests/unit/test_entity_linking.py` | Part 09 |  |
| `tests/unit/test_event_extraction_schema.py` | Part 09 |  |
| `tests/unit/test_graph_propagation.py` | Part 14 |  |
| `tests/unit/test_event_risk_calibration.py` | Part 14 |  |
| `tests/unit/test_walk_forward_folds.py` | Part 07 |  |
| `tests/unit/test_purging_embargo.py` | Part 07 |  |
| `tests/unit/test_paired_loss_differences.py` | Part 10 |  |
| `tests/unit/test_block_bootstrap.py` | Part 10 |  |
| `tests/unit/test_model_eligibility.py` | Part 10 |  |
| `tests/unit/test_slot_selection.py` | Part 10 |  |
| `tests/unit/test_signal_combiner.py` | Part 10 |  |
| `tests/unit/test_covariance.py` | Part 11 |  |
| `tests/unit/test_portfolio_constraints.py` | Part 11 |  |
| `tests/unit/test_kelly_leverage.py` | Part 11 |  |
| `tests/unit/test_no_trade_band.py` | Part 11 |  |
| `tests/unit/test_execution_simulator.py` | Part 12 |  |
| `tests/unit/test_safety_controls.py` | Part 12 |  |
| `tests/unit/test_run_manifest.py` | Part 03 |  |

## tests/integration

| Planned module | Status | Note |
|---|---|---|
| `tests/integration/test_screening_to_universe.py` | merged | `tests/integration/test_stage_handoffs.py` (Stage 00 and 01 checkpoints) |
| `tests/integration/test_universe_to_data_request.py` | merged | `tests/integration/test_stage_handoffs.py` (Stage 2 checkpoint) |
| `tests/integration/test_core_and_news_feature_stores.py` | merged | `tests/integration/test_stage_handoffs.py` (Stage 2 and 02D checkpoints) |
| `tests/integration/test_stage_handoffs.py` | Part 04 |  |
| `tests/integration/test_run_artifacts.py` | Part 13 |  |
| `tests/integration/test_fallback_behaviour.py` | Part 13 |  |
| `tests/integration/test_r_adapter.py` | Part 14 |  |

## tests/invariants

| Planned module | Status | Note |
|---|---|---|
| `tests/invariants/test_no_lookahead.py` | Part 06 |  |
| `tests/invariants/test_survivorship_safety.py` | Part 05 |  |
| `tests/invariants/test_no_deep_retrieval_before_g1.py` | Part 06 |  |
| `tests/invariants/test_news_not_used_for_admission.py` | Part 05 |  |
| `tests/invariants/test_universe_fixed_membership.py` | Part 13 |  |
| `tests/invariants/test_baseline_has_no_news_features.py` | Part 08 |  |
| `tests/invariants/test_arms_differ_only_by_news.py` | Part 10 |  |
| `tests/invariants/test_missing_news_not_zero_filled.py` | Part 09 |  |
| `tests/invariants/test_no_auto_promotion.py` | Part 10 |  |
| `tests/invariants/test_infeasible_means_no_trade.py` | Part 11 |  |
| `tests/invariants/test_hard_limits_independence.py` | Part 13 |  |
| `tests/invariants/test_status_semantics.py` | Part 03 |  |
| `tests/invariants/test_narrative_uses_structured_outputs.py` | Part 12 |  |

## Glossary

| Term | Meaning in this project |
|---|---|
| **AQP** | Advanced Quantitative Pipeline (design v14, layout v3). |
| **Sleeve** | A separately governed sub-portfolio (here ASX interday) with its own calendar, costs, risk budget, experiments, paper book and run folders. |
| **Filter first** | Stage 0 screens on a small extract, G1 fixes the universe, and only then does Stage 2 request detailed data and news for the admitted names. Enforced by an import contract. |
| **Screening extract** | The minimum fields Stage 0 may read (`screening_request.yaml`): no news, no exposures, no intraday data. |
| **Coverage tier (T1/T2/T3)** | Assigned at universe formation; decides which detailed sources the data request may ask for each name. |
| **Data request / `request_id`** | The frozen list of names, tiers, sources, fields and dates Stage 2 retrieves; its id is a hash of its content. |
| **Company Reference and Exposure Dataset** | Sector, distress probability and dated revenue shares for admitted names; built in Stage 2, used by stress tests and macro features. |
| **X_core** | The shared core feature set (momentum, reversal, volatility, quality). Contains **no** news-derived variables. |
| **Z_news** | Versioned news covariates keyed by `(security_id, date)`, published by Stage 02D. Covariates, never forecasts or trade signals. |
| **Missing news** | `NaN`, never zero: where the feed was not watching, "no events" means "unknown". |
| **`published_ts` / `first_seen_ts` / `processed_ts`** | When the source says it published / when our feed delivered it / when we finished processing it. Only `processed_ts` decides when a story may count. |
| **Usable day** | The first trading day whose 17:00 decision instant is at or after a story's `processed_ts`. |
| **Story status** | `OK` or the reason code of the first check a story failed (`NEWS_DUPLICATE`, `NEWS_UNLINKED`, `NEWS_NOT_YET_AVAILABLE`, …). Stories are marked, never dropped. |
| **Extractor / prompt version** | The object that fills the fixed five-field event schema (lexicon or LLM), and the frozen instructions it uses. |
| **Evidence span** | The exact quote from the story that supports an extracted event. An event whose evidence is not in the story is rejected. |
| **Arm** | `BASELINE` (X_core only) or `NEWS_CHALLENGER` (X_core + Z_news) of one model. |
| **Model slot** | One model in one arm, keyed `model@ARM`, e.g. `ridge_location_scale@NEWS_CHALLENGER`. |
| **Experiment** | A registered baseline/challenger pair (`experiments.yaml`), hashed before results are seen. A changed spec needs a new id. |
| **`matched_only`** | The missingness policy: the challenger is evaluated only on rows where Z_news exists, and the arms are compared on the rows both cover. |
| **Matched coverage** | Matched rows ÷ baseline test rows. Below 80%, the challenger cannot pass. |
| **Paired test** | DM test, block-bootstrap interval and net Sharpe change of the challenger against its own baseline, on matched rows. |
| **Block bootstrap** | Resampling whole blocks of consecutive values, so the interval respects serial correlation. |
| **Holm adjustment** | Step-down correction of p-values across all experiments tested in a run. |
| **Ablation** | Refitting the challenger without each news feature (and without the whole block); removing the block must worsen the loss. |
| **Arm parity** | The arms may differ only by the registered news features. Checked on every run. |
| **Challenger decision** | `CHALLENGER_UNAVAILABLE`, `CHALLENGER_INELIGIBLE`, `NO_INCREMENTAL_VALUE` or `SHADOW_CANDIDATE`. |
| **Shadow candidate** | A challenger that passed everything. Recorded and monitored; does **not** trade. |
| **Approved arm** | The arm allowed to trade for an experiment (`approved_arm`). Changes only through a reviewed G7 commit. |
| **Approved fallback** | A degraded path listed in `fallbacks.yaml`. The only one shipped: `news_features_unavailable` → baseline arms only, run `DEGRADED`. |
| **Valid time / availability time** | The period a fact describes / when the system could first have used it (`availability_ts`). |
| **As-of query / vintage** | Return only rows available by the decision time / one published version of a fact; restatements add vintages. |
| **Decision time** | The tz-aware moment a decision is taken (`--as-of` date at 17:00 Sydney). |
| **Label / label end** | The target `Σ_{s=2}^{h+1}(r_i − r_b)` and the date its last return is known. Used for purging. |
| **Purging / embargo** | Removing training rows whose label window overlaps the test period / a buffer after it. |
| **Walk-forward** | Train on the past, test on the next block, move forward. |
| **Pinball loss** | Proper score for quantiles: `max(τe, (τ−1)e)`. The primary forecast score. |
| **Coverage (of an interval)** | Share of outcomes inside the 5–95% interval (target 90%). |
| **Diebold–Mariano (DM)** | Test of whether one forecast's loss is lower than another's beyond chance, with HAC variance. |
| **Deflated Sharpe ratio (DSR)** | Probability that the true Sharpe exceeds what the best of N trials would reach by luck. |
| **Trial registry** | Append-only record of every distinct model configuration evaluated (`outputs/state/trial_registry.jsonl`). |
| **Yardstick** | `zero_excess` and `rolling_mean`: references for skill, never weighted. |
| **Eligible** | Passed absolute G3. Only an eligible, approved slot is used. |
| **Black–Litterman / Ledoit–Wolf** | Prior-plus-views expected returns / shrinkage covariance estimator. |
| **ES (expected shortfall)** | Average loss beyond VaR; here 97.5%, historical simulation, 5-day. |
| **Relaxation policy** | The ordered constraints that may be loosened when the optimiser is infeasible. Never liquidity or tail risk. |
| **Fractional Kelly / no-trade band** | `c·μ/(σ² + Var μ)` as a cap on exposure / ignore weight changes smaller than 0.5%. |
| **Gate (G1–G7)** | A project-defined checkpoint with a pass condition and a failure action. Not regulatory rule numbers. |
| **PASS / DEGRADED / BLOCKED** | Run status: healthy / approved fallback active / critical failure, no new risk. |
| **TRADE / REDUCE / HOLD / NO-TRADE** | Per-security decision. A `PASS` run can be all `NO-TRADE`. |
| **Reason code** | Stable machine-readable string attached to every failure, exclusion, mark and decision. |
| **Quarantine / mandatory exit** | Exclude a security for one run (G2) / the only way a member leaves a universe version (`EXIT-*`). |
| **Hand-off object** | The typed output one stage passes to the next (`ScreeningData`, `UniverseSelection`, `ResearchData`, `NewsFeatures`, `ForecastBundle`, `CombinedForecast`, `PortfolioTarget`, `OrderRequest`). |
| **Run manifest** | `run_manifest.json`: identity, config hash, code revision, lineage, stages, gates, records, artifacts, warnings, status. |
| **Kill switch / reconciliation** | The file `outputs/state/KILL_SWITCH` / expected positions compared with the broker's; a mismatch engages the switch. |
| **Research / paper mode** | Research: the book is never changed. Paper: the paper book is updated and runs must move forward in time. |
| **Component** | One captioned block of a file in this guide ("component 2 of 4"). Type a file's components in order, into the same file. |
| **Spine** | The modules built in Parts 01–13: one sleeve, every stage and control, end to end. |
