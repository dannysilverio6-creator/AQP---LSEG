# Part 01 — Windows, PowerShell, VS Code and the project skeleton

**Goal:** an `AQP-PYTHON` folder that VS Code, PowerShell, pip and pytest all agree on. That means one Python interpreter (the project's `.venv`), the full layout-v3 folder tree, an installable package called `aqp_python`, and a script that tells you when something is wrong.

**Repository overview:** §3 (the tree), §13 (Windows and VS Code setup), §14 (Git hygiene).

Most "it worked yesterday" problems on Windows come from two Pythons: pip installs a library into one while VS Code or pytest runs the other. Everything in this part exists to prevent that.

## What you will build in this part

| Step | File or command | Components |
|---|---|---|
| 1.4 | the layout-v3 folder tree | `build_repository_shell.py --create` |
| 1.5 | `pyproject.toml`, `requirements*.txt`, `.gitignore`, `.env.example`, `README.md` | 2 + 1 + 1 + 2 + 1 + 1 |
| 1.6 | `src/aqp_python/__init__.py`, `version.py`, `logging_config.py` | 1 + 1 + 2 |
| 1.7 | `scripts/validate_environment.py` | 2 |
| 1.8 | `.vscode/settings.json`, `launch.json`, `extensions.json` | 1 + 1 + 1 |

---

## 1.1 Install the tools once

| Tool | Why | Check in PowerShell |
|---|---|---|
| Python 3.11 or newer (python.org installer, tick **"Add python.exe to PATH"** and keep the **py launcher**) | Runs the pipeline. 3.11 is the minimum because the code uses `enum.StrEnum` and `datetime.UTC`. | `py --version` and `py -0p` (lists every installed Python) |
| Git | Small, reversible steps and a history of decisions | `git --version` |
| VS Code + the **Python**, **Python Debugger** and **Ruff** extensions | Editor, test explorer, debugger | `code --version` |

If `py` is not recognised, reinstall Python with the launcher option, then close and reopen the terminal. Do not work around it by mixing interpreters.

## 1.2 How this guide is laid out (read once)

Every Python file is shown as a series of **components**. A caption tells you exactly where each one goes:

> **File:** `src/aqp_python/storage/vintage_store.py` — component 2 of 2: latest_vintage

- Type the components of a file **in order, into the same file**, one under the other. Component 1 always starts the file (docstring and imports).
- A component that begins indented (for example the second half of a long function) continues the code above it. Keep its indentation exactly.
- After each component you get three things: **what it does**, **why it is written that way**, and often a **Try it** box: a few lines you paste into a scratch file and run, with the exact output you should see. Running them is the fastest way to understand a component before you build on it.
- A few files are shown whole because they are short.

Try-it snippets are run as scripts from the project root. Save them in `build\scratch.py` (the `build` folder is ignored by Git) and run `python build\scratch.py`.

## 1.3 Create the folder and the virtual environment

```powershell
Set-Location $HOME\Documents          # or wherever you keep projects
```

You received `build_repository_shell.py` with the repository overview. Copy it into this folder, then let it create the whole layout-v3 tree:

```powershell
py -3 build_repository_shell.py --counts
py -3 build_repository_shell.py --create .
Set-Location .\AQP-PYTHON
New-Item -ItemType Directory -Force scripts | Out-Null
Move-Item ..\build_repository_shell.py .\scripts\
code .
```

`--counts` prints what will be created, and `--create .` creates `AQP-PYTHON` in the current folder. The script needs nothing but the standard library, so it runs before any virtual environment exists.

In VS Code open **Terminal → New Terminal** and make sure the dropdown shows **PowerShell**. The prompt should end in `AQP-PYTHON>`. Every later command runs from here.

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

## 1.4 What the shell gave you, and the three things it does not create

The shell follows the overview's convention: folders, `__init__.py` package markers and **zero-byte placeholders** for configuration and documentation. It does **not** create the planned Python modules; you write those, one component at a time, over the next twelve parts. When a part says "File: `configs/gates.yaml`", you are filling in a placeholder that already exists.

Three things the guide needs are not in the layout, so create them now:

```powershell
New-Item -ItemType File -Force tests\__init__.py | Out-Null
New-Item -ItemType Directory -Force src\aqp_python\connectors\synthetic | Out-Null
New-Item -ItemType File -Force src\aqp_python\connectors\synthetic\__init__.py | Out-Null
(Get-ChildItem src -Recurse -Filter __init__.py).Count
```

The last line should print `38` (the 37 package markers from the shell plus the new `synthetic` package).

- `tests\__init__.py` lets tests import shared helpers with `from tests.conftest import ...`.
- `connectors\synthetic\` holds the synthetic market generator (Part 04). It is a stand-in for a data provider, so it lives with the connectors. Appendix B lists every module the guide adds to the overview's layout.

Copy the design into `docs\design\` now: `AQP_OVERVIEW.md` (the frozen v14 specification) and the eleven stage files into `docs\design\stages\`. When this guide says "design Stage 0" or "design 2.2", it means those documents.

**Why `src\aqp_python` and not just `aqp_python`?** With the "src layout", Python can import your package only after it is *installed* (1.6). Tests therefore exercise the same import path the CLI will use, and a stray file in the project root can never shadow the real package.

## 1.5 Packaging and housekeeping files

`pyproject.toml` is the single source of truth for the package name, Python version, dependencies, the `aqp` command, and the pytest and ruff settings.

**File:** `pyproject.toml` — component 1 of 2: build system and project metadata

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "aqp-python"
version = "0.2.0"
description = "AQP - Advanced Quantitative Pipeline (design v14, research and paper-trading build)"
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
```

**What it does.** Declares the package `aqp-python` (imported as `aqp_python`), the minimum Python version, the seven runtime libraries and the developer tools, and installs a console command `aqp` that calls `aqp_python.cli:main`.

**Why it is written this way.**

- **Lower bounds, not pins.** `pandas>=2.1` means "tested from 2.1 upwards". When the project matures, freeze the exact set you validated with `python -m pip freeze > requirements-lock.txt` and record it in the run manifest.
- **`tzdata`** is easy to miss. Windows has no IANA time-zone database, so `ZoneInfo("Australia/Sydney")` fails without this package. Every decision time in AQP is time-zone aware, so the dependency is essential.
- `[tool.setuptools.packages.find] where = ["src"]` is what makes the src layout work.

**File:** `pyproject.toml` — component 2 of 2: tool settings: ruff and pytest markers

```toml
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
  "batch01: phase 1 screening foundation (contracts, config, storage, data_foundation, Stage 00, G2)",
  "batch02: phase 2 universe formation (Stage 01, G1, coverage tiers)",
  "batch03: phase 3 post-filter data expansion (data request, company reference, X_core)",
  "batch04: phase 4 no-news baselines (validation framework, forecast contract, models)",
  "batch05: phase 5 AI-NLP news feature engine (Stage 02D, Z_news)",
  "batch06: phase 6 paired challenger evaluation (Stage 02E, G3, combination)",
  "batch07: phase 7 risk, optimisation and sizing (Stages 03-05, G4-G6)",
  "batch08: phase 8 execution simulation and hard controls (Stage 06, safety)",
  "batch09: phase 9 monitoring, reporting, full runs and controlled release",
  "extension: Part 14 extension modules that are not wired into the default run",
  "invariant: design rules that must hold for every release",
]
```

**What it does.** Configures the linter (ruff) and registers the pytest **markers**.

**Why it is written this way.** The markers `batch01` … `batch09` are the nine phases of the design's implementation roadmap (design document 09), in order: screening foundation, universe formation, data expansion, no-news baselines, the news feature engine, paired challenger evaluation, risk/optimisation/sizing, execution and hard controls, monitoring and release. `python -m pytest -m batch05` runs exactly the news-engine tests. `invariant` marks the design rules that must hold in every release, and `extension` marks the Part 14 modules that are not part of the default run. The overview's `tests\test_batches\` folders each get a README later that names the marker to run.

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

**File:** `.gitignore` — component 1 of 2: Python, tooling and secrets

```gitignore
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.import_linter_cache/
*.egg-info/
build/
.env
.env.*
!.env.example
```

**File:** `.gitignore` — component 2 of 2: data that is licensed, large or generated

```gitignore
data/raw/
data/vintages/
data/interim/
data/processed/
data/text_corpus/
data/features/
data/universes/
data/data_requests/
data/screening/
models/
outputs/
```

**Why these folders.** `data\samples\` is **not** ignored: the synthetic sample data is a small, reproducible test fixture. Everything the pipeline generates (universes, data requests, feature stores, the text corpus, run folders) is ignored. Real provider data and licensed news never go in Git: provider licences usually forbid redistribution, and a successful connection does not change that (overview §14).

**File:** `.env.example`

```bash
# Placeholder names only. Copy to .env locally; never commit real values.
LSEG_APP_KEY=
NEWS_API_KEY=
LLM_API_KEY=
BROKER_API_KEY=
```

Placeholder names only. The guide never reads these; Part 14 explains where a news or LLM key would be used.

**File:** `README.md`

````markdown
# AQP-PYTHON

Advanced Quantitative Pipeline (design v14, repository layout v3) - Python implementation.

Screening decides what is researched; research models propose forecasts; the news engine
supplies covariates only; the validated pipeline proposes target positions; only independent
risk and execution controls authorise orders.

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

The README states the project's one-sentence rule. Every part of this guide is a way of keeping that sentence true.

## 1.6 The package itself: three small files

**File:** `src/aqp_python/__init__.py`

```python
"""AQP - Advanced Quantitative Pipeline (design v14).

Importing this package must never open a data-provider session, read
credentials, call a language model or contact a broker. Side effects belong
in explicit commands.
"""

__all__ = ["__version__"]
__version__ = "0.2.0"
```

**Why no side effects.** Pytest imports every module while collecting tests. If an import opened an LSEG session, called a language model or contacted a broker, merely listing the tests would touch a provider.

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

**What it does.** Returns the current Git commit, or `unversioned`. **Why:** every run manifest records which code produced it. `check=False` and the timeout mean a Git problem can never crash a run, and the fallback is explicit, so a missing revision is visible rather than blank.

**File:** `src/aqp_python/logging_config.py` — component 1 of 2: imports and the log format

```python
"""One place to configure logging for commands and runs."""
import logging
from pathlib import Path

LOG_FORMAT = "%(asctime)s %(levelname)-7s %(name)s | %(message)s"
```

**File:** `src/aqp_python/logging_config.py` — component 2 of 2: configure_logging

```python
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

**What it does.** Sends log lines to the terminal and, for a run, also to `run.log` inside the run folder.

**Why it is written this way.** `_aqp = True` tags the handlers this function adds, so calling it twice (once from the CLI, once from the orchestrator) replaces them instead of printing every line twice. Log the run ID, stage and reason codes, but never credentials, story text or whole rows of licensed data.

Now install the package in **editable** mode. Editable means Python imports straight from `src\aqp_python`, so every edit is live without reinstalling.

```powershell
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -c "import aqp_python; print(aqp_python.__version__, aqp_python.__file__)"
```

Expected: `0.2.0 C:\...\AQP-PYTHON\src\aqp_python\__init__.py`. If the path points anywhere else (for example `site-packages`), you are not in editable mode; run the install command again from the project root.

## 1.7 A script that tells you whether the environment is right

**File:** `scripts/validate_environment.py` — component 1 of 2: docstring, imports and the import list

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
```

**What it does.** Names the eight imports the project depends on, as *distribution name → import name*. You `pip install scikit-learn` but `import sklearn`, and you `pip install PyYAML` but `import yaml`; the dictionary records both.

**File:** `scripts/validate_environment.py` — component 2 of 2: main: interpreter, version and import checks

```python
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

**What it does.** Prints the interpreter, version and folder, then tries each import. **Why the checks are in this order:** if you are in the wrong folder nothing else matters (exit 2); then the wrong interpreter (exit 1); then each import. `sys.prefix != sys.base_prefix` is true only inside a virtual environment, and checking for `.venv` in the path catches "a virtual environment, but the wrong one".

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
  ok   aqp-python (editable)    0.2.0
READY
```

The exit code is the useful part: `0` ready, `1` wrong interpreter, `2` something missing.

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

Press **Ctrl+Shift+P → Python: Select Interpreter** and choose `.\.venv\Scripts\python.exe`. Open the **Testing** panel (flask icon); once tests exist they appear there and can be run or debugged one at a time. The two launch configurations let you set a breakpoint anywhere and press F5: one runs a research run, the other debugs whichever test file is open.

## 1.9 How testing works in this project (read once)

- pytest finds files named `test_*.py` under `tests\`, and functions named `test_*` inside them.
- `tests\conftest.py` holds shared **fixtures**: objects that tests receive by naming them as arguments, such as a temporary copy of the project.
- Tests never touch your real `data\`, `outputs\` or `configs\`. They work on copies in a temporary folder.
- Run narrow, then wide:

```powershell
python -m pytest tests\unit\test_quantile_grid.py -q          # one file
python -m pytest tests\unit\test_config_loader.py::test_missing_limit_fails_closed -q
python -m pytest -m batch01 -q                                # one roadmap phase
python -m pytest -x --lf -q                                   # stop at first failure; rerun last failures
python -m pytest -q                                           # everything
```

Right now `python -m pytest` reports `no tests ran` (exit code 5). That is expected.

## 1.10 First commit

```powershell
git init
git add .
git commit -m "Layout v3 shell, packaging and environment check"
```

From now on, commit at the end of every part. Small commits make it easy to find the change that broke a test.

## Checkpoint

- [ ] `python .\scripts\validate_environment.py` prints `READY`.
- [ ] `python -c "import aqp_python; print(aqp_python.__file__)"` points into `src\aqp_python`.
- [ ] `(Get-ChildItem src -Recurse -Filter __init__.py).Count` prints 38.
- [ ] VS Code's interpreter is `.venv\Scripts\python.exe`.

**Check your understanding.** (1) Why does the shell create `__init__.py` files but not the modules? (2) Which two commands would you run if `import sklearn` failed in VS Code but worked in PowerShell?

Next: [Part 02 — contracts, configuration and gates](02_contracts_configuration_and_gates.md).
