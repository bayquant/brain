---
tags: [python, tooling, uv, astral]
---
# UV Astral

- [Using uv with Jupyter](https://docs.astral.sh/uv/guides/integration/jupyter/)

There are two modes in UV.

---

## 1) Project mode (the important one)

uv treats the directory as a Python project with a `pyproject.toml`.

**Key commands:**
```bash
uv sync
uv run …
```

**What happens:**
- Resolves deps from `pyproject.toml`
- Locks them in `uv.lock`
- Creates/uses a project environment
- Installs your project itself (editable by default)
- Replaces `pip install -e .`

**Use when:** you're developing a Python project.

---

## 2) Environment / pip mode

uv behaves like a fast `pip + venv` replacement. No `[project]` table, no lockfile-driven workflow.

**Key commands:**
```bash
uv venv
uv pip install …
```

**What happens:**
- Just creates / mutates a virtualenv
- No project install
- No lockfile-driven workflow
- Need to run `pip install -e .` to make it editable

**Use when:** you just want a Python environment, not a project.

---

## Activating a virtual environment

```bash
uv venv                        # Create .venv
source .venv/bin/activate      # Activate (macOS/Linux)
deactivate                     # Deactivate
```

In project mode, `uv run` activates the environment automatically — no manual activation needed.

---

## dependency-groups vs optional-dependencies

`[dependency-groups]` (PEP 735) are for dependencies that are only ever used locally during development — they're not part of the package at all, not even as an optional install. Only uv knows about them.

Typical use case:
- `uv sync` — every developer gets pytest, linters, etc. automatically
- `pip install spread-sniper` — end users get only core deps, dev tools never mentioned

| | `[dependency-groups]` | `[project.optional-dependencies]` |
|---|---|---|
| Part of package spec | No | Yes |
| Installable via pip | No | Yes — `pip install pkg[dev]` |
| Visible to non-uv tools | No | Yes |
| Use case | Local dev tooling | Optional features for end users |
