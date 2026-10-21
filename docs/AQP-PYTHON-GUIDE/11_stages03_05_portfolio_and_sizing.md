# Part 11 — Stages 03 to 05: portfolio construction, risk gates and sizing

**Goal:** turn the combined forecast distribution into target weights that respect every constraint, check the result's tail risk (G4), decide how much total exposure to take (G5), and turn the difference between the current book and the target into a list of orders, each checked on its own (G6). Every security ends with a decision and a reason code, including the ones that do not trade.

**Design:** Stages 3–5 (*"Stage 3 ranks and constructs; Stage 4 checks; Stage 5 sizes"*), G4–G6. **Roadmap:** phase 7, portfolio and sizing. **Overview:** §8.

This part is almost unchanged from the v12 guide. What changed in v14 is the input: Stage 03 receives only `CombinedForecast` (Part 10), whose `used_slots` say which arm produced it, and the company reference built for admitted names in Part 06. News never reaches this stage except through an approved arm's forecasts.

```mermaid
flowchart LR
    CF[(CombinedForecast)] --> BL[Black-Litterman<br/>expected returns]
    P[(prices)] --> COV[Ledoit-Wolf<br/>covariance]
    COV --> BL
    BL --> OPT[cost-aware optimiser<br/>+ approved relaxations]
    OPT -->|infeasible| HOLD([hold: no new risk])
    OPT --> RISK[ES and stress]
    RISK --> G4{G4}
    G4 -->|breach| SCALE[scale toward cash] --> G4
    G4 --> LEV[leverage:<br/>min of caps × drawdown]
    LEV --> G5{G5}
    G5 --> DEC[per-name decisions:<br/>band, cost, G6]
    DEC --> ORD[(order requests +<br/>position report)]
    BROKER[(paper book)] --> STATE[PortfolioState] --> OPT & LEV & DEC
```

## What you will build in this part

| Section | File | Components |
|---|---|---|
| 11.1 | `contracts/portfolio_schema.py`, `connectors/broker/broker_interface.py`, `connectors/broker/paper_broker.py`, `stage_06_execution/portfolio_state.py` | 1 + 1 + 2 + 1 |
| 11.2 | `stages_03_04_portfolio/covariance_estimator.py`, `expected_returns.py` | 1 + 1 |
| 11.3 | `portfolio_optimizer.py`, `relaxation_policy.py` | 3 + 1 |
| 11.4 | `portfolio_risk.py`, `gates/g4_portfolio_risk.py` | 2 + 1 |
| 11.5 | `stages_03_04_portfolio/service.py` | 6 |
| 11.6 | `stage_05_sizing/leverage_rules.py`, `gates/g5_leverage_margin.py` | 2 + 1 |
| 11.7 | `gates/g6_trade_eligibility.py`, `trade_decision.py`, `position_report.py` | 1 + 2 + 1 |
| 11.8 | `stage_05_sizing/service.py` | 3 |
| 11.9 | tests, gate tests and an invariant | |

---

## 11.1 The book: what we hold, and where it is kept

**File:** `src/aqp_python/contracts/portfolio_schema.py`

```python
"""Hand-off objects for the book state and the portfolio target."""
from dataclasses import dataclass, field

import pandas as pd


@dataclass(frozen=True)
class PortfolioState:
    """Current holdings, read from the broker by Stage 06 and passed forward."""
    positions: pd.Series          # shares by security_id
    cash: float
    prices: pd.Series             # latest validated close by security_id
    equity_history: pd.Series     # NAV by date (for drawdown)

    @property
    def nav(self) -> float:
        held = self.positions.reindex(self.prices.index).fillna(0.0)
        return float(self.cash + (held * self.prices).sum())

    def weights(self) -> pd.Series:
        value = self.positions * self.prices.reindex(self.positions.index)
        return (value / self.nav).fillna(0.0)


@dataclass(frozen=True)
class PortfolioTarget:
    """Stages 03-04 output: relative target weights plus the risk evidence behind them."""
    weights: pd.Series            # relative weights before leverage (sum <= 1)
    current: pd.Series            # current weights on the same index
    mu: pd.Series                 # posterior expected h-day excess return
    mu_covariance: pd.DataFrame   # posterior uncertainty of mu
    covariance: pd.DataFrame      # h-day return covariance
    feasible: bool
    tradable: bool                # False -> Stage 05 may not create new risk
    relaxations: list[str] = field(default_factory=list)
    risk: dict = field(default_factory=dict)
```

**What it does.** Two hand-off objects. `PortfolioState` is the current book: shares, cash, latest prices and the NAV history used for drawdown; `nav` and `weights()` are computed from them, never stored. `PortfolioTarget` is the Stages 03–04 output: relative weights (summing to at most 1, before leverage), the expected returns and their uncertainty, the covariance, and two flags. **`tradable=False` means Stage 05 may not add risk**, whatever the weights say.

**File:** `src/aqp_python/connectors/broker/broker_interface.py`

```python
"""What any broker adapter must provide. Live adapters are added only after sign-off."""
from typing import Protocol

import pandas as pd


class Broker(Protocol):
    def positions(self) -> pd.Series: ...
    def cash(self) -> float: ...
    def apply_fill(self, security_id: str, signed_quantity: int, price: float, fees: float) -> None: ...
    def record_equity(self, date: str, nav: float) -> None: ...
    def equity_history(self) -> pd.Series: ...
    def save(self) -> None: ...
```

**What it does.** The `Protocol` any broker must satisfy. The pipeline only ever talks to this interface, so a live adapter can replace the paper broker without changing a stage, and only after the sign-off in Part 14.

**File:** `src/aqp_python/connectors/broker/paper_broker.py` — component 1 of 2: PaperBroker: open the book

```python
"""A paper broker that keeps its book in a JSON file. No real orders are ever sent."""
import json
from pathlib import Path

import pandas as pd


class PaperBroker:
    def __init__(self, path: Path, initial_cash: float) -> None:
        self.path = Path(path)
        if self.path.is_file():
            book = json.loads(self.path.read_text(encoding="utf-8"))
        else:
            book = {"cash": float(initial_cash), "positions": {}, "equity": {}}
        self._book = book

    def positions(self) -> pd.Series:
        return pd.Series(self._book["positions"], dtype=float)

    def cash(self) -> float:
        return float(self._book["cash"])
```

**File:** `src/aqp_python/connectors/broker/paper_broker.py` — component 2 of 2: apply_fill, equity history and atomic save

```python
    def apply_fill(self, security_id: str, signed_quantity: int, price: float, fees: float) -> None:
        held = self._book["positions"].get(security_id, 0) + int(signed_quantity)
        if held < 0:
            raise ValueError(f"fill would create a short position in {security_id}")
        if held == 0:
            self._book["positions"].pop(security_id, None)
        else:
            self._book["positions"][security_id] = held
        self._book["cash"] -= signed_quantity * price + fees

    def record_equity(self, date: str, nav: float) -> None:
        self._book["equity"][date] = float(nav)

    def equity_history(self) -> pd.Series:
        series = pd.Series(self._book["equity"], dtype=float)
        series.index = pd.to_datetime(series.index)
        return series.sort_index()

    def save(self) -> None:
        self.path.parent.mkdir(parents=True, exist_ok=True)
        temporary = self.path.with_suffix(".tmp")
        temporary.write_text(json.dumps(self._book, indent=2), encoding="utf-8")
        temporary.replace(self.path)
```

**What it does.** Keeps the book in a JSON file. `apply_fill` refuses any fill that would create a short position (the sleeve is long-only, open decision 5). `save` writes a temporary file and then renames it over the old one. **Why:** a rename is atomic on Windows and Linux. If the computer stops half-way through writing, you keep yesterday's complete book rather than half of today's.

```python
import tempfile
from pathlib import Path

from aqp_python.connectors.broker.paper_broker import PaperBroker

book = Path(tempfile.mkdtemp()) / "paper_book.json"
broker = PaperBroker(book, initial_cash=1_000_000)
broker.apply_fill("SYN01", 2_000, 25.00, fees=7.50)          # buy 2,000 at $25 with $7.50 fees
broker.record_equity("2026-10-16", 1_000_000 - 7.50)
broker.save()

reopened = PaperBroker(book, initial_cash=0)                   # the file wins over initial_cash
print("positions:", reopened.positions().to_dict(), "| cash:", reopened.cash())
try:
    reopened.apply_fill("SYN01", -3_000, 25.00, fees=0.0)
except ValueError as error:
    print("refused:", error)
```

```text
positions: {'SYN01': 2000.0} | cash: 949992.5
refused: fill would create a short position in SYN01
```

**File:** `src/aqp_python/stages/stage_06_execution/portfolio_state.py`

