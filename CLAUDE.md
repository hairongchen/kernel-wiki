# Kernel Wiki — Schema

This is a personal knowledge base about the **Linux kernel**, maintained by an LLM (Claude) on behalf of the user. The wiki follows the LLM Wiki pattern: raw sources go in, structured wiki pages come out.

## Directory structure

```
kernel-wiki/
├── CLAUDE.md          # this file — schema and conventions
├── raw/               # immutable source documents (user-curated)
│   └── assets/        # images and non-text attachments
├── wiki/              # LLM-generated and LLM-maintained markdown
│   ├── index.md       # content catalog (LLM-maintained)
│   ├── log.md         # chronological activity log (append-only)
│   ├── overview.md    # high-level map of wiki coverage
│   ├── sources/       # source summary pages (src-<slug>.md)
│   ├── entities/      # entity pages (subsystems, data structures, functions)
│   ├── concepts/      # concept pages (cross-cutting ideas)
│   ├── comparisons/   # comparison pages (side-by-side analyses)
│   └── analyses/      # analysis pages (deep-dives, syntheses)
└── gist.md            # original idea document (reference only)
```

## Page types

| Type | Naming | Purpose |
|------|--------|---------|
| **Source summary** | `sources/src-<slug>.md` | One per ingested source. Summarizes key content, links to related pages. |
| **Entity** | `entities/<entity-name>.md` | A kernel subsystem, data structure, function, person, or project. |
| **Concept** | `concepts/concept-<name>.md` | A cross-cutting idea (e.g., lockless programming, memory ordering). |
| **Comparison** | `comparisons/cmp-<name>.md` | Side-by-side analysis of two or more alternatives. |
| **Analysis** | `analyses/analysis-<name>.md` | Deep-dive or synthesis produced from a query. |
| **Overview** | `overview.md` | High-level map of the wiki's coverage and key findings. |

## Page format

Every wiki page uses this template:

```markdown
---
type: <source|entity|concept|comparison|analysis|overview>
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [list of source slugs this page draws from]
tags: [relevant tags]
---

# Title

Content here. Use standard markdown links to reference other wiki pages.

## See also

- [related-page](relative/path/related-page.md)
```

## Workflows

### Ingest

When the user adds a new source to `raw/` and asks to ingest it:

1. Read the source document fully.
2. Discuss key takeaways with the user if they want interaction, or proceed directly if they say to batch-process.
3. Create a source summary page (`wiki/sources/src-<slug>.md`).
4. Create or update entity pages (`wiki/entities/`) for any kernel subsystems, data structures, functions, or people mentioned significantly.
5. Create or update concept pages (`wiki/concepts/`) for cross-cutting ideas.
6. Update `wiki/index.md` with new/changed pages.
7. Append an entry to `wiki/log.md`.
8. Update `wiki/overview.md` if the new source shifts the big picture.

### Query

When the user asks a question:

1. Read `wiki/index.md` to find relevant pages.
2. Read those pages.
3. Synthesize an answer with citations (`[[page]]` links).
4. If the answer is substantial and reusable, offer to file it as a new wiki page (analysis or comparison type).

### Lint

When the user asks for a health check:

1. Scan all wiki pages for: broken wikilinks, orphan pages, contradictions, stale claims, missing cross-references, concepts mentioned but lacking their own page.
2. Report findings and suggest fixes.
3. Apply fixes with user approval.

## Conventions

- Use standard markdown links with relative paths for internal links (e.g., `[process-management](entities/process-management.md)` from `wiki/`, or `[process-management](process-management.md)` within `entities/`). Aliased links use `[display text](path.md)`. Paths must be relative to the linking file's directory.
- Tags in frontmatter use lowercase, hyphenated format: `memory-management`, `scheduler`, `file-systems`.
- Source slugs are derived from the source filename without extension.
- Keep pages focused. If a page grows beyond ~300 lines, consider splitting.
- When new information contradicts existing wiki content, note the contradiction explicitly and cite both sources.
- Dates use ISO 8601 format (YYYY-MM-DD).
- Write in clear, technical English. Assume the reader has general CS knowledge but may not know kernel internals.
- No git operations — the user has explicitly requested no git usage.
