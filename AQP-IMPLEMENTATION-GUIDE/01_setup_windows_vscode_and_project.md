# Part 01 — Windows, PowerShell, VS Code and the project skeleton

**Goal:** an `AQP-PYTHON` folder that VS Code, PowerShell, pip and pytest all agree on. That means one Python interpreter (the project's `.venv`), an installable package called `aqp_python`, and a script that tells you when something is wrong.

Most "it worked yesterday" problems on Windows come from two Pythons: pip installs a library into one while VS Code or pytest runs the other. Everything in this part exists to prevent that.

---

## 1.1 Install the tools once

| Tool | Why | Check in PowerShell |
|---|---|---|
| Python 3.11 or newer (python.org installer, tick **"Add python.exe to PATH"** and keep the **py launcher**) | Runs the pipeline. 3.11 is the minimum because the code uses `enum.StrEnum` and `datetime.UTC`. | `py --version` and `py -0p` (lists every installed Python) |
| Git | Small, reversible steps and a history of decisions | `git --version` |
| VS Code + the **Python**, **Python Debugger** and **Ruff** extensions | Editor, test explorer, debugger | `code --version` |

If `py` is not recognised, reinstall Python with the launcher option, then close and reopen the terminal. Do not try to work around it by mixing interpreters.

## 1.2 Create the folder and open it

```powershell
Set-Location $HOME\Documents          # or wherever you keep projects
New-Item -ItemType Directory -Name AQP-PYTHON | Out-Null
Set-Location .\AQP-PYTHON
code .
```

In VS Code, open **Terminal → New Terminal** and make sure the dropdown shows **PowerShell**. The prompt should end in `AQP-PYTHON>`. Every later command runs from here.

## 1.3 Create and activate the virtual environment

```powershell
py -3.11 -m venv .venv                 # or -3.12 / -3.13: any 3.11+ works
.\.venv\Scripts\Activate.ps1
python -c "import sys; print(sys.executable)"
```

The last line must print a path ending in `AQP-PYTHON\.venv\Scripts\python.exe`, and the prompt now starts with `(.venv)`.

**If activation is blocked** with "running scripts is disabled on this system", allow scripts for this terminal session only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
```

`-Scope Process` affects only the current window, so it does not weaken the machine's policy. If your organisation forbids even that, skip activation and call the interpreter directly: `.\.venv\Scripts\python.exe -m pytest` works exactly like `python -m pytest`.

## 1.4 Create the skeleton in one go

Paste this into PowerShell. It creates every folder you need and an empty `__init__.py` in each Python package. It never overwrites an existing file, so it is safe to run again.

```powershell
$folders = @("configs\sleeves", "data\samples", "docs\design", "docs\decisions", "scripts",
             "tests\unit", "tests\integration", "tests\invariants", ".vscode")
$packages = @("", "config", "contracts", "gates", "connectors", "connectors\broker", "storage",
              "orchestration", "safety", "compiler", "reporting", "stages",
              "stages\stage_00_data", "stages\stage_01_universe", "stages\stage_02_forecasting",
              "stages\stage_02_forecasting\financial", "stages\stage_02_forecasting\validation",
              "stages\stages_03_04_portfolio", "stages\stage_05_sizing",
              "stages\stage_06_execution", "stages\stage_07_monitoring")
foreach ($f in $folders) { New-Item -ItemType Directory -Force -Path $f | Out-Null }
foreach ($p in $packages) {
    $dir = Join-Path "src\aqp_python" $p
    New-Item -ItemType Directory -Force -Path $dir | Out-Null
    $init = Join-Path $dir "__init__.py"
    if (-not (Test-Path $init)) { New-Item -ItemType File -Path $init | Out-Null }
}
if (-not (Test-Path "tests\__init__.py")) { New-Item -ItemType File -Path "tests\__init__.py" | Out-Null }
(Get-ChildItem src, tests -Recurse -Filter __init__.py).Count
```

The last line should print `22`. Copy `AQP_OVERVIEW.md` into `docs\design\` now. It is the frozen specification this code answers to.

**Why `src\aqp_python` and not just `aqp_python`?** With the "src layout", Python can import your package only after it is *installed* (section 1.6). Tests therefore exercise the same import path the CLI will use, and a stray file in the project root can never shadow the real package.

**Why an `__init__.py` everywhere?** It marks a folder as a regular package, so `from aqp_python.stages.stage_00_data import service` works. `tests\__init__.py` lets tests import shared helpers with `from tests.conftest import ...`.

## 1.5 Packaging and housekeeping files

`pyproject.toml` is the single source of truth for the package name, Python version, dependencies, the `aqp` command, and the pytest and ruff settings.

**File:** `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "aqp-python"
version = "0.1.0"
description = "AQP - Advanced Quantitative Pipeline (research and paper-trading build)"
requires-python = ">=3.11"
dependencies = [
  "numpy>=1.26",
  "pandas>=2.1",
  "scipy>=1.11",
  "scikit-learn>=1.3",
  "PyYAML>=6.0",
  "matplotlib>=3.8",
  "tzdata>=2024.1",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0",
  "import-linter>=2.0",
  "ruff>=0.5",
]

[project.scripts]
aqp = "aqp_python.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 120
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "UP"]
ignore = ["B905"]          # zip() without strict= is fine for equal-length inputs here

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra"
markers = [
  "batch01: data foundation (config, contracts, storage, Stage 00, G2)",
  "batch02: universe and baselines (Stage 01, G1)",
  "batch03: validation framework (folds, purging, scoring, G3)",
  "batch04: forecast layers (Stage 02 models and contract)",
  "batch05: risk, optimisation and sizing (Stages 03-05, G4-G6)",
  "batch06: execution simulation (Stage 06)",
  "batch07: safety, orchestration, reporting and paper trading",
  "batch08: challenger lifecycle (G7) and extension layers",
  "invariant: design rules that must hold for every release",
]
```

Things to notice:

- **Lower bounds, not pins.** `pandas>=2.1` means "tested from 2.1 upwards". The reference code was tested with the oldest bounds (pandas 2.1, numpy 1.26, scipy 1.11, scikit-learn 1.3 on Python 3.11) and with current releases. When the project matures, freeze the exact set you validated with `python -m pip freeze > requirements-lock.txt` and record it in the run manifest.
- **`tzdata`** is easy to miss. Windows has no IANA time-zone database, so `ZoneInfo("Australia/Sydney")` fails without this package. Every decision time in AQP is time-zone aware, so the dependency is essential.
- **`[project.scripts] aqp = ...`** installs an `aqp.exe` command. `aqp run --as-of 2026-10-16` and `python -m aqp_python.cli run --as-of 2026-10-16` are the same thing.
- **pytest markers** `batch01` to `batch07` mirror the roadmap phases in design §17, so `python -m pytest -m batch03` runs exactly the validation-framework tests. `batch08` covers the challenger lifecycle and the extension layers (Part 12). `invariant` marks the design rules that must hold in every release.

The two requirements files mirror `pyproject.toml` for people who prefer `pip install -r`:

**File:** `requirements.txt`

```text
# Runtime dependencies (mirror of pyproject.toml [project].dependencies)
numpy>=1.26
pandas>=2.1
scipy>=1.11
scikit-learn>=1.3
PyYAML>=6.0
matplotlib>=3.8
tzdata>=2024.1
```

**File:** `requirements-dev.txt`

```text
# Developer tools (mirror of pyproject.toml [project.optional-dependencies].dev)
-r requirements.txt
pytest>=8.0
import-linter>=2.0
ruff>=0.5
```

`.gitignore` keeps secrets, licensed data and generated output out of Git:

**File:** `.gitignore`

```gitignore
# Python and tooling
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.import_linter_cache/
*.egg-info/
build/

