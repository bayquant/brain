# Pre-commit and Ruff

## Pre-commit Framework

Pre-commit is a framework that runs hooks automatically before each git commit. It's configured via `.pre-commit-config.yaml`.

### Setup

**Install pre-commit hook (one-time per repo):**
```bash
uv run pre-commit install
```

This registers the git hook so it runs automatically before commits.

**Uninstall pre-commit hook:**
```bash
uv run pre-commit uninstall
```

### Running Pre-commit

**Run automatically on commit:**
```bash
git commit -m "message"
# Hook runs automatically, blocks commit if files are modified
```

**Run manually on all files:**
```bash
uv run pre-commit run --all-files
```

**Run manually on staged files only:**
```bash
uv run pre-commit run
```

### How Pre-commit Works

1. You attempt to commit
2. Pre-commit hook runs defined checks
3. If files are modified by hooks, commit is **blocked**
4. You stage the modified files and commit again
5. Hook passes (nothing to fix), commit succeeds

### Workflow Example

```bash
# Make changes
git add .

# Try to commit
git commit -m "my changes"
# ❌ Pre-commit runs, ruff formats files, commit blocked

# Stage the formatted files
git add .

# Commit again
git commit -m "my changes"
# ✅ Commit succeeds (hook finds nothing to fix)
```

---

## Ruff - Python Linter & Formatter

Ruff is a fast Python linter and code formatter written in Rust. It combines the functionality of black, isort, and many linting tools into one.

### Installation

Ruff is declared in `pyproject.toml` under dev dependencies:

```toml
[dependency-groups]
dev = [
    "ruff>=0.6",
    "pre-commit>=4.4.0",
    # ...
]
```

Install with:
```bash
uv sync --group dev
```

### Configuration

**pyproject.toml:**
```toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "W", "I"]  # Rules to enforce
ignore = ["E501"]              # Ignore line-too-long

[tool.ruff.lint.isort]
known-first-party = ["package1", "package2"]  # Your project packages
```

**Rule legend:**
- `E` - pycodestyle errors (PEP 8 style violations)
- `F` - Pyflakes errors (undefined names, unused imports)
- `W` - pycodestyle warnings
- `I` - isort (import sorting)
- `E501` - line too long (ignored; handled by formatter)

### Commands

**Check and fix linting issues (imports, unused code, etc.):**
```bash
uv run ruff check --fix .
```

Fixes:
- Sorts imports by section (stdlib → third-party → first-party → local)
- Removes unused imports
- Fixes PEP 8 violations (except line length)

**Format code (whitespace, line wrapping, quotes):**
```bash
uv run ruff format .
```

Fixes:
- Indentation, trailing commas
- Line length (respects `line-length = 88`)
- Quote style normalization

**Run both (recommended):**
```bash
uv run ruff check --fix . && uv run ruff format .
```

**Check specific file:**
```bash
uv run ruff check path/to/file.py
uv run ruff format path/to/file.py
```

### Import Organization

Ruff organizes imports into 4 sections (with blank lines between):

```python
# Standard library imports
import os
import sys

# Third-party imports
import numpy
import pandas

# First-party imports
from myproject.service import MyService

# Local imports
from .module import function
```

**Import style:**
- **First-party** (different packages): Use absolute imports
  - `from spread_sniper import ...`
  - `from mail.service import ...`
- **Local** (same package): Use relative imports
  - `from .module import ...`
  - `from ..sibling import ...`

### Pre-commit Integration

**.pre-commit-config.yaml:**
```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: Lint with ruff
        entry: uv run ruff check --fix
        language: system
        types: [python]
        stages: [pre-commit]
      - id: ruff-format
        name: Format with ruff
        entry: uv run ruff format
        language: system
        types: [python]
        stages: [pre-commit]
```

This runs both checks and formatting automatically before commits.

### Common Fixes

**Consolidated imports:**
```python
# Before (multiple single imports)
from module import func1
from module import func2
from module import func3

# After (grouped)
from module import func1, func2, func3
```

**Sorted imports:**
```python
# Before (unsorted)
from package import z_func
from package import a_func

# After (sorted alphabetically)
from package import a_func, z_func
```

**Unused imports removed:**
```python
# Before
import os  # unused
from module import func

# After
from module import func
```

### Tips

- Run ruff check and format before committing manually if you want to review changes first
- Use `uv run pre-commit run --all-files` to reformat all files in a repo
- Ignore `E501` (line too long) because the formatter handles it better than the linter
- Configure `known-first-party` in pyproject.toml so ruff knows which packages are yours
