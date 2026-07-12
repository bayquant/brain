---
tags: [logging, observability, python]
---
# Logging

Logging records structured messages about what a program is doing, so its behavior can be observed after the fact without a debugger. The central design idea is to **separate emitting a message from deciding what to do with it** — application code emits; configuration, set once at startup, decides whether it shows, where it goes, and how it looks.

---

## CORE IDEA

Naive logging (e.g. `print`) fuses two jobs: the code decides to record *and* decides where it goes, immediately. Real logging splits them:

```
code that EMITS  ─────►  a message + a severity        (does not know or care where it ends up)
                            │
config (once, at startup) ─►  decides: shown? where? what format?
```

This split is what lets a library emit messages without dictating output to the application that uses it — the application owns the configuration.

---

## THE FOUR PIECES

```
your code → Logger → [level check] → Handler(s) → Formatter → output
```

- **Logger** — what you call (`logger.info(...)`). Named, and requested (not constructed): `logging.getLogger("name")` returns the same object every time.
- **Level** — a severity threshold on the logger; messages below it are dropped.
- **Handler** — a destination (console/stderr, file, network). Attached to loggers. No handler anywhere → nothing is emitted.
- **Formatter** — how a record is rendered to text (layout, timestamps, color). Attached to a handler.

---

## LEVELS

| Level | Numeric | Use for |
|---|---|---|
| DEBUG | 10 | diagnostic detail, off in production |
| INFO | 20 | normal milestones ("downloaded X") |
| WARNING | 30 | unexpected but handled |
| ERROR | 40 | an operation failed |
| CRITICAL | 50 | the program may not continue |

A logger set to `WARNING` silently drops `info`/`debug`. Emitting an expected-but-noisy event at `DEBUG` means it's invisible by default and only appears when someone lowers the threshold — the standard way to keep chatter available without it being on all the time.

---

## THE LOGGER HIERARCHY

Logger names form a tree, split on dots:

```
""  (root)
└── "myapp"
    ├── "myapp.storage"
    └── "myapp.network"
```

A record emitted on a child **propagates up** to its ancestors' handlers. Two consequences:

- Attach **one** handler to `"myapp"` and every `myapp.*` logger flows into it.
- Set the level on `"myapp"` once and it governs all children — so verbosity is tunable **per subsystem** from the outside.

Convention: each module does `logger = logging.getLogger(__name__)`, giving names like `myapp.storage.s3` that mirror the package layout.

---

## CONFIGURING OUTPUT

Configuration is **application code, run once at startup** (top of a script, first cell of a notebook, in `main()`). The 90% case:

```python
import logging

logging.basicConfig(level=logging.INFO)   # adds a handler to root, sets format + level
```

Without this, output is silent. `basicConfig` bundles handler + default format + root level into one call. Turning the knobs:

```python
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%H:%M:%S",
    # filename="run.log",   # send to a file instead of the console
)
```

Tune each subsystem independently (the hierarchy payoff):

```python
logging.basicConfig(level=logging.WARNING)               # quiet by default
logging.getLogger("myapp").setLevel(logging.DEBUG)       # but this app: verbose
logging.getLogger("botocore").setLevel(logging.WARNING)  # silence a chatty dependency
```

`basicConfig` is just a shortcut for building a handler + formatter and attaching it to the root logger. For complex setups there is `logging.config.dictConfig({...})`, but `basicConfig` plus a few `setLevel` calls covers most needs.

---

## LIBRARY VS APPLICATION

The single most important rule: **a library must not configure logging.** It should not call `basicConfig`, add stream handlers, or touch the root logger — that forces output and format on every consumer.

A library only:

- Emits under named loggers: `logging.getLogger(__name__)`.
- Attaches a **`NullHandler`** to its top-level logger so an unconfigured import is silent (no output, no "no handlers could be found" warning).

```python
# in the library, once:
logging.getLogger("myapp").addHandler(logging.NullHandler())
```

The **application** then opts in with `basicConfig`/`dictConfig`. Because everything (the library, plus dependencies like boto3) logs through the same stdlib machinery, one configuration controls them all — no parallel logging systems to configure separately.

---

## LAZY FORMATTING

Pass arguments to the logging call rather than pre-formatting the string:

```python
logger.info("downloaded %s", key)     # good — %s filled in only if the record is emitted
logger.info(f"downloaded {key}")      # wasteful — f-string built every call, even if suppressed
```

With `%`-style args the interpolation happens **only if** the level passes and a handler exists, so a suppressed `debug` call costs almost nothing. It also keeps the raw template intact, which log aggregators can group on. (Python's stdlib logging uses `%`-style, not `str.format`/f-strings, in the call itself.)

---

## COMMON PITFALLS

- **Configuring logging inside a library** — calling `basicConfig` or adding handlers forces output/format on every consumer. Emit only; attach a `NullHandler`.
- **Not setting a level** — the root logger defaults to `WARNING`, so `INFO`/`DEBUG` are silently dropped. The usual "why don't I see my logs?" cause.
- **f-strings / `.format` in log calls** — always pays the formatting cost even when the message is suppressed, and loses the reusable template.
- **Logging to stdout** — pollutes real program output (and clutters notebook cells). Logs belong on stderr.
- **Duplicate lines** — a handler on both a child logger and root (via propagation) emits twice; attach at one level, or set `propagate = False`.
- **`print` instead of logging** — no levels, no routing, no way for a consumer to turn it down.
- **Logging secrets / PII** — messages persist and get shipped to aggregators; never log credentials or personal data.