# Secrets
.env
.env.*
!.env.example

# Data that is licensed, large or generated
data/raw/
data/vintages/
data/interim/
data/processed/
data/universes/
models/
outputs/
```

`data/universes/` and `outputs/` are generated by runs. `data/samples/` is **not** ignored: the synthetic sample data is a small, reproducible test fixture. Real provider data never goes in Git; provider licences usually forbid redistribution, and a successful connection does not change that.

**File:** `.env.example`

```bash
# Placeholder names only. Copy to .env locally; never commit real values.
LSEG_APP_KEY=
BROKER_API_KEY=
```

**File:** `README.md`

````markdown
# AQP-PYTHON

Advanced Quantitative Pipeline (design specification v12) - Python implementation.

Research models propose forecasts; the validated pipeline proposes target positions;
only independent risk and execution controls authorise orders.

```powershell
.\.venv\Scripts\Activate.ps1
python -m aqp_python.cli validate-config
python -m aqp_python.cli build-sample-data
python -m aqp_python.cli run --as-of 2026-10-16
python -m pytest -q
```

Every run writes an immutable folder `outputs/runs/<run_id>/` with a manifest, the
configuration snapshot, every stage's artifacts, plots and a report. Sample data is
synthetic. A passing run is not evidence of profitability or live readiness.
````

## 1.6 The package itself: three small files

**File:** `src/aqp_python/__init__.py`

```python
"""AQP - Advanced Quantitative Pipeline.

Importing this package must never open a data-provider session, read
credentials or contact a broker. Side effects belong in explicit commands.
"""

