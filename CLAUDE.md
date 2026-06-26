# Obsidian Vault Writing Guide

## FILE STRUCTURE

Every note must follow this exact order:

```
---
tags: [tag1, tag2]
---
# Note Title

Content starts here.
```

Rules:
- Frontmatter (`---` block) must be the **first thing in the file** — no content before it
- The `# H1` heading immediately follows the closing `---`, no blank line
- No duplicate content

## FRONTMATTER FIELDS

Required:

```yaml
---
tags: [relevant, kebab-case, tags]
---
```

- `tags`: lowercase, hyphenated, array format

## SECTION SEPARATORS

Separate every `## H2` section with a horizontal rule:

```
---

## Next Section
```

## HEADING HIERARCHY

```
# H1 — title only, one per file
## H2 — MAJOR SECTIONS (all caps)
### H3 — SUBSECTIONS WITHIN A SECTION (all caps)
```

- H1 is the only heading that uses title case — it matches the note title.
- H2 and below must be ALL CAPS.
- Never skip levels (e.g., don't jump from H2 to H4).

## CODE BLOCKS

Always specify the language:

````
```bash
git status
```

```python
x = 1
```
````

For plain text or terminal output with no language, use ` ``` ` with no tag.

## INTERNAL LINKS

Use Obsidian wiki-links to reference other notes:

```
[[Note Title]]
```

## LISTS

- Use `-` for unordered lists (not `*` or `+`)
- Indent with 2 or 4 spaces for nested items

## TABLES

Align pipes for readability:

```
| Column A | Column B |
|---|---|
| value    | value    |
```

## CHECKLIST FOR NEW NOTES

1. Frontmatter at the top with `tags`
2. H1 heading matching the title
3. Sections use H2 and below
4. Code blocks have language tags
5. No content duplicated
