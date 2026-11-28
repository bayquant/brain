---
tags: [make, makefile, build, automation, devops, cli]
---
# Makefiles

`make` is a build automation tool. A `Makefile` declares **targets**, the **prerequisites** each target depends on, and the **recipe** (shell commands) that produces the target. `make` rebuilds a target only when a prerequisite is newer than it, which makes it useful for compiling code and, more commonly today, as a task runner for project commands.

---

## ANATOMY OF A RULE

```make
target: prerequisite1 prerequisite2
	recipe command 1
	recipe command 2
```

| Part | Meaning |
|---|---|
| target        | File to produce, or a name for an action |
| prerequisites | Targets or files that must be up to date first |
| recipe        | Shell commands, **each line must start with a TAB** |

Running `make target` checks prerequisites recursively, then runs the recipe if the target is missing or older than any prerequisite. Running `make` with no argument builds the **first** target in the file (the default goal).

---

## PHONY TARGETS

A target that is not a file (e.g. `test`, `clean`) must be declared `.PHONY`. Otherwise a file named `test` in the directory would make `make test` think it is up to date.

```make
.PHONY: test clean

test:
	pytest

clean:
	rm -rf build dist .pytest_cache
```

---

## VARIABLES

```make
PYTHON := python3
SRC_DIR := src

run:
	$(PYTHON) $(SRC_DIR)/main.py
```

### ASSIGNMENT OPERATORS

| Operator | Behavior |
|---|---|
| `:=` | Simple: expanded once, at definition (preferred) |
| `=`  | Recursive: expanded each time it is used |
| `?=` | Set only if not already defined (allows env or CLI override) |
| `+=` | Append |

Override from the command line: `make run PYTHON=python3.12`.

### AUTOMATIC VARIABLES

| Variable | Expands to |
|---|---|
| `$@` | The target name |
| `$<` | The first prerequisite |
| `$^` | All prerequisites (deduplicated) |
| `$*` | The stem matched by a `%` pattern |

---

## PATTERN RULES

`%` matches any non-empty string, so one rule covers many files.

```make
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

Builds any `foo.o` from `foo.c`.

---

## EXAMPLE: C PROJECT

```make
CC := gcc
CFLAGS := -Wall -O2
OBJS := main.o util.o

app: $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

.PHONY: clean
clean:
	rm -f app $(OBJS)
```

Editing only `util.c` recompiles `util.o` and relinks `app`; `main.o` is reused.

---

## EXAMPLE: PYTHON PROJECT (TASK RUNNER)

```make
.DEFAULT_GOAL := help
.PHONY: help install lint test clean

help:  ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "} {printf "%-12s %s\n", $$1, $$2}'

install:  ## Create venv and install dependencies
	uv venv
	uv pip install -r requirements.txt

lint:  ## Run ruff
	uv run ruff check .

test:  ## Run tests
	uv run pytest

clean:  ## Remove caches
	rm -rf .pytest_cache .ruff_cache __pycache__
```

The `## comment` convention lets `make help` list every target automatically.

---

## USEFUL FEATURES

### SILENCING AND ERRORS

```make
build:
	@echo "building"      # @ hides the command itself
	-rm missing.txt       # - ignores a non-zero exit status
```

### SHELL STATE

Each recipe line runs in its **own shell**, so `cd` does not persist between lines. Chain them or use a backslash:

```make
deploy:
	cd app && ./deploy.sh

deploy-multi:
	cd app; \
	./deploy.sh
```

### ESCAPING `$`

Make expands `$(...)` itself. To pass a shell variable, double the dollar sign:

```make
show:
	@for f in *.txt; do echo $$f; done
```

### FUNCTIONS

```make
SRCS := $(wildcard src/*.c)
OBJS := $(patsubst src/%.c,build/%.o,$(SRCS))
```

### CONDITIONALS

```make
ifeq ($(ENV),prod)
  FLAGS := -O3
else
  FLAGS := -g
endif
```

### INCLUDING FILES

```make
-include .env.mk   # leading - ignores the file if it is missing
```

### ORDER-ONLY PREREQUISITES

Prerequisites after `|` must exist but do not trigger a rebuild when they change. Useful for output directories.

```make
build/%.o: src/%.c | build
	$(CC) -c $< -o $@

build:
	mkdir -p build
```

---

## COMMON FLAGS

| Command | Effect |
|---|---|
| `make -n`      | Dry run: print recipes without running them |
| `make -j4`     | Run up to 4 jobs in parallel |
| `make -B`      | Force rebuild of all targets |
| `make -C dir`  | Run in another directory |
| `make -f file` | Use a different Makefile |
| `make -k`      | Keep going after errors |
| `make VAR=val` | Override a variable |

---

## COMMON PITFALLS

- Recipes indented with spaces instead of a **TAB** fail with `missing separator`.
- Forgetting `.PHONY` on action targets like `test` or `clean`.
- Expecting `cd` or `export` on one recipe line to affect the next.
- Using `$VAR` instead of `$$VAR` for shell variables.
- macOS ships GNU Make 3.81, so newer features (e.g. `.ONESHELL`, `--output-sync`) need `brew install make` (invoked as `gmake`).

---

## RELATED

- [[Docker]]
- [[pre-commit-and-ruff]]
- [[UV Astral]]
- [[Terminal]]