__all__ = ["__version__"]
__version__ = "0.1.0"
```

Importing the package must have **no side effects**. Pytest imports every module while collecting tests; if an import opened an LSEG session or a broker connection, merely listing the tests would touch a provider.

**File:** `src/aqp_python/version.py`

```python
"""Version and code-revision lookup used by run manifests."""
import subprocess
from pathlib import Path


def get_code_revision(root: Path) -> str:
    """Return the current Git commit, or 'unversioned' when Git is unavailable.

    A run manifest must say which code produced it. The fallback is explicit so
    a missing revision is visible rather than silently blank.
    """
    try:
        result = subprocess.run(
            ["git", "rev-parse", "--short", "HEAD"],
            cwd=root, capture_output=True, text=True, timeout=5, check=False,
        )
    except (OSError, subprocess.TimeoutExpired):
        return "unversioned"
    revision = result.stdout.strip()
    return revision if result.returncode == 0 and revision else "unversioned"
```

Every run manifest records which code produced it. `git rev-parse --short HEAD` returns the commit; if Git is not available, the function says `unversioned` explicitly rather than leaving the field blank. `check=False` and the timeout mean a Git problem can never crash a run.

**File:** `src/aqp_python/logging_config.py`

```python
"""One place to configure logging for commands and runs."""
import logging
from pathlib import Path

LOG_FORMAT = "%(asctime)s %(levelname)-7s %(name)s | %(message)s"


def configure_logging(level: str = "INFO", log_file: Path | None = None) -> None:
    numeric = getattr(logging, level.upper(), None)
    if not isinstance(numeric, int):
        raise ValueError(f"Unsupported log level: {level}")
    root = logging.getLogger()
    root.setLevel(numeric)
    # Remove handlers we added earlier so repeated calls never duplicate lines.
    for handler in list(root.handlers):
        if getattr(handler, "_aqp", False):
            root.removeHandler(handler)
            handler.close()
    handlers: list[logging.Handler] = [logging.StreamHandler()]
    if log_file is not None:
        log_file.parent.mkdir(parents=True, exist_ok=True)
        handlers.append(logging.FileHandler(log_file, encoding="utf-8"))
    for handler in handlers:
        handler.setFormatter(logging.Formatter(LOG_FORMAT))
        handler._aqp = True  # type: ignore[attr-defined]
        root.addHandler(handler)
    logging.getLogger("matplotlib").setLevel(logging.WARNING)   # silence font-cache chatter
```

Two details matter here. `_aqp = True` tags the handlers this function adds, so calling it twice (once from the CLI, once from the orchestrator) replaces them instead of printing every line twice. Passing `log_file=...` also writes the log into the run folder (`run.log`). Log the run ID, stage and reason codes, but never credentials or whole rows of licensed data.

Now install the package in **editable** mode. Editable means Python imports straight from `src\aqp_python`, so every edit is live without reinstalling.

```powershell
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -c "import aqp_python; print(aqp_python.__version__, aqp_python.__file__)"
```

Expected: `0.1.0 C:\...\AQP-PYTHON\src\aqp_python\__init__.py`. If the path points anywhere else (for example `site-packages`), you are not in editable mode. Run the install command again from the project root.

## 1.7 A script that tells you whether the environment is right

**File:** `scripts/validate_environment.py`

```python
"""Check that this terminal uses the project's .venv and that every dependency imports.

Run from the repository root:  python scripts/validate_environment.py
Exit code 0 = ready, 1 = wrong interpreter, 2 = missing project file or import.
"""
import importlib
import sys
from pathlib import Path

IMPORTS = {"numpy": "numpy", "pandas": "pandas", "scipy": "scipy", "scikit-learn": "sklearn",
           "PyYAML": "yaml", "matplotlib": "matplotlib", "tzdata": "tzdata", "pytest": "pytest",
           "aqp-python (editable)": "aqp_python"}