```python
"""The read side of the book: Stage 06 owns the broker, and lends its state to Stages 03-05."""
from pathlib import Path

import pandas as pd

from aqp_python.connectors.broker.paper_broker import PaperBroker
from aqp_python.contracts.data_schemas import ResearchData
from aqp_python.contracts.portfolio_schema import PortfolioState
from aqp_python.orchestration.run_context import RunContext


def state_dir(ctx: RunContext) -> Path:
    return ctx.resolve(ctx.config["pipeline"]["state_dir"])


def open_broker(ctx: RunContext) -> PaperBroker:
    path = state_dir(ctx) / ctx.sleeve / "paper_book.json"
    return PaperBroker(path, ctx.config["sleeve"]["initial_nav"])


def latest_prices(data: ResearchData) -> pd.Series:
    """Last validated close per security (held names that stopped trading keep their last price)."""
    last = data.prices.sort_values("date").groupby("security_id").tail(1)
    return last.set_index("security_id")["close"]


def load_portfolio_state(ctx: RunContext, data: ResearchData) -> PortfolioState:
    broker = open_broker(ctx)
    return PortfolioState(broker.positions(), broker.cash(), latest_prices(data), broker.equity_history())
```

**What it does.** Stage 06 owns the broker (it is the only stage that changes the book); this module lends a read-only `PortfolioState` to Stages 03–05. The book lives in `outputs\state\asx_interday\paper_book.json`, outside any run folder, because it carries over from one day to the next.

## 11.2 Risk and expected returns

**File:** `src/aqp_python/stages/stages_03_04_portfolio/covariance_estimator.py`

```python
"""Ledoit-Wolf shrinkage covariance, scaled to the forecast horizon."""
import numpy as np
import pandas as pd
from sklearn.covariance import LedoitWolf


def daily_returns_window(prices: pd.DataFrame, ids: list[str], end: pd.Timestamp,
                         lookback: int) -> pd.DataFrame:
    wide = prices.pivot(index="date", columns="security_id", values="return")
    wide = wide.loc[wide.index <= end, ids].tail(lookback)
    return wide.dropna(how="any")


def ledoit_wolf_covariance(returns: pd.DataFrame, horizon: int) -> pd.DataFrame:
    """Sigma_h = h * Sigma_daily (i.i.d. scaling; a conservative engine can replace it later)."""
    if len(returns) < 60:
        raise ValueError(f"only {len(returns)} complete return rows; need at least 60")
    estimate = LedoitWolf().fit(returns.to_numpy()).covariance_ * horizon
    if not np.all(np.isfinite(estimate)):
        raise ValueError("covariance estimate is not finite")
    return pd.DataFrame(estimate, index=returns.columns, columns=returns.columns)
```

**What it does.** Takes the last `covariance_lookback_days` of complete daily returns and fits a **Ledoit–Wolf** covariance: the sample covariance shrunk towards a simple structured target by the amount that minimises expected estimation error. Then it scales to the horizon, `Σ_h = h · Σ_daily`. It refuses fewer than 60 rows. **Why shrink:** with 23 names and a year of data, the sample covariance has 276 entries estimated from noisy data; an optimiser exploits its errors, putting large weights on pairs that merely *look* uncorrelated.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/expected_returns.py`

```python
"""Black-Litterman: blend equilibrium returns with confidence-weighted forecast views.

mu_BL = [(tau Sigma)^-1 + P' Omega^-1 P]^-1 [(tau Sigma)^-1 pi + P' Omega^-1 q]

pi     equilibrium (CAPM) return: beta_i x market risk premium over the horizon
P = I  one view per security
q      pi + forecast median (the forecast is an excess return over the benchmark)
Omega  tau x diag(s^2), s = IQR / 1.349 - a wider predictive distribution means
       a less confident view
"""
import numpy as np
import pandas as pd

from aqp_python.contracts.quantile_grid import MEDIAN_COLUMN


def equilibrium_returns(beta: pd.Series, premium_annual: float, horizon: int) -> pd.Series:
    return (beta * premium_annual * horizon / 252).rename("pi")


def black_litterman(forecasts: pd.DataFrame, covariance: pd.DataFrame, prior: pd.Series,
                    tau: float) -> tuple[pd.Series, pd.DataFrame]:
    ids = list(covariance.index)
    view = forecasts.set_index("security_id").loc[ids]
    pi = prior.reindex(ids).to_numpy()
    q = pi + view[MEDIAN_COLUMN].to_numpy()
    spread = ((view["q0.750"] - view["q0.250"]) / 1.349).to_numpy()
    prior_precision = np.linalg.inv(tau * covariance.to_numpy())
    view_precision = np.diag(1.0 / (tau * spread ** 2))
    posterior_cov = np.linalg.inv(prior_precision + view_precision)
    mu = posterior_cov @ (prior_precision @ pi + view_precision @ q)
    return (pd.Series(mu, index=ids, name="mu_bl"),
            pd.DataFrame(posterior_cov, index=ids, columns=ids))
```

**What it does.** Black–Litterman blends two sources of expected return. The **prior** `π` is what the market would expect from risk alone, `β_i × market premium × h/252`. The **views** `q = π + median forecast` come from Part 10's combined forecast. Each view's confidence is set by its own forecast's spread: `s = IQR / 1.349` (for a normal distribution the interquartile range is 1.349 standard deviations), so **a wide predictive distribution is a weak view**. The posterior is a precision-weighted average.

```python
import numpy as np
import pandas as pd

from aqp_python.stages.stages_03_04_portfolio.expected_returns import black_litterman

ids = ["A", "B", "C"]
covariance = pd.DataFrame(np.diag([5e-4, 5e-4, 5e-4]) + 1e-4, index=ids, columns=ids)
prior = pd.Series(0.0012, index=ids)                       # beta 1 x 6% a year over 5 days
forecasts = pd.DataFrame({"security_id": ids, "q0.500": [0.010, 0.010, -0.010],
                          "q0.250": [0.007, -0.020, -0.013], "q0.750": [0.013, 0.040, -0.007]})
mu, _ = black_litterman(forecasts, covariance, prior, tau=0.05)
summary = forecasts.assign(view=prior.to_numpy() + forecasts["q0.500"], iqr=forecasts["q0.750"] - forecasts["q0.250"],
                           posterior=mu.to_numpy())
print(summary[["security_id", "view", "iqr", "posterior"]].round(4).to_string(index=False))
```

```text
security_id    view   iqr  posterior
          A  0.0112 0.006     0.0108
          B  0.0112 0.060     0.0034
          C -0.0088 0.006    -0.0084
```

A and B have the same median view, but A's interquartile range is ten times narrower, so A's posterior stays close to its view while B's is pulled most of the way back to the prior of 0.0012. This is how forecast *uncertainty*, not just the forecast median, reaches the portfolio.

## 11.3 The optimiser and its relaxations

**File:** `src/aqp_python/stages/stages_03_04_portfolio/portfolio_optimizer.py` — component 1 of 3: docstring and imports

```python
"""Cost-aware mean-variance optimiser (SLSQP).

max_w  mu'w - (gamma/2) w'Sigma w - sum_i (c_i |dw_i| + eta_i |dw_i|^1.5) - kappa sum_i p_i |w_i|
s.t.   0 <= w_i <= min(name cap, liquidity cap_i),  sum w <= 1,
       sector sums <= sector cap,  sum |dw| <= turnover cap
"""
from dataclasses import dataclass

import numpy as np
import pandas as pd
from scipy.optimize import minimize

SMOOTH = 1e-8         # |x| ~ sqrt(x^2 + SMOOTH) keeps the problem differentiable
FEASIBILITY_TOL = 1e-6
```

The objective, term by term:

| Term | Meaning |
|---|---|
| `μ'w` | expected return |
| `(γ/2) w'Σw` | risk penalty, with risk aversion γ |
| `Σ c_i |Δw_i|` | spread and commission, linear in the trade |
| `Σ η_i |Δw_i|^1.5` | market impact, growing faster than the trade (the square-root law) |
| `κ Σ p_i |w_i|` | distress penalty, with `p_i` from the company reference |

**File:** `src/aqp_python/stages/stages_03_04_portfolio/portfolio_optimizer.py` — component 2 of 3: the problem and result types

```python
@dataclass(frozen=True)
class OptimisationProblem:
    mu: pd.Series
    covariance: pd.DataFrame
    current: pd.Series
    upper: pd.Series              # min(name cap, liquidity cap) per security
    sectors: pd.Series
    linear_cost: pd.Series        # c_i
    impact: pd.Series             # eta_i
    distress: pd.Series           # p_i
    risk_aversion: float
    distress_penalty: float
    max_sector_weight: float
    max_turnover: float


@dataclass(frozen=True)
class OptimisationResult:
    weights: pd.Series
    success: bool
    message: str
```

**File:** `src/aqp_python/stages/stages_03_04_portfolio/portfolio_optimizer.py` — component 3 of 3: optimise

