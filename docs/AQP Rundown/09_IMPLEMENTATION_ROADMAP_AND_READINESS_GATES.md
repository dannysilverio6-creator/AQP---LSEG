# Implementation Roadmap and Readiness Gates

> **AQP Architecture Version 14** · [Document index](README.md) · [Overall design](00_AQP_OVERALL_DESIGN.md)


**In scope:** Implementation order and acceptance criteria for each build phase.

---

Build the system in the same order it runs. Do not implement a downstream model until the upstream contract
and its failure tests pass.

| Phase | Deliverable | Exit criterion |
|---|---|---|
| 1. Screening foundation (Stage 0) | Candidate/ticker registry, point-in-time screening data, calendars, identifiers, corporate status and G2 checks | As-of, schema, timestamp, units and coverage tests pass; the screen uses only information available at its cutoff |
| 2. Universe formation (Stage 1 / G1) | Screening filters, rejection reason codes, strategy classification, fixed universe version and coverage tiers | Survivorship/look-ahead tests pass; membership is reproducible and deep data requests are not issued before G1 |
| 3. Post-filter data expansion (Stage 2) | Run-specific data manifest, detailed financial/macro/firm inputs, company exposure graph, source lineage and core feature store $X^{core}$ | Source joins, vintages, freshness, units, missingness and G2 failure paths pass; only admitted names receive full analysis-data requests |
| 4. No-news baseline models (Stages 2A–2C) | Financial, macro and firm/microstructure baselines plus the common forecast contract | Each baseline has leakage-safe targets, a simple comparator, stable outputs and walk-forward evaluation |
| 5. AI-NLP feature engine (Stage 2D) | Timestamped text ingestion, entity resolution, fixed extraction schema, quantitative $Z^{news}$ features, coverage and version lineage | Text availability, duplicates, evidence/schema, processing latency, missingness and feature stability tests pass |
| 6. Paired challenger evaluation (Stage 2E / G3) | Matched news-augmented challengers, registered experiment pairs, paired loss/economic results, uncertainty and multiple-testing controls | Baseline/challenger differ only by registered NLP-derived features; model eligibility and incremental value are reported separately; no challenger auto-promotes |
| 7. Risk, optimisation and sizing (Stages 3–5) | Covariance/expected shortfall, stress library, cost-aware optimiser, sleeve allocation, leverage and margin checks | Constraint and infeasibility tests pass; invalid inputs or infeasible portfolios cannot create tradable weights |
| 8. Execution simulation and hard controls (Stage 6) | Fill/cost model, order lifecycle, independent limits, kill switch and reconciliation | Cost/fill stress tests, duplicate/restart, stale-data, margin, broker reconciliation and emergency-stop tests pass |
| 9. Monitoring, shadow and controlled release (Stage 7 / G7) | Per-security/sleeve plots, structured reports, drift monitoring, paper/shadow operation, approvals and rollback | Reproducible runs, stable calibration/coverage/latency, tested incident response, written approval and versioned rollback are demonstrated |

A stage is complete only when its contracts, positive and failure tests, point-in-time controls, versioned
configuration, reason codes, reproducible outputs and downstream failure actions are documented. The
baseline remains an explicit control throughout development; it must not be deleted just because a challenger
shows promise in one fold or one metric.

---

## Navigate this documentation

**← Previous:** [Operating Contract and Run Outputs](08_OPERATING_CONTRACT_AND_RUN_OUTPUTS.md)  
**Next →:** [Selected Technical References](10_SELECTED_TECHNICAL_REFERENCES.md)  
**Quick links:** [Overall design](00_AQP_OVERALL_DESIGN.md) · [Document index](README.md) · [Glossary and open decisions](07_NOTATION_GLOSSARY_AND_OPEN_DECISIONS.md)