def main() -> int:
    executable = Path(sys.executable)
    print(f"python executable : {executable}")
    print(f"python version    : {sys.version.split()[0]}")
    print(f"working directory : {Path.cwd()}")
    status = 0
    if not Path("pyproject.toml").is_file():
        print("ERROR: pyproject.toml not found - open a terminal in the AQP-PYTHON folder")
        return 2
    # sys.prefix differs from sys.base_prefix only inside a virtual environment.
    if sys.prefix == sys.base_prefix or ".venv" not in Path(sys.prefix).parts:
        print("WARNING: not running from AQP-PYTHON\\.venv - activate it or use .venv\\Scripts\\python.exe")
        status = 1
    if sys.version_info < (3, 11):  # noqa: UP036 - this script may run on the wrong Python
        print("ERROR: Python 3.11 or newer is required")
        return 2
    for distribution, module in IMPORTS.items():
        try:
            imported = importlib.import_module(module)
            version = getattr(imported, "__version__", "ok")
            print(f"  ok   {distribution:<24} {version}")
        except ImportError as error:
            print(f"  FAIL {distribution:<24} {error}")
            status = 2
    print("READY" if status == 0 else "NOT READY")
    return status


if __name__ == "__main__":
    raise SystemExit(main())
```

```powershell
python .\scripts\validate_environment.py
```

```text
python executable : C:\Users\you\Documents\AQP-PYTHON\.venv\Scripts\python.exe
python version    : 3.12.7
working directory : C:\Users\you\Documents\AQP-PYTHON
  ok   numpy                    2.1.3
  ok   pandas                   2.2.3
  ...
  ok   aqp-python (editable)    0.1.0
READY
```

The exit code is the useful part: `0` means ready, `1` means the wrong interpreter, and `2` means something is missing. Note the **distribution name vs import name** difference the script handles: you `pip install scikit-learn` but `import sklearn`, and you `pip install PyYAML` but `import yaml`.

## 1.8 VS Code settings

**File:** `.vscode/settings.json`

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}\\.venv\\Scripts\\python.exe",
  "python.terminal.activateEnvironment": true,
  "python.testing.pytestEnabled": true,
  "python.testing.unittestEnabled": false,
  "python.testing.pytestArgs": ["tests"],
  "files.exclude": {"**/__pycache__": true, "**/.pytest_cache": true},
  "[python]": {"editor.rulers": [120]}
}
```

**File:** `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "AQP: research run (2026-10-16)",
      "type": "debugpy",
      "request": "launch",
      "module": "aqp_python.cli",
      "args": ["run", "--as-of", "2026-10-16"],
      "cwd": "${workspaceFolder}",
      "justMyCode": true
    },
    {
      "name": "AQP: debug current test file",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["${file}", "-x", "-s"],
      "cwd": "${workspaceFolder}",
      "justMyCode": false
    }
  ]
}
```

**File:** `.vscode/extensions.json`

```json
{"recommendations": ["ms-python.python", "ms-python.debugpy", "charliermarsh.ruff"]}
```

Then press **Ctrl+Shift+P → Python: Select Interpreter** and choose `.\.venv\Scripts\python.exe`. Open the **Testing** panel (flask icon). Once tests exist, they appear there and can be run or debugged one at a time. The two launch configurations let you set a breakpoint anywhere in the pipeline and press F5: one runs a research run, the other debugs whichever test file is open.

## 1.9 How testing works in this project (read once)

- pytest finds files named `test_*.py` under `tests\`, and functions named `test_*` inside them.
- `tests\conftest.py` holds shared **fixtures**: objects that tests receive by naming them as arguments, such as a temporary copy of the project.
- Tests never touch your real `data\`, `outputs\` or `configs\`. They work on copies in a temporary folder.
- Run narrow, then wide:

```powershell
python -m pytest tests\unit\test_contracts.py -q            # one file
python -m pytest tests\unit\test_contracts.py::test_worst_status_orders_blocked_over_degraded_over_pass -q
python -m pytest -m batch01 -q                              # one roadmap batch
python -m pytest -x --lf -q                                 # stop at first failure; rerun last failures
python -m pytest -q                                         # everything
```

Right now `python -m pytest` reports `no tests ran` (exit code 5). That is expected because no tests exist yet.

## 1.10 First commit

```powershell
git init
git add .
git commit -m "Project skeleton, packaging and environment check"
```

From now on, commit at the end of every part (each part ends with a suggested message). Small commits make it easy to find the change that broke a test.

## Checkpoint

- [ ] `python .\scripts\validate_environment.py` prints `READY`.
- [ ] `python -c "import aqp_python; print(aqp_python.__file__)"` points into `src\aqp_python`.
- [ ] VS Code's interpreter is `.venv\Scripts\python.exe`.
- [ ] `git log --oneline` shows one commit.

Next: [Part 02 — contracts, configuration and gates](02_contracts_configuration_and_gates.md).
