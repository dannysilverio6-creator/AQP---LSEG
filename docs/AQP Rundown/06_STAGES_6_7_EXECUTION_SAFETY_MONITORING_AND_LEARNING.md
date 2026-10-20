# Stages 6–7 — Execution, Hard Limits, Monitoring and Learning

> **AQP Architecture Version 14** · [Document index](README.md) · [Overall design](00_AQP_OVERALL_DESIGN.md)


**In scope:** Order controls, reconciliation, monitoring, shadow evaluation and lifecycle.

---

Execution uses approved order requests only. Hard limits are independent of forecast, optimisation and
monitoring models.

```mermaid
%%{init: {"theme": "base", "flowchart": {"nodeSpacing": 22, "rankSpacing": 28, "curve": "linear"}, "themeVariables": {"fontSize": "14px"}}}%%
flowchart TB
    A[Approved order requests from Stage 5] --> B["Refresh market data; validate<br/>security, venue and position"]
    B --> C{Pre-trade controls and kill switch permit order?}
    C -->|No| X[Reject/cancel order; halt new risk if required]
    C -->|Yes| D["Choose schedule using urgency,<br/>liquidity and cost estimates"]
    D --> E["Route order using approved<br/>broker/venue configuration"]
    E --> F["Monitor fills, partial fills,<br/>rejects and timeouts"]
    F --> G{Order completed or validly cancelled?}
    G -->|No| H[Cancel/replace only under approved rules]
    H --> F
    G -->|Yes| I[Reconcile broker fills, cash and positions]
    I --> J{Reconciliation passed?}
    J -->|No| Y[Block further orders and raise operational incident]
    J -->|Yes| K["Calculate fees, slippage and<br/>implementation shortfall"]
    K --> L([Confirmed positions and execution records])
    classDef input fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef process fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef stop fill:#fce8e6,stroke:#b3261e,color:#5f1410;
    class A input;
    class B,D,E,F,H,I,K,L process;
    class C,G,J gate;
    class X,Y stop;
```

## Independent safety controls

Security, exposure, daily-loss, drawdown and market-state checks run independently of the forecast and
optimisation models. A breach blocks new risk and triggers the configured cancellation, risk-reduction
and incident-review policy. These controls cannot be bypassed by a model output.

## Hard limits and monitoring

| Control | Illustrative behaviour |
|---|---|
| Name/gross leverage/participation | Reject orders beyond approved limits |
| Kill switch | Cancel open orders and block new risk on breach, stale data or operational incident |
| Reconciliation | Block further orders if broker fills, cash or positions do not reconcile |
| Drawdown | Reduce risk according to the approved drawdown policy |
| Monitoring | Track P&L, forecast calibration, feature drift, risk backtests, execution error, slippage and data quality |

Illustrative alerts (calibrate per sleeve; these are not universal regulatory thresholds): 90% interval
coverage outside 87–93% for 20 sessions; alpha half-life more than 50% shorter than at admission; VaR
backtest p-value below 0.01; realised execution-cost error above 30%. Alerts trigger the configured
response, not automatic model promotion.

## Monitoring and controlled learning

```mermaid
%%{init: {"theme": "base", "flowchart": {"nodeSpacing": 22, "rankSpacing": 28, "curve": "linear"}, "themeVariables": {"fontSize": "14px"}}}%%
flowchart TB
    A["Run records, forecasts, realised returns,<br/>fills, costs and positions"] --> B["Calculate P&L attribution, calibration,<br/>feature drift, risk backtests and execution error"]
    B --> C[Generate per-security and per-sleeve plots]
    B --> D["Generate narrative from verified<br/>structured metrics"]
    B --> E{Threshold breach or material degradation?}
    E -->|No| F[Continue monitoring; retain current model]
    E -->|Yes| G["Apply approved response:<br/>recalibrate or reduce risk"]
    G --> H["Train and assess candidate model<br/>using walk-forward tests"]
    H --> I["Run candidate model in shadow mode;<br/>no order authority"]
    I --> J{G7 approval and calibration intact?}
    J -->|Yes| K[Versioned promotion with rollback plan]
    J -->|No / unclear| L[Retain current model; document failure]
    F --> M([Auditable run package])
    C --> M
    D --> M
    K --> M
    L --> M
    M --> N["Review attribution and update<br/>research hypotheses"]
    N --> O["Change membership only through<br/>approved universe re-formation"]
    classDef data fill:#eaf2f8,stroke:#477b9e,color:#19364a;
    classDef process fill:#edf7ed,stroke:#54845e,color:#1d4026;
    classDef gate fill:#fff2d9,stroke:#b9862b,color:#593c0c,stroke-width:1.5px;
    classDef safe fill:#f3f4f6,stroke:#6b7280,color:#202938;
    class A data;
    class B,C,D,G,H,I,N,O process;
    class E,J gate;
    class F,K,L,M safe;
```

A model may move through `candidate → G3 eligible → shadow mode → G7 approved production model →
retired`. Promotion requires independent review, a pre-agreed shadow period (e.g. at least 60 sessions),
intact calibration, evidence against the current approved model and a rollback path. A candidate model
never replaces the current approved model automatically.

Performance attribution should separate selection, forecasting, portfolio allocation,
factor/sector/country effects and execution shortfall. Findings can inform later research but must not
alter a fixed universe version without an approved re-formation.

---

## Navigate this documentation

**← Previous:** [Stage 5 — Leverage, Sizing and Position Report](05_STAGE_5_LEVERAGE_SIZING_AND_POSITION_REPORT.md)  
**Next →:** [Notation, Glossary and Open Decisions](07_NOTATION_GLOSSARY_AND_OPEN_DECISIONS.md)  
**Quick links:** [Overall design](00_AQP_OVERALL_DESIGN.md) · [Document index](README.md) · [Glossary and open decisions](07_NOTATION_GLOSSARY_AND_OPEN_DECISIONS.md)
