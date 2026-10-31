---
tags: [python, modules, packages, cli]
---
# Python Modules and Entry Points

How Python finds code, how it starts it, and what `__init__`, `__main__`, and `__package__` mean. Related: [[UV Astral]], [[Terminal]].

---

## FINDING CODE: SYS.PATH

`import x` searches the folders in `sys.path`, in order. If the folder that *contains* `x` is not on it, you get `No module named x`.

| Source | Example |
|---|---|
| Current directory (`-m`, `-c`) | running from the repo root |
| Script's own folder (`python file.py`) | `sys.path[0]` is that file's folder |
| Installed packages | `uv sync` links the project into `.venv` |
| `PYTHONPATH` | manual override |

---

## WHAT A FOLDER IS: `__INIT__.PY`

| Folder | Kind | Behavior |
|---|---|---|
| Has `__init__.py` | regular package | one owner of the name; the file runs on import |
| No `__init__.py` | namespace package (PEP 420) | still importable, but same-named folders on `sys.path` merge silently |

`sys.path` decides *where* Python looks. `__init__.py` decides *what* it found. They are independent.

---

## STARTING CODE: THREE WAYS

| Command | Runs | `__package__` |
|---|---|---|
| `python file.py` | the file at that path | empty |
| `python -m pkg.mod` | the module found by import name | `"pkg"` |
| `python -c "code"` | the string | empty |

- `-m pkg` imports `pkg` normally, then runs `pkg/__main__.py`.
- Relative imports (`from .x import y`) need `__package__`, so they work under `-m` and fail when the file is run by path:

```
ImportError: attempted relative import with no known parent package
```

---

## `__NAME__` AND THE GUARD

`__name__` is `"__main__"` for the file Python *started* (the entry point), by path or by `-m`. For an imported module it is the dotted import name.

```python
if __name__ == "__main__":
    raise SystemExit(main())
```

The guard keeps `main()` from running when the file is merely imported (tests, tooling, `python -c "import pkg.__main__"`).

| How the file is loaded | `__name__` | Guard runs `main()`? |
|---|---|---|
| `python -m pkg` | `"__main__"` | yes |
| `import pkg.__main__` | `"pkg.__main__"` | no |

---

## SYSTEMEXIT

An exception that ends the program quietly, with no traceback.

| Raised with | Exit code |
|---|---|
| `0` or `None` | 0 (success) |
| an int | that int |
| a string | prints it to stderr, code 1 |

- `raise SystemExit(main())` hands `main()`'s return value to the shell.
- argparse raises `SystemExit(2)` on bad arguments.
- It inherits from `BaseException`, so `except Exception:` does not swallow it.

---

## LAYOUT PATTERN FOR A CLI PACKAGE

```
pkg/
  __init__.py      # regular package marker (can be empty)
  __main__.py      # `python -m pkg`: calls main(), guarded
  main.py          # build_parser() and main()
```

- The console script (`[project.scripts]`) and `python -m pkg` are two doors to the same `main()`.
- Parents know their children; children never import their parent.

---

## QUICK CHECKS

```bash
python -c "import pkg; print(pkg.__file__)"                # where it is imported from
python -c "import pkg.__main__ as m; print(m.__name__)"    # pkg.__main__ (imported)
python -m pkg                                              # __name__ is __main__ (entry)
echo $?                                                    # exit code of the last command
```
