---
tags: [caching, architecture, data-engineering]
---
# Cache Manifest

A manifest is a small index file that describes what a cache holds, without holding the data itself. Before fetching anything, the program reads the manifest to learn what is already stored and what is missing. See [[Caching]] for the general concepts.

---

## WHAT IT IS

```
manifest.json          the index: "what do I have, and where is it?"
data/
├── aapl-2025.parquet  the cached data itself
└── ibm-2025.parquet
```

- The **data files** are big and expensive to read.
- The **manifest** is tiny and cheap to read.
- Checking the manifest first avoids opening or re-downloading data just to find out it is already there (a hit) or not (a miss).

---

## EXAMPLE

Illustrative layout for a market data range cache:

```json
{
  "schema_version": 1,
  "bar": {
    "aapl": {
      "ranges": [
        {
          "start": "2025-01-01T00:00:00Z",
          "end": "2025-06-01T00:00:00Z",
          "path": "data/aapl-2025.parquet"
        }
      ]
    }
  }
}
```

A lookup for `("bar", "aapl")` finds which date ranges are cached. Any part of the requested range not covered by an entry has to be fetched from the provider.

---

## RULES AND BEST PRACTICES

### KEEP IT SMALL AND SEPARATE

- Store only metadata (keys, ranges, paths, timestamps). Never the data itself.
- The manifest is the index and the data files are the records. If they disagree, the cache is wrong.

### USE RELATIVE PATHS

- Store paths relative to the cache directory, never absolute paths like `/Users/name/project/...`.
- Absolute paths break on other machines and leak local details if the file is committed.

### WRITE ATOMICALLY

- Write the new manifest to a temporary file, then rename it over the old one.
- A crash mid-write then leaves the old manifest intact instead of a half-written, corrupt one.

```python
tmp_path = manifest_path.with_suffix(".tmp")
tmp_path.write_text(json.dumps(manifest))
tmp_path.replace(manifest_path)  # atomic rename
```

### ADD A SCHEMA VERSION

- Include a `schema_version` field so future code can detect an old format and migrate or rebuild it.

### NORMALIZE KEYS

- Pick one canonical form and apply it everywhere, for example lowercase symbols.
- Otherwise `"AAPL"` and `"aapl"` become two separate entries and the cache silently misses.

### BE EXPLICIT ABOUT RANGES AND TIME

- Store dates in UTC, ISO 8601.
- Use half-open ranges, inclusive of `start` and exclusive of `end`, so adjacent ranges never overlap or leave gaps.

### MAKE IT REBUILDABLE

- The manifest should be derivable by scanning the data files. If it is lost or corrupt, rebuild it instead of failing.
- Treat a missing or unreadable manifest as an empty cache, not a crash.

### ONE WRITER AT A TIME

- Two processes updating the manifest can overwrite each other's changes.
- Use a file lock or a single owner process for writes. Reads are safe.

### DON'T COMMIT IT

- A manifest describes local cache state, not source code. Add it and the data directory to `.gitignore`.

---

## COMMON PITFALLS

- Manifest and data drifting apart, such as a data file deleted by hand but still listed
- Updating the manifest before the data file is fully written, so a crash leaves an entry pointing at nothing
- Hardcoded absolute paths
- No version field, so a format change breaks old caches with no clear error