```python
def _abs(x: np.ndarray) -> np.ndarray:
    return np.sqrt(x * x + SMOOTH)


def optimise(problem: OptimisationProblem) -> OptimisationResult:
    ids = list(problem.mu.index)
    mu, sigma = problem.mu.to_numpy(), problem.covariance.loc[ids, ids].to_numpy()
    w0 = problem.current.reindex(ids).fillna(0.0).to_numpy()
    c, eta = problem.linear_cost.loc[ids].to_numpy(), problem.impact.loc[ids].to_numpy()
    p, upper = problem.distress.loc[ids].to_numpy(), problem.upper.loc[ids].to_numpy()

    def negative_utility(w: np.ndarray) -> float:
        dw = _abs(w - w0)
        return -(mu @ w - 0.5 * problem.risk_aversion * w @ sigma @ w
                 - c @ dw - eta @ dw ** 1.5 - problem.distress_penalty * p @ _abs(w))

    constraints = [{"type": "ineq", "fun": lambda w: 1.0 - w.sum()},
                   {"type": "ineq", "fun": lambda w: problem.max_turnover - _abs(w - w0).sum()}]
    for sector in sorted(problem.sectors.loc[ids].unique()):
        mask = (problem.sectors.loc[ids] == sector).to_numpy()
        constraints.append({"type": "ineq",
                            "fun": lambda w, m=mask: problem.max_sector_weight - w[m].sum()})
    start = np.clip(w0, 0.0, upper)
    solution = minimize(negative_utility, start, method="SLSQP", bounds=list(zip(np.zeros(len(ids)), upper)),
                        constraints=constraints, options={"maxiter": 500, "ftol": 1e-12})
    w = solution.x
    feasible = (solution.success
                and all(con["fun"](w) >= -FEASIBILITY_TOL for con in constraints)
                and np.all(w >= -FEASIBILITY_TOL) and np.all(w <= upper + FEASIBILITY_TOL))
    weights = pd.Series(np.clip(w, 0.0, None), index=ids).where(lambda s: s > 1e-6, 0.0)
    return OptimisationResult(weights, bool(feasible), str(solution.message))
```

**What it does.** Solves the problem with SciPy's SLSQP, which handles bounds and inequality constraints. `|x|` is not differentiable at zero, so `_abs` uses `√(x² + 10⁻⁸)`, a smooth version. **After** solving, every constraint is checked again with a tolerance: the solver's own `success` flag is not trusted on its own. Weights below 10⁻⁶ are set to zero.

```python
import numpy as np
import pandas as pd

from aqp_python.stages.stages_03_04_portfolio.portfolio_optimizer import OptimisationProblem, optimise

ids = ["A", "B", "C", "D"]
problem = OptimisationProblem(
    mu=pd.Series([0.010, 0.008, 0.004, -0.002], index=ids),
    covariance=pd.DataFrame(np.diag([4e-4] * 4) + 1e-4, index=ids, columns=ids),
    current=pd.Series(0.0, index=ids), upper=pd.Series(0.40, index=ids),
    sectors=pd.Series(["banks", "banks", "mining", "mining"], index=ids),
    linear_cost=pd.Series(0.0005, index=ids), impact=pd.Series(0.0, index=ids),
    distress=pd.Series(0.0, index=ids), risk_aversion=2.0, distress_penalty=0.0,
    max_sector_weight=0.60, max_turnover=2.0)
for cap in (0.60, 0.30):
    result = optimise(OptimisationProblem(**{**problem.__dict__, "max_sector_weight": cap}))
    print(f"sector cap {cap:.2f}: success={result.success} weights={result.weights.round(3).to_dict()}")
```

```text
sector cap 0.60: success=True weights={'A': 0.4, 'B': 0.2, 'C': 0.4, 'D': 0.0}
sector cap 0.30: success=True weights={'A': 0.3, 'B': 0.0, 'C': 0.3, 'D': 0.0}
```

With a 60% sector cap, the two banks share 60% and the cap on each name (40%) binds for A and C. With a 30% cap, each sector can hold only its best name. D has a negative expected return and gets nothing: the sleeve is long-only.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/relaxation_policy.py`

```python
"""If the optimiser is infeasible, relax ONLY pre-approved constraints, in order.

Liquidity and tail-risk limits are never on the list. If nothing works, the
sleeve keeps its current positions and is flagged - an infeasible solution is
never turned into tradable weights.
"""
from dataclasses import replace

from aqp_python.stages.stages_03_04_portfolio.portfolio_optimizer import (
    OptimisationProblem,
    OptimisationResult,
    optimise,
)

RELAXABLE = {"max_turnover", "max_sector_weight"}


def optimise_with_relaxation(problem: OptimisationProblem, order: list[str],
                             step: float) -> tuple[OptimisationResult, list[str]]:
    unknown = set(order) - RELAXABLE
    if unknown:
        raise ValueError(f"constraints {sorted(unknown)} may not be relaxed")
    result = optimise(problem)
    relaxed: list[str] = []
    for name in order:
        if result.success:
            break
        problem = replace(problem, **{name: getattr(problem, name) * step})
        relaxed.append(f"{name}->{getattr(problem, name):.4f}")
        result = optimise(problem)
    return result, relaxed
```

**What it does.** If the problem is infeasible, loosens **only** the constraints in `relaxation_order` (from the sleeve file), one at a time, by `relaxation_step`, and records each change. Anything not in `RELAXABLE` raises an error before the optimiser even runs. **Why the list is in code and in configuration:** the configuration chooses the order; the code sets the limit. A configuration that asked to relax the liquidity cap would be refused, not obeyed.

## 11.4 Tail risk, stress and G4

**File:** `src/aqp_python/stages/stages_03_04_portfolio/portfolio_risk.py` — component 1 of 2: horizon returns, VaR and ES

```python
"""Tail risk (historical simulation) and the stress library."""
import numpy as np
import pandas as pd


def horizon_returns(daily: pd.DataFrame, horizon: int) -> pd.DataFrame:
    """Overlapping h-day compounded returns from daily returns."""
    return (1 + daily).rolling(horizon).apply(np.prod, raw=True).dropna() - 1


def var_es(weights: pd.Series, scenarios: pd.DataFrame, alpha: float = 0.975) -> tuple[float, float]:
    """Historical-simulation VaR and ES of the portfolio, reported as positive losses."""
    pnl = scenarios[weights.index].to_numpy() @ weights.to_numpy()
    var = -np.quantile(pnl, 1 - alpha)
    tail = pnl[pnl <= -var]
    es = -tail.mean() if len(tail) else var
    return float(var), float(es)
```

**What it does.** Builds overlapping h-day compounded returns from history, applies today's weights, and reports the 97.5% **VaR** (the loss exceeded on 2.5% of horizons) and **expected shortfall** (the average loss on those horizons), as positive numbers.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/portfolio_risk.py` — component 2 of 2: betas and stress_losses

```python
def betas(daily: pd.DataFrame, benchmark: pd.Series) -> pd.Series:
    bench = benchmark.reindex(daily.index)
    return daily.apply(lambda r: np.cov(r, bench)[0, 1] / bench.var())


def stress_losses(weights: pd.Series, reference: pd.DataFrame, beta: pd.Series,
                  scenarios: dict) -> dict[str, float]:
    """Loss (positive number) of each scenario applied through betas, revenue shares and sectors."""
    ref = reference.set_index("security_id").reindex(weights.index)
    losses = {}
    for name, shock in scenarios.items():
        impact = pd.Series(0.0, index=weights.index)
        if "market" in shock:
            impact += beta.reindex(weights.index).fillna(1.0) * shock["market"]
        if "revenue_cn" in shock:
            impact += ref["revenue_share_cn"].fillna(0.0) * shock["revenue_cn"]
        if "sector" in shock:
            impact += (ref["sector"] == shock["sector"]).astype(float) * shock["shock"]
        losses[name] = float(-(weights * impact).sum())
    return losses
```

