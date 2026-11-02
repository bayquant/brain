---
tags: [python, decorators, closures]
---
# Python Decorators

A decorator is a function that takes a function and returns a replacement for it. Decorators with arguments need one extra layer, which is where most of the confusion comes from. Related: [[Python Modules and Entry Points]], [[Caching]].

---

## PLAIN DECORATOR

```python
@my_decorator
def f(x): ...
```

is shorthand for `f = my_decorator(f)`. Two layers are enough:

```python
def my_decorator(func):  # receives the function
    def wrapper(*args, **kwargs):  # receives each call
        return func(*args, **kwargs)

    return wrapper
```

---

## DECORATOR WITH ARGUMENTS

```python
@range_cache(start="start", end="end", key=["symbol"])
def f(...): ...
```

is shorthand for `f = range_cache(start="start", end="end", key=["symbol"])(f)`. The call happens in two steps:

1. `range_cache(start=..., end=..., key=...)` runs first and must return a decorator.
2. That decorator is called with `f` and returns the wrapper.

So there are three layers:

```python
def range_cache(start, end, key):  # 1. receives the settings
    def decorator(func):  # 2. receives the function
        def wrapper(*args, **kwargs):  # 3. receives each call
            ...

        return wrapper

    return decorator
```

| Layer | Runs | Receives | Returns |
|---|---|---|---|
| `range_cache` | once, at the `@` line | `start`, `end`, `key` | `decorator` |
| `decorator` | once, right after | the function | `wrapper` |
| `wrapper` | every call to the function | the call's arguments | the result |

---

## DECORATOR FACTORY

The outer layer is called a decorator factory: a function whose job is to build and return another function from the settings it is given. `range_cache` does no caching itself. It returns a configured `decorator`.

```python
by_symbol = range_cache(start="start", end="end", key=["symbol"])
by_kind = range_cache(start="start", end="end", key=["kind", "symbol"])
```

These are two separate decorators, each remembering its own settings through a closure. Applying one to a function is `@by_symbol`.

| Layer | Role |
|---|---|
| `range_cache` | decorator factory (makes decorators) |
| `decorator` | the decorator (makes wrappers) |
| `wrapper` | the replacement function |

"Factory" is a design-pattern name, not Python syntax. Because settings passed to the factory are captured in the closure, it is also the natural place to add options later, such as a storage backend.

---

## CLOSURES

The inner layers can read the outer layers' variables. `wrapper` uses `start`, `end`, `key`, `signature`, and `func_dir` even though they were defined in the layers above.

Work that depends only on the settings or the function (inspecting the signature, validating parameter names, computing paths) belongs in `decorator`. It runs once at import time, so mistakes fail early. Work that depends on the arguments of a call belongs in `wrapper`.

---

## FUNCTOOLS.WRAPS

`@functools.wraps(func)` on the wrapper copies the original function's `__name__`, `__doc__`, and signature metadata. Without it, `fetch.__name__` would say `wrapper`, which breaks introspection and makes tracebacks confusing.

---

## RELATED

- [[Cache Manifest]]: the range cache decorator keeps one manifest per decorated function.
