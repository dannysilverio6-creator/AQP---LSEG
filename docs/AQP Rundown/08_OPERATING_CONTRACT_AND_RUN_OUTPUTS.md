# Operating Contract and Run Outputs

> **AQP Architecture Version 14** · [Document index](README.md) · [Overall design](00_AQP_OVERALL_DESIGN.md)


**In scope:** Run manifest, lineage, model evaluation records and terminal statuses.

---

Every run creates a unique `run_id` and records:

| Record | Required fields |
|---|---|
| Run/config | Start/end timestamps, code revision, config version, market, sleeve, cutoff, calendar and horizons |
| Data lineage | Provider/dataset, extraction and observation timestamps, availability-time rule, row counts and schema version |
| News features | Source/coverage counts, publication and first-seen times, NLP/LLM and prompt versions, feature schema, entity-map/relationship-graph version, quality flags and missingness reasons |
| Universe | Universe version, eligible names, exits and reason codes |
| Models/validation | Registered experiment ID, `BASELINE`/`NEWS_CHALLENGER` arm, model/feature and NLP/prompt versions, training window, fold/purge/embargo settings, paired metrics, matched sample/coverage, uncertainty, calibration/net-cost results, trial count and promotion decision |
| Portfolio/decision | Forecasts, current/target weights, costs, risk contributions, binding constraints, `TRADE`/`REDUCE`/`HOLD`/`NO-TRADE` and reason codes |
| Execution | Order requests, submissions/fills, timestamps, fees, cancellations/rejections and implementation shortfall |
| Outputs | Per-security and per-sleeve plots, diagnostics, warnings and narrative report |

Run status uses the `PASS`, `DEGRADED` and `BLOCKED` definitions in [Overall Design — Design Principles and Gates](00_AQP_OVERALL_DESIGN.md#2-design-principles-and-gates). Store the status and
reason codes in every run manifest.

The narrative report must be generated from structured outputs. An AI-generated summary may describe
measured results but must not invent numbers, override a gate or issue an order. Plot/report failures
are logged and never conceal safety failures. Fallbacks must be tested and cannot bypass data integrity,
margin, hard exposure limits, kill switches or reconciliation.

---

---

**Related documents:** [Overall design](00_AQP_OVERALL_DESIGN.md) · [Notation, glossary and open decisions](07_NOTATION_GLOSSARY_AND_OPEN_DECISIONS.md) · [Implementation roadmap and readiness gates](09_IMPLEMENTATION_ROADMAP_AND_READINESS_GATES.md)