**What it does.** Each stress scenario in the sleeve file is a combination of a market move (applied through each name's beta), a China revenue shock (through `revenue_share_cn` from the company reference) and a sector shock. **Why exposures matter here:** this is what the Company Reference and Exposure Dataset from Part 06 is for; a China slowdown hurts a miner with 60% Chinese revenue more than a bank, whatever their betas.

**File:** `src/aqp_python/gates/g4_portfolio_risk.py`

```python
"""G4 - portfolio risk: concentration, tail risk and stress loss of the target weights."""
from collections.abc import Mapping

import pandas as pd

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G4"
TOLERANCE = 1e-6


def check_portfolio(weights: pd.Series, sectors: pd.Series, expected_shortfall: float,
                    worst_stress_loss: float, portfolio_rules: Mapping,
                    gate_rules: Mapping) -> GateResult:
    codes, notes = [], []
    if (weights.abs() > portfolio_rules["max_name_weight"] + TOLERANCE).any():
        codes.append(RC.RISK_CONCENTRATION)
        notes.append("name weight above cap")
    sector_weights = weights.groupby(sectors.reindex(weights.index)).sum()
    if (sector_weights.abs() > portfolio_rules["max_sector_weight"] + TOLERANCE).any():
        codes.append(RC.RISK_CONCENTRATION)
        notes.append("sector weight above cap")
    if expected_shortfall > gate_rules["max_expected_shortfall"]:
        codes.append(RC.RISK_ES_BREACH)
        notes.append(f"ES {expected_shortfall:.4f}")
    if worst_stress_loss > gate_rules["max_stress_loss"]:
        codes.append(RC.RISK_STRESS_BREACH)
        notes.append(f"stress loss {worst_stress_loss:.4f}")
    evidence = {"expected_shortfall": expected_shortfall, "worst_stress_loss": worst_stress_loss,
                "max_sector_weight": float(sector_weights.abs().max()) if len(sector_weights) else 0.0}
    if codes:
        return GateResult(GATE, GateOutcome.BLOCK, "portfolio", tuple(dict.fromkeys(codes)),
                          "; ".join(notes), evidence)
    return GateResult(GATE, GateOutcome.PASS, "portfolio", (), "", evidence)
```

**What it does.** Blocks the target if a name or sector exceeds its cap, or if expected shortfall or the worst stress loss exceeds the G4 limits. Each failure has its own reason code.

## 11.5 The Stages 03–04 service

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 1 of 6: imports

```python
"""Stages 03-04 entry point: covariance -> Black-Litterman -> optimiser -> tail risk -> G4."""
import json

import numpy as np
import pandas as pd

from aqp_python.contracts.data_schemas import ResearchData
from aqp_python.contracts.forecast_schema import CombinedForecast
from aqp_python.contracts.portfolio_schema import PortfolioState, PortfolioTarget
from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.gates.g4_portfolio_risk import check_portfolio
from aqp_python.orchestration.run_context import RunContext
from aqp_python.orchestration.stage_status import StageResult
from aqp_python.stages.stages_03_04_portfolio.covariance_estimator import (
    daily_returns_window,
    ledoit_wolf_covariance,
)
from aqp_python.stages.stages_03_04_portfolio.expected_returns import black_litterman, equilibrium_returns
from aqp_python.stages.stages_03_04_portfolio.portfolio_optimizer import OptimisationProblem
from aqp_python.stages.stages_03_04_portfolio.portfolio_risk import (
    betas,
    horizon_returns,
    stress_losses,
    var_es,
)
from aqp_python.stages.stages_03_04_portfolio.relaxation_policy import optimise_with_relaxation

STAGE = "stages_03_04_portfolio"
RISK_LOOKBACK_DAYS = 750
```

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 2 of 6: _hold: the safe target

```python
def _hold(current: pd.Series, reason: str) -> PortfolioTarget:
    """Fallback target: keep what we have, create no new risk."""
    empty = pd.DataFrame(index=current.index, columns=current.index, dtype=float)
    return PortfolioTarget(current.copy(), current, pd.Series(0.0, index=current.index), empty,
                           empty, feasible=False, tradable=False, risk={"hold_reason": reason})
```

**What it does.** The safe fallback target: today's weights, `feasible=False`, `tradable=False`. **Why holding is safe, and flattening is not:** selling everything is itself a large set of trades, with costs and market impact, made because of a software or data problem.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 3 of 6: build_problem: costs, caps and penalties

```python
def build_problem(ctx: RunContext, data: ResearchData, ids: list[str], mu: pd.Series,
                  covariance: pd.DataFrame, current: pd.Series, nav: float) -> OptimisationProblem:
    pcfg, costs = ctx.config["sleeve"]["portfolio"], ctx.config["sleeve"]["costs"]
    recent = data.prices.loc[data.prices["security_id"].isin(ids)]
    recent = recent.loc[recent["date"] > ctx.decision_date - pd.offsets.BDay(60)]
    adv = recent.groupby("security_id")["dollar_volume"].median().reindex(ids)
    daily_vol = recent.groupby("security_id")["return"].std().reindex(ids)
    liquidity_cap = pcfg["participation_cap"] * adv * pcfg["exit_days"] / nav
    reference = data.company_reference.set_index("security_id")
    # Costs are paid once but a position is held for several horizons, so they are
    # amortised over `holding_periods` to compare with a one-horizon expected return.
    amortise = 1.0 / costs["holding_periods"]
    return OptimisationProblem(
        mu=mu, covariance=covariance, current=current.reindex(ids).fillna(0.0),
        upper=np.minimum(pcfg["max_name_weight"], liquidity_cap),
        sectors=reference.loc[ids, "sector"],
        linear_cost=pd.Series((costs["half_spread_bps"] + costs["commission_bps"]) * 1e-4 * amortise, index=ids),
        impact=costs["impact_coefficient"] * daily_vol * np.sqrt(nav / adv) * amortise,
        distress=reference.loc[ids, "distress_probability"],
        risk_aversion=pcfg["risk_aversion"], distress_penalty=pcfg["distress_penalty"],
        max_sector_weight=pcfg["max_sector_weight"], max_turnover=pcfg["max_turnover"])
```

**What it does.** Builds the optimiser's inputs. The upper bound on each name is the smaller of the name cap and a **liquidity cap**: the position we could exit in `exit_days` while trading at most `participation_cap` of the median daily dollar volume. Costs are divided by `holding_periods` because a cost paid once is spread over several horizons of expected return.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 4 of 6: run_stage: covariance, Black-Litterman, optimiser

```python
def run_stage(ctx: RunContext, data: ResearchData, combined: CombinedForecast,
              state: PortfolioState) -> tuple[StageResult, PortfolioTarget]:
    result = StageResult(STAGE)
    sleeve = ctx.config["sleeve"]
    pcfg, horizon = sleeve["portfolio"], ctx.config["forecast_models"]["horizon_days"]
    current = state.weights()

    if combined.combined.empty:
        result.reason_codes.append(str(RC.NO_ELIGIBLE_MODEL))
        result.warnings.append("no eligible forecasts: holding current positions (NO-TRADE)")
        target = _hold(current, str(RC.NO_ELIGIBLE_MODEL))
        _write(ctx, result, target, pd.Series(dtype=float))
        return result, target

    ids = sorted(combined.combined["security_id"])
    daily = daily_returns_window(data.prices, ids, ctx.decision_date, pcfg["covariance_lookback_days"])
    covariance = ledoit_wolf_covariance(daily, horizon)
    bench = data.benchmark.set_index("date")["benchmark_return"]
    prior = equilibrium_returns(betas(daily, bench), pcfg["market_risk_premium_annual"], horizon)
    mu, mu_cov = black_litterman(combined.combined, covariance, prior, pcfg["bl_tau"])
    problem = build_problem(ctx, data, ids, mu, covariance, current, state.nav)
    solution, relaxed = optimise_with_relaxation(problem, pcfg["relaxation_order"], pcfg["relaxation_step"])
    if relaxed:
        result.reason_codes.append(str(RC.OPT_RELAXED))
        result.warnings.append(f"approved relaxations used: {relaxed}")
    if not solution.success:
        result.reason_codes.append(str(RC.OPT_INFEASIBLE))
        result.warnings.append(f"optimiser infeasible ({solution.message}); holding current positions")
        target = _hold(current, str(RC.OPT_INFEASIBLE))
        _write(ctx, result, target, problem.upper)
        return result, target
```

**What it does.** No combined forecast means `NO_ELIGIBLE_MODEL` and hold. Otherwise covariance, prior, Black–Litterman, optimiser with relaxations; infeasible means `OPT_INFEASIBLE` and hold.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 5 of 6: run_stage: tail risk, stress and G4

```python
    # Names we hold but can no longer forecast (exits, quarantine) get a target of zero.
    weights = solution.weights.reindex(sorted(set(ids) | set(current.index))).fillna(0.0)

    # Tail risk and stress on the relative weights, with a scale-toward-cash response (G4).
    history = daily_returns_window(data.prices, ids, ctx.decision_date, RISK_LOOKBACK_DAYS)
    scenarios = horizon_returns(history, horizon)
    beta = betas(history, bench)
    gate_rules = ctx.config["gates"]["g4_portfolio_risk"]
    sectors = data.company_reference.set_index("security_id")["sector"]

    def assess(w: pd.Series):
        live = w.loc[ids]
        _, es = var_es(live, scenarios)
        stress = stress_losses(live, data.company_reference, beta, sleeve["stress_scenarios"])
        return es, stress, check_portfolio(live, sectors, es, max(stress.values()), pcfg, gate_rules)

    es, stress, gate = assess(weights)
    if not gate.passed and set(gate.reason_codes) <= {RC.RISK_ES_BREACH, RC.RISK_STRESS_BREACH}:
        scale = min(gate_rules["max_expected_shortfall"] / max(es, 1e-12),
                    gate_rules["max_stress_loss"] / max(max(stress.values()), 1e-12)) * 0.99
        weights.loc[ids] *= scale
        result.warnings.append(f"G4 breach: scaled toward cash by {scale:.3f}")
        es, stress, gate = assess(weights)
    result.gates.append(gate)
    tradable = gate.passed
    if not tradable:
        result.reason_codes.extend(map(str, gate.reason_codes))
        target = _hold(current, "G4")
        _write(ctx, result, target, problem.upper)
        return result, target

    covariance_all = covariance.reindex(index=weights.index, columns=weights.index).fillna(0.0)
    risk = {"expected_shortfall_975": es, "stress_losses": stress,
            "volatility_h": float(np.sqrt(weights @ covariance_all @ weights)),
            "gross_relative": float(weights.sum()), "names": int((weights > 0).sum())}
    target = PortfolioTarget(weights, current.reindex(weights.index).fillna(0.0),
                             mu.reindex(weights.index).fillna(0.0),
                             mu_cov.reindex(index=weights.index, columns=weights.index).fillna(0.0),
                             covariance_all, feasible=True, tradable=True, relaxations=relaxed, risk=risk)
    _write(ctx, result, target, problem.upper)
    result.metrics.update(risk)
    return result, target
```

**What it does.** Names we hold but can no longer forecast get a target of zero. Then expected shortfall and stress are computed and G4 checked. If the **only** breaches are tail-risk limits, the weights are scaled towards cash once, just enough to fit (with a 1% margin), and checked again. Concentration breaches are never scaled away: scaling every weight down would not fix *which* names are concentrated.

**File:** `src/aqp_python/stages/stages_03_04_portfolio/service.py` — component 6 of 6: _write: target weights and risk summary

```python
def _write(ctx: RunContext, result: StageResult, target: PortfolioTarget, upper: pd.Series) -> None:
    table = pd.DataFrame({"current_weight": target.current, "target_weight": target.weights,
                          "mu_bl": target.mu, "upper_bound": upper}).rename_axis("security_id")
    table.to_csv(ctx.paths.sub("portfolio", "target_weights.csv"))
    summary = {"feasible": target.feasible, "tradable": target.tradable,
               "relaxations": target.relaxations, **target.risk}
    ctx.paths.sub("portfolio", "risk_summary.json").write_text(json.dumps(summary, indent=2, default=str))
    result.artifacts.update({"target_weights": "portfolio/target_weights.csv",
                             "risk_summary": "portfolio/risk_summary.json"})
```

## 11.6 How much in total: leverage and G5

```text
u        = σ_p² + Var(μ_p)                    risk + uncertainty about the edge
L_kelly  = max(0, c · μ_p / u),   c ∈ [0.25, 0.5]   fractional Kelly
L_vol    = σ_target / σ_p                     volatility targeting
d_DD     = max(0, 1 − DD / DD_max)            drawdown brake
L        = min(L_vol, L_kelly, L_max, L_margin) · d_DD
```

**File:** `src/aqp_python/stages/stage_05_sizing/leverage_rules.py` — component 1 of 2: docstring and LeverageComponents

```python
"""Total exposure: L = min(L_vol, L_kelly, L_max, L_margin) * d_DD.

u       = sigma_p^2 + Var(mu_p)                 (risk plus uncertainty in the edge)
L_kelly = max(0, c * mu_p / u),  0.25 <= c <= 0.5
L_vol   = sigma_target / sigma_p
d_DD    = max(0, 1 - DD / DD_max)
Invalid inputs give L = 0 - they must never increase leverage.
"""
import math
from dataclasses import asdict, dataclass

from aqp_python.contracts.reason_codes import ReasonCode as RC


@dataclass(frozen=True)
class LeverageComponents:
    kelly: float
    volatility: float
    maximum: float
    margin: float
    drawdown_multiplier: float
    cap: float
    final: float
    reason_code: str

    def to_dict(self) -> dict:
        return asdict(self)
```

**File:** `src/aqp_python/stages/stage_05_sizing/leverage_rules.py` — component 2 of 2: compute_leverage

```python
def _finite(*values: float) -> bool:
    return all(isinstance(v, (int, float)) and math.isfinite(v) for v in values)


def compute_leverage(*, mu_p: float, sigma_p: float, mu_variance: float, target_vol_h: float,
                     kelly_fraction: float, max_leverage: float, margin_capacity: float,
                     drawdown: float, max_drawdown: float) -> LeverageComponents:
    if (not _finite(mu_p, sigma_p, mu_variance, target_vol_h, drawdown, max_drawdown)
            or sigma_p <= 0 or max_drawdown <= 0 or mu_variance < 0):
        return LeverageComponents(0, 0, max_leverage, margin_capacity, 0, 0, 0.0,
                                  str(RC.LEVERAGE_INVALID_INPUT))
    u = sigma_p ** 2 + mu_variance
    kelly = max(0.0, kelly_fraction * mu_p / u)
    volatility = target_vol_h / sigma_p
    cap = min(volatility, kelly, max_leverage, margin_capacity)
    multiplier = max(0.0, 1.0 - max(drawdown, 0.0) / max_drawdown)
    final = cap * multiplier
    if multiplier == 0:
        reason = str(RC.DRAWDOWN_LIMIT)
    elif final == 0:
        reason = str(RC.LEVERAGE_ZERO)
    else:
        reason = ""
    return LeverageComponents(kelly, volatility, max_leverage, margin_capacity,
                              multiplier, cap, final, reason)
```

**What it does.** Every input is checked first: anything non-finite, a zero volatility or a zero maximum drawdown gives leverage **0** and `LEVERAGE_INVALID_INPUT`. **Why zero and not "the last good value":** an input that cannot be trusted must never be able to *increase* risk.

```python
import numpy as np

from aqp_python.stages.stage_05_sizing.leverage_rules import compute_leverage

common = dict(sigma_p=0.025, target_vol_h=0.10 * np.sqrt(5 / 252), kelly_fraction=0.5,
              max_leverage=1.0, margin_capacity=0.8 / 0.5, max_drawdown=0.15)
print(f"{'mu_p':>6} {'Var(mu)':>8} {'DD':>5} | {'kelly':>6} {'vol':>6} {'cap':>6} {'d_DD':>5} {'final':>6} code")
for mu_p, mu_var, dd in [(0.001, 0.0, 0.0), (0.001, 6.25e-4, 0.0), (0.001, 6.25e-4, 0.075), (0.001, 0.0, 0.15),
                         (-0.001, 0.0, 0.0)]:
    lev = compute_leverage(mu_p=mu_p, mu_variance=mu_var, drawdown=dd, **common)
    print(f"{mu_p:>6} {mu_var:>8.2e} {dd:>5} | {lev.kelly:>6.3f} {lev.volatility:>6.3f} {lev.cap:>6.3f} "
          f"{lev.drawdown_multiplier:>5.2f} {lev.final:>6.3f} {lev.reason_code}")
```

```text
  mu_p  Var(mu)    DD |  kelly    vol    cap  d_DD  final code
 0.001 0.00e+00   0.0 |  0.800  0.563  0.563  1.00  0.563 
 0.001 6.25e-04   0.0 |  0.400  0.563  0.400  1.00  0.400 
 0.001 6.25e-04 0.075 |  0.400  0.563  0.400  0.50  0.200 
 0.001 0.00e+00  0.15 |  0.800  0.563  0.563  0.00  0.000 DRAWDOWN_LIMIT
-0.001 0.00e+00   0.0 |  0.000  0.563  0.000  1.00  0.000 LEVERAGE_ZERO
```

Row by row: volatility targeting binds first (0.563); adding uncertainty about the edge halves the Kelly cap, which then binds (0.400); a drawdown of half the limit halves the exposure (0.200); at the limit the brake stops all new risk; and a negative expected return gives zero.

**File:** `src/aqp_python/gates/g5_leverage_margin.py`

```python
"""G5 - leverage and margin: the final leverage must be finite, bounded and leave headroom."""
import math
from collections.abc import Mapping

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G5"


def check_leverage(leverage: float, max_leverage: float, margin_headroom: float,
                   drawdown_multiplier: float, rules: Mapping) -> GateResult:
    evidence = {"leverage": leverage, "max_leverage": max_leverage,
                "margin_headroom": margin_headroom, "drawdown_multiplier": drawdown_multiplier}
    if not math.isfinite(leverage) or leverage < 0:
        return GateResult(GATE, GateOutcome.BLOCK, "sleeve", (RC.LEVERAGE_INVALID_INPUT,),
                          "non-finite or negative leverage", evidence)
    codes = []
    if leverage > max_leverage + 1e-9:
        codes.append(RC.HARD_LIMIT_GROSS)
    if margin_headroom < rules["min_margin_headroom"] - 1e-9:
        codes.append(RC.MARGIN_HEADROOM)
    if drawdown_multiplier <= 0:
        codes.append(RC.DRAWDOWN_LIMIT)
    if codes:
        return GateResult(GATE, GateOutcome.BLOCK, "sleeve", tuple(codes), "block new risk", evidence)
    return GateResult(GATE, GateOutcome.PASS, "sleeve", (), "", evidence)
```

**What it does.** The final leverage must be finite and non-negative, within the maximum, leave the required margin headroom, and not be in a drawdown stop.

## 11.7 Name by name: the no-trade band, cost and G6

**File:** `src/aqp_python/gates/g6_trade_eligibility.py`

```python
"""G6 - final per-trade eligibility. Any failure blocks that order."""
from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.gates.gate_result import GateOutcome, GateResult

GATE = "G6"


def check_trade(security_id: str, *, data_age_days: int, max_data_age_days: int,
                quarantined: bool, has_eligible_forecast: bool, risk_reducing: bool,
                upstream_blocked: bool) -> GateResult:
    codes = []
    if data_age_days > max_data_age_days:
        codes.append(RC.DATA_TOO_OLD)
    if quarantined:
        codes.append(RC.DATA_STALE)
    if not has_eligible_forecast and not risk_reducing:
        codes.append(RC.NO_ELIGIBLE_MODEL)
    if upstream_blocked and not risk_reducing:
        codes.append(RC.UPSTREAM_BLOCKED)
    if codes:
        return GateResult(GATE, GateOutcome.BLOCK, security_id, tuple(codes), "order blocked")
    return GateResult(GATE, GateOutcome.PASS, security_id)
```

**What it does.** The last check on each order: data not too old, the name not quarantined, an eligible forecast exists, and nothing upstream blocked new risk. **The asymmetry:** a *risk-reducing* order passes the forecast and upstream checks. If the models fail, you must still be able to sell a name that left the universe.

**File:** `src/aqp_python/stages/stage_05_sizing/trade_decision.py` — component 1 of 2: imports

```python
"""Turn target weights into TRADE / REDUCE / HOLD / NO-TRADE decisions and order requests."""
import numpy as np
import pandas as pd

from aqp_python.contracts.order_schema import OrderRequest, Side
from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.contracts.run_status import TradeDecision
from aqp_python.gates.g6_trade_eligibility import check_trade
from aqp_python.gates.gate_result import GateResult
```

**File:** `src/aqp_python/stages/stage_05_sizing/trade_decision.py` — component 2 of 2: decide_trades: band, cost, G6, order

```python
def decide_trades(*, final_weights: pd.Series, current: pd.Series, mu: pd.Series,
                  prices: pd.Series, impact: pd.Series, linear_cost: float, nav: float,
                  band: float, min_ratio: float, holding_periods: float, forecast_ids: set[str], quarantined: set[str],
                  data_age_days: int, max_data_age_days: int, new_risk_allowed: bool,
                  run_id: str) -> tuple[pd.DataFrame, list[OrderRequest], list[GateResult]]:
    rows, orders, gates = [], [], []
    for sid in sorted(set(final_weights.index) | set(current.index)):
        target, now = float(final_weights.get(sid, 0.0)), float(current.get(sid, 0.0))
        dw = target - now
        exit_trade = target == 0.0 and now > 0.0
        risk_reducing = dw < 0
        price = float(prices.get(sid, np.nan))
        cost = abs(dw) * nav * (linear_cost + float(impact.get(sid, 0.0)) * abs(dw) ** 0.5)
        benefit = abs(dw) * nav * abs(float(mu.get(sid, 0.0))) * holding_periods
        row = {"security_id": sid, "current_weight": now, "target_weight": target, "delta_weight": dw,
               "expected_benefit": benefit, "expected_cost": cost}
        if abs(dw) < band and not exit_trade:
            action = TradeDecision.HOLD if now > 0 else TradeDecision.NO_TRADE
            rows.append({**row, "action": str(action), "reason_code": str(RC.WITHIN_NO_TRADE_BAND)})
            continue
        if not risk_reducing and not new_risk_allowed:
            rows.append({**row, "action": str(TradeDecision.NO_TRADE), "reason_code": str(RC.UPSTREAM_BLOCKED)})
            continue
        if not exit_trade and benefit < min_ratio * cost:
            action = TradeDecision.HOLD if now > 0 else TradeDecision.NO_TRADE
            rows.append({**row, "action": str(action), "reason_code": str(RC.BELOW_COST_THRESHOLD)})
            continue
        gate = check_trade(sid, data_age_days=data_age_days, max_data_age_days=max_data_age_days,
                           quarantined=sid in quarantined and not risk_reducing,
                           has_eligible_forecast=sid in forecast_ids, risk_reducing=risk_reducing,
                           upstream_blocked=not new_risk_allowed)
        gates.append(gate)
        quantity = int(abs(dw) * nav // price) if np.isfinite(price) and price > 0 else 0
        if not gate.passed or quantity == 0:
            code = gate.reason_codes[0] if gate.reason_codes else RC.BELOW_COST_THRESHOLD
            rows.append({**row, "action": str(TradeDecision.NO_TRADE), "reason_code": str(code)})
            continue
        action = TradeDecision.REDUCE if risk_reducing else TradeDecision.TRADE
        order = OrderRequest(order_id=f"{run_id}-{sid}", security_id=sid,
                             side=Side.SELL if risk_reducing else Side.BUY, quantity=quantity,
                             reference_price=price, reason_code=str(RC.TRADE_APPROVED))
        order.validate()
        orders.append(order)
        rows.append({**row, "action": str(action), "reason_code": str(RC.TRADE_APPROVED),
                     "quantity": quantity, "side": str(order.side)})
    return pd.DataFrame(rows), orders, gates
```

**What it does.** For every name in the target or the book, in this order:

1. **Band.** If `|Δw|` is below `no_trade_band` (and it is not an exit), do nothing: `WITHIN_NO_TRADE_BAND`.
2. **No new risk.** If the trade adds risk and new risk is not allowed: `UPSTREAM_BLOCKED`.
3. **Cost.** If the expected benefit `|Δw| · NAV · |μ| · holding_periods` is less than `min_benefit_cost_ratio` times the cost: `BELOW_COST_THRESHOLD`. Exits skip this test.
4. **G6.** Then round down to whole shares and build an `OrderRequest`, validated by the contract from Part 02.

```python
import pandas as pd

from aqp_python.stages.stage_05_sizing.trade_decision import decide_trades

ids = ["A", "B", "C", "D"]
table, orders, gates = decide_trades(
    final_weights=pd.Series({"A": 0.050, "B": 0.003, "C": 0.000, "D": 0.020}),
    current=pd.Series({"A": 0.000, "B": 0.000, "C": 0.030, "D": 0.000}),
    mu=pd.Series({"A": 0.010, "B": 0.010, "C": 0.000, "D": 0.0005}),
    prices=pd.Series(10.0, index=ids), impact=pd.Series(0.0, index=ids), linear_cost=0.0015,
    nav=1_000_000, band=0.005, min_ratio=1.5, holding_periods=2, forecast_ids={"A", "B", "D"},
    quarantined=set(), data_age_days=0, max_data_age_days=1, new_risk_allowed=True, run_id="demo")
print(table[["security_id", "delta_weight", "expected_benefit", "expected_cost", "action", "reason_code"]]
      .round(4).to_string(index=False))
print("orders:", [(o.security_id, str(o.side), o.quantity) for o in orders])
```

```text
security_id  delta_weight  expected_benefit  expected_cost   action          reason_code
          A         0.050            1000.0           75.0    TRADE       TRADE_APPROVED
          B         0.003              60.0            4.5 NO-TRADE WITHIN_NO_TRADE_BAND
          C        -0.030               0.0           45.0   REDUCE       TRADE_APPROVED
          D         0.020              20.0           30.0 NO-TRADE BELOW_COST_THRESHOLD
orders: [('A', 'BUY', 5000), ('C', 'SELL', 3000)]
```

A is worth buying; B's change is inside the 0.5% band; C must go because it is no longer forecast, so it is sold even though the "benefit" is zero; D's expected gain of $20 does not cover 1.5 × its $30 cost.

**File:** `src/aqp_python/stages/stage_05_sizing/position_report.py`

```python
"""Every security gets a report line - trade or NO-TRADE - with its reason code."""
import pandas as pd

from aqp_python.contracts.quantile_grid import LOWER_90, MEDIAN_COLUMN, UPPER_90


def build_position_report(decisions: pd.DataFrame, combined: pd.DataFrame,
                          reference: pd.DataFrame) -> pd.DataFrame:
    forecast = combined.set_index("security_id")[[LOWER_90, MEDIAN_COLUMN, UPPER_90]]
    forecast.columns = ["forecast_q05", "forecast_median", "forecast_q95"]
    ref = reference.set_index("security_id")[["sector", "distress_probability", "revenue_share_cn"]]
    return (decisions.set_index("security_id").join(forecast).join(ref).reset_index())
```

**What it does.** Joins each decision to its forecast interval and exposures, so the report shows, for every name, what was forecast, what was decided and why.

## 11.8 The Stage 05 service

**File:** `src/aqp_python/stages/stage_05_sizing/service.py` — component 1 of 3: imports

```python
"""Stage 05 entry point: leverage (G5) -> per-name decisions (G6) -> order requests."""
import json

import numpy as np
import pandas as pd

from aqp_python.contracts.data_schemas import ResearchData
from aqp_python.contracts.forecast_schema import CombinedForecast
from aqp_python.contracts.order_schema import OrderRequest
from aqp_python.contracts.portfolio_schema import PortfolioState, PortfolioTarget
from aqp_python.contracts.universe_schema import UniverseSelection
from aqp_python.gates.g5_leverage_margin import check_leverage
from aqp_python.orchestration.run_context import RunContext
from aqp_python.orchestration.stage_status import StageResult
from aqp_python.stages.stage_05_sizing.leverage_rules import compute_leverage
from aqp_python.stages.stage_05_sizing.position_report import build_position_report
from aqp_python.stages.stage_05_sizing.trade_decision import decide_trades

STAGE = "stage_05_sizing"
```

**File:** `src/aqp_python/stages/stage_05_sizing/service.py` — component 2 of 3: run_stage: total exposure (G5)

```python
def run_stage(ctx: RunContext, data: ResearchData, universe: UniverseSelection,
              combined: CombinedForecast, target: PortfolioTarget,
              state: PortfolioState) -> tuple[StageResult, list[OrderRequest]]:
    result = StageResult(STAGE)
    sleeve = ctx.config["sleeve"]
    sizing, costs = sleeve["sizing"], sleeve["costs"]
    horizon = ctx.config["forecast_models"]["horizon_days"]
    w = target.weights

    # 1. Leverage: only when Stages 03-04 produced a tradable target with some exposure.
    equity = state.equity_history
    drawdown = float(1 - equity.iloc[-1] / equity.cummax().iloc[-1]) if len(equity) else 0.0
    margin_capacity = (1 - sizing["margin_buffer"]) / sizing["initial_margin_rate"]
    lev, gross, new_risk_allowed, mu_p, sigma_p = None, 0.0, False, None, None
    if target.tradable and w.sum() > 0:
        mu_p = float(w @ target.mu.reindex(w.index).fillna(0.0))              # h-day expected return
        sigma_p = float(np.sqrt(w @ target.covariance.loc[w.index, w.index] @ w))
        lev = compute_leverage(
            mu_p=mu_p, sigma_p=sigma_p,
            mu_variance=float(w @ target.mu_covariance.loc[w.index, w.index] @ w),
            target_vol_h=sizing["target_volatility_annual"] * np.sqrt(horizon / 252),
            kelly_fraction=sizing["kelly_fraction"], max_leverage=sizing["max_leverage"],
            margin_capacity=margin_capacity, drawdown=drawdown, max_drawdown=sizing["max_drawdown"])
        gross = lev.final * float(w.sum())
        headroom = 1 - gross * sizing["initial_margin_rate"]
        g5 = check_leverage(gross, sizing["max_leverage"], headroom, lev.drawdown_multiplier,
                            ctx.config["gates"]["g5_leverage_margin"])
        result.gates.append(g5)
        new_risk_allowed = g5.passed and lev.final > 0
        if lev.reason_code:
            result.reason_codes.append(lev.reason_code)
```

**What it does.** Leverage is computed only for a tradable target with some exposure. `mu_variance = w' Var(μ) w` comes from Black–Litterman's posterior uncertainty: the less sure the forecasts, the smaller the Kelly cap.

**File:** `src/aqp_python/stages/stage_05_sizing/service.py` — component 3 of 3: run_stage: per-name decisions (G6) and the position report

```python
    # 2. Per-security decisions, cost/benefit, no-trade band and G6.
    if new_risk_allowed:
        final_weights = w * lev.final
    else:   # no new risk: hold, but still close names that left the universe or are quarantined
        final_weights = target.current.where(target.current.index.isin(universe.active_ids), 0.0)
    latest = data.prices.loc[data.prices["date"] == data.prices["date"].max()]
    prices = latest.set_index("security_id")["close"]
    recent = data.prices.loc[data.prices["date"] > ctx.decision_date - pd.offsets.BDay(60)]
    adv = recent.groupby("security_id")["dollar_volume"].median()
    daily_vol = recent.groupby("security_id")["return"].std()
    impact = costs["impact_coefficient"] * daily_vol * np.sqrt(state.nav / adv)
    data_age = len(data.trading_days[data.trading_days > data.prices["date"].max()])
    decisions, orders, g6 = decide_trades(
        final_weights=final_weights, current=target.current, mu=target.mu, prices=prices,
        impact=impact, linear_cost=(costs["half_spread_bps"] + costs["commission_bps"]) * 1e-4,
        nav=state.nav, band=sizing["no_trade_band"], min_ratio=sizing["min_benefit_cost_ratio"],
        holding_periods=costs["holding_periods"],
        forecast_ids=set(combined.combined["security_id"]), quarantined=set(data.quarantined),
        data_age_days=data_age, max_data_age_days=ctx.config["gates"]["g6_trade_eligibility"]["max_data_age_days"],
        new_risk_allowed=new_risk_allowed, run_id=ctx.run_id)
    result.gates.extend(g6)

    report = (build_position_report(decisions, combined.combined, data.company_reference)
              if len(decisions) else decisions)
    report.to_csv(ctx.paths.sub("sizing", "position_report.csv"), index=False)
    pd.DataFrame([o.to_dict() for o in orders]).to_csv(ctx.paths.sub("sizing", "order_requests.csv"), index=False)
    summary = {"leverage": lev.to_dict() if lev else None, "gross_exposure": gross, "drawdown": drawdown,
               "nav": state.nav, "sigma_p_h": sigma_p, "mu_p_h": mu_p, "orders": len(orders),
               "new_risk_allowed": new_risk_allowed}
    ctx.paths.sub("sizing", "sizing_summary.json").write_text(json.dumps(summary, indent=2, default=str))
    result.artifacts.update({"position_report": "sizing/position_report.csv",
                             "order_requests": "sizing/order_requests.csv",
                             "sizing_summary": "sizing/sizing_summary.json"})
    actions = decisions["action"].value_counts().to_dict() if len(decisions) else {}
    result.metrics.update({"leverage": lev.final if lev else 0.0, "gross_exposure": gross,
                           "orders": len(orders), "actions": actions})
    return result, orders
```

**What it does.** If new risk is allowed, the final weights are the target times the leverage. If not, the book is held, **except** that names no longer active in the universe are set to zero, so they can still be sold. Then decisions, orders, the position report and a sizing summary.

## 11.9 Tests

**File:** `tests/unit/test_portfolio_constraints.py`

```python
import numpy as np
import pandas as pd
import pytest

from aqp_python.stages.stages_03_04_portfolio.expected_returns import black_litterman
from aqp_python.stages.stages_03_04_portfolio.portfolio_optimizer import OptimisationProblem, optimise
from aqp_python.stages.stages_03_04_portfolio.relaxation_policy import optimise_with_relaxation

pytestmark = pytest.mark.batch07
IDS = ["A", "B", "C", "D"]


def problem(**overrides) -> OptimisationProblem:
    base = dict(
        mu=pd.Series([0.010, 0.008, 0.002, -0.004], index=IDS),
        covariance=pd.DataFrame(np.diag([4e-4] * 4) + 1e-4, index=IDS, columns=IDS),
        current=pd.Series(0.0, index=IDS), upper=pd.Series(0.4, index=IDS),
        sectors=pd.Series(["x", "x", "y", "y"], index=IDS),
        linear_cost=pd.Series(0.0005, index=IDS), impact=pd.Series(0.0, index=IDS),
        distress=pd.Series(0.0, index=IDS), risk_aversion=2.0, distress_penalty=0.0,
        max_sector_weight=0.6, max_turnover=2.0)
    return OptimisationProblem(**{**base, **overrides})


def test_optimiser_respects_bounds_and_sector_caps():
    result = optimise(problem())
    w = result.weights
    assert result.success
    assert (w <= 0.4 + 1e-6).all() and w[["A", "B"]].sum() <= 0.6 + 1e-6
    assert w["A"] > w["C"] and w["D"] == 0.0


def test_impossible_constraints_are_reported_as_infeasible():
    # We hold 60% in A, A's cap is now 10%, and turnover is frozen: no feasible point exists.
    result = optimise(problem(current=pd.Series([0.6, 0, 0, 0], index=IDS), max_turnover=0.0,
                              upper=pd.Series(0.1, index=IDS)))
    assert not result.success


def test_relaxation_only_touches_approved_constraints():
    with pytest.raises(ValueError, match="may not be relaxed"):
        optimise_with_relaxation(problem(), ["participation_cap"], 1.5)
    tight = problem(current=pd.Series([0.3, 0, 0, 0], index=IDS), max_turnover=0.0,
                    upper=pd.Series(0.25, index=IDS))
    result, relaxed = optimise_with_relaxation(tight, ["max_turnover"], 1.5)
    assert relaxed and relaxed[0].startswith("max_turnover")


def test_black_litterman_moves_from_prior_toward_confident_views():
    cov = pd.DataFrame(np.diag([4e-4, 4e-4]), index=["A", "B"], columns=["A", "B"])
    prior = pd.Series([0.001, 0.001], index=["A", "B"])
    forecasts = pd.DataFrame({"security_id": ["A", "B"], "q0.500": [0.02, 0.0],
                              "q0.250": [-0.0001, -0.1], "q0.750": [0.0001, 0.1]})
    mu, _ = black_litterman(forecasts, cov, prior, tau=0.05)
    assert mu["A"] == pytest.approx(0.021, rel=0.01)       # tight IQR: view dominates
    assert mu["B"] == pytest.approx(0.001, abs=2e-4)       # wide IQR: stays near prior
```

**File:** `tests/unit/test_covariance.py`

```python
import numpy as np
import pandas as pd
import pytest

from aqp_python.stages.stages_03_04_portfolio.covariance_estimator import ledoit_wolf_covariance
from aqp_python.stages.stages_03_04_portfolio.portfolio_risk import stress_losses, var_es

pytestmark = pytest.mark.batch07


def test_covariance_scales_with_horizon_and_needs_enough_history():
    returns = pd.DataFrame(np.random.default_rng(0).normal(0, 0.01, (250, 3)), columns=list("ABC"))
    one, five = ledoit_wolf_covariance(returns, 1), ledoit_wolf_covariance(returns, 5)
    assert np.allclose(five.to_numpy(), 5 * one.to_numpy())
    with pytest.raises(ValueError, match="need at least 60"):
        ledoit_wolf_covariance(returns.head(30), 5)


def test_expected_shortfall_is_at_least_var():
    scenarios = pd.DataFrame({"A": np.linspace(-0.1, 0.1, 401)})
    var, es = var_es(pd.Series({"A": 1.0}), scenarios)
    assert es >= var > 0


def test_china_stress_runs_through_revenue_exposure():
    reference = pd.DataFrame({"security_id": ["A", "B"], "sector": ["x", "y"], "revenue_share_cn": [0.5, 0.0]})
    losses = stress_losses(pd.Series({"A": 0.5, "B": 0.5}), reference, pd.Series({"A": 1.0, "B": 1.0}),
                           {"china": {"revenue_cn": -0.2}})
    assert losses["china"] == pytest.approx(0.05)             # 0.5 weight x 0.5 share x 20%
```

**File:** `tests/unit/test_kelly_leverage.py`

```python
import pytest

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.stages.stage_05_sizing.leverage_rules import compute_leverage

pytestmark = pytest.mark.batch07


def test_leverage_is_the_smallest_cap_times_the_drawdown_multiplier():
    lev = compute_leverage(mu_p=0.01, sigma_p=0.02, mu_variance=0.0, target_vol_h=0.01,
                           kelly_fraction=0.5, max_leverage=1.0, margin_capacity=0.8,
                           drawdown=0.05, max_drawdown=0.10)
    assert lev.volatility == pytest.approx(0.5) and lev.cap == pytest.approx(0.5)
    assert lev.final == pytest.approx(0.25)


@pytest.mark.parametrize("kwargs", [dict(sigma_p=float("nan")), dict(sigma_p=0.0), dict(max_drawdown=0.0)])
def test_invalid_inputs_never_increase_leverage(kwargs):
    base = dict(mu_p=0.01, sigma_p=0.02, mu_variance=0.0, target_vol_h=0.01, kelly_fraction=0.5,
                max_leverage=1.0, margin_capacity=0.8, drawdown=0.0, max_drawdown=0.1)
    lev = compute_leverage(**{**base, **kwargs})
    assert lev.final == 0.0 and lev.reason_code == RC.LEVERAGE_INVALID_INPUT
```

**File:** `tests/unit/test_no_trade_band.py`

```python
import pandas as pd
import pytest

from aqp_python.contracts.reason_codes import ReasonCode as RC
from aqp_python.stages.stage_05_sizing.trade_decision import decide_trades

pytestmark = pytest.mark.batch07


def decisions(**overrides):
    base = dict(final_weights=pd.Series({"A": 0.05, "B": 0.002, "C": 0.0}),
                current=pd.Series({"A": 0.0, "B": 0.0, "C": 0.03}),
                mu=pd.Series({"A": 0.01, "B": 0.01, "C": 0.0}), prices=pd.Series({"A": 10.0, "B": 10.0, "C": 10.0}),
                impact=pd.Series(0.0, index=["A", "B", "C"]), linear_cost=0.0015, nav=1e6, band=0.005,
                min_ratio=1.5, holding_periods=1, forecast_ids={"A", "B"}, quarantined=set(),
                data_age_days=0, max_data_age_days=1, new_risk_allowed=True, run_id="r")
    return decide_trades(**{**base, **overrides})


def test_band_cost_and_exit_rules():
    table, orders, _ = decisions()
    by_id = table.set_index("security_id")
    assert by_id.loc["A", "action"] == "TRADE"
    assert by_id.loc["B", "reason_code"] == RC.WITHIN_NO_TRADE_BAND
    assert by_id.loc["C", "action"] == "REDUCE"          # name left the forecast set: exit
    assert {o.security_id for o in orders} == {"A", "C"}


def test_no_new_risk_still_allows_exits():
    table, orders, _ = decisions(new_risk_allowed=False)
    assert {o.security_id for o in orders} == {"C"}
    assert table.set_index("security_id").loc["A", "reason_code"] == RC.UPSTREAM_BLOCKED
```

Add the G4–G6 component to `tests\unit\test_gates.py`:

**File:** `tests/unit/test_gates.py` — component 6 of 6: G4, G5 and G6 (Part 11)

```python
@pytest.mark.batch07
def test_g4_concentration_and_tail_risk():
    import pandas as pd

    from aqp_python.gates.g4_portfolio_risk import check_portfolio
    weights, sectors = pd.Series({"A": 0.09, "B": 0.05}), pd.Series({"A": "x", "B": "y"})
    rules = {"max_name_weight": 0.08, "max_sector_weight": 0.30}
    result = check_portfolio(weights, sectors, 0.07, 0.10, rules, {"max_expected_shortfall": 0.06,
                                                                     "max_stress_loss": 0.25})
    assert set(result.reason_codes) == {RC.RISK_CONCENTRATION, RC.RISK_ES_BREACH}


@pytest.mark.batch07
def test_g5_blocks_non_finite_or_excess_leverage():
    from aqp_python.gates.g5_leverage_margin import check_leverage
    assert check_leverage(float("nan"), 1.0, 0.5, 1.0, {"min_margin_headroom": 0.2}).reason_codes == (
        RC.LEVERAGE_INVALID_INPUT,)
    assert RC.HARD_LIMIT_GROSS in check_leverage(1.2, 1.0, 0.5, 1.0, {"min_margin_headroom": 0.2}).reason_codes


@pytest.mark.batch07
def test_g6_lets_risk_reducing_orders_through_without_a_forecast():
    from aqp_python.gates.g6_trade_eligibility import check_trade
    common = dict(data_age_days=0, max_data_age_days=1, quarantined=False, has_eligible_forecast=False,
                  upstream_blocked=False)
    assert check_trade("A", risk_reducing=True, **common).passed
    assert RC.NO_ELIGIBLE_MODEL in check_trade("A", risk_reducing=False, **common).reason_codes
```

The invariant: a hold target can never be traded on.

**File:** `tests/invariants/test_infeasible_means_no_trade.py`

```python
import pandas as pd
import pytest

from aqp_python.contracts.portfolio_schema import PortfolioTarget
from aqp_python.stages.stages_03_04_portfolio.service import _hold

pytestmark = pytest.mark.invariant


def test_hold_target_is_never_tradable():
    current = pd.Series({"A": 0.05, "B": 0.02})
    target = _hold(current, "OPT_INFEASIBLE")
    assert isinstance(target, PortfolioTarget)
    assert not target.tradable and not target.feasible
    pd.testing.assert_series_equal(target.weights, current)
```

The checkpoint, Stages 00 to 05 on the sample data with an empty book:

**File:** `tests/integration/test_stage_handoffs.py` — component 8 of 9: test_stages_03_to_05

```python
@pytest.mark.batch07
def test_stages_03_to_05(make_context):
    ctx = make_context()
    out = run_through(ctx, "stage_05_sizing")
    target, orders = out["target"], out["orders"]
    print("\n", target.weights[target.weights > 1e-4].round(4), "\norders:", len(orders))
    assert target.tradable and target.weights.sum() <= 1 + 1e-9
    assert orders and all(o.side == "BUY" for o in orders)     # an empty book can only buy
```

```powershell
python -m pytest -m batch07 -q
python -m pytest tests\integration\test_stage_handoffs.py -k 03_to_05 -s -q --basetemp=build\pytest
```

```text
SYN04    0.0038
SYN10    0.0800
SYN12    0.0800
SYN28    0.0800
SYN31    0.0001
SYN32    0.0800
SYN33    0.0597
SYN38    0.0800
SYN40    0.0800
orders: 8
1 passed in 6.74s
```

Several names sit at the 8% name cap. A tiny weight such as SYN31's falls inside the no-trade band after leverage, which is why there are fewer orders than names with weight.

**Experiments.**

1. In `configs\sleeves\sleeve_asx_interday.yaml`, set `max_sector_weight: 0.05` and rerun. The problem is still feasible, because the weights only have to sum to *at most* 1: the optimiser simply invests less, with no sector above 5%. Infeasibility needs a book that already breaks the new limits and a turnover cap too small to fix it in one day, which is what `test_impossible_constraints_are_reported_as_infeasible` builds. Set it back.
2. Set `max_expected_shortfall` in `configs\gates.yaml` to `0.005` and rerun. Stages 03–04 now warn `G4 breach: scaled toward cash by 0.146`, and the target weights sum to about 0.09 instead of about 0.60. Set it to `0.0005`: the scale is 0.015, every trade falls inside the no-trade band, and there are no orders. Set it back to `0.06`.

## Checkpoint

- [ ] `python -m pytest -m batch07 -q` → `17 passed`.
- [ ] The Stages 03–05 checkpoint passes and every order is a BUY.
- [ ] You can name the two situations in which Stage 05 still sends orders when new risk is not allowed.

**Check your understanding.** (1) Two names have the same median forecast. Why might Black–Litterman give them very different expected returns? (2) Why does a concentration breach lead to holding, while a tail-risk breach leads to scaling towards cash? (3) Why does the cost test not apply to exits?

```powershell
git add .
git commit -m "Stages 03-05: Black-Litterman, cost-aware optimiser, G4 risk, leverage, G5, trade decisions, G6"
```

Next: [Part 12 — safety, execution, monitoring and reporting](12_safety_execution_monitoring_reporting.md).
