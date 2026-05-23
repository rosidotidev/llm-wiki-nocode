---
description: "Wiki page format conventions (frontmatter, structure, wikilinks). Use when: writing wiki pages, creating wiki content, updating wiki pages, wiki formatting."
---

# Wiki Page Format Reference

All wiki pages MUST follow these conventions. This is the single source of truth for page structure.

## Directory Layout

```
<WIKI_ROOT_DIR>/
  wiki/
    index.md        — Catalog of all pages
    log.md          — Append-only timeline of operations
    sources/        — One page per ingested source
    entities/       — Pages for named entities
    concepts/       — Pages for abstract concepts
    synthesis/      — Pages from approved query answers
  raw/              — Immutable source documents
  questions_approved/  — User-promoted answers for integration
  questions_pending/   — Auto-generated query answers
```

## Frontmatter (all pages)

```yaml
---
title: "Page Title"
type: "source|entity|concept|synthesis"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug-1", "source-slug-2"]
---
```

## Source Page (`wiki/sources/<slug>.md`)

Additional frontmatter: `source_file`, `entities`, `concepts`

Body sections: Summary, Key Takeaways, Entities Mentioned, Concepts Covered.

## Entity Page (`wiki/entities/<slug>.md`)

Additional frontmatter: `entity_type` (person|tool|company|project|framework|other)

Body sections: Overview, From [[sources/<slug>]] (one per source), Connections.

## Concept Page (`wiki/concepts/<slug>.md`)

Body sections: Definition, Deep Dive, From [[sources/<slug>]] (one per source), Connections.

## Synthesis Page (`wiki/synthesis/<slug>.md`)

Additional frontmatter: `query`

Body: Full answer content with [[wikilinks]] citations.

## Wikilink Format

Always: `[[category/slug]]` — e.g. `[[entities/openai]]`, `[[concepts/rag]]`

## Index Format

```markdown
---
title: Wiki Index
type: index
---

# Wiki Index

## Sources
- [[sources/<slug>]]

## Entities
- [[entities/<slug>]]

## Concepts
- [[concepts/<slug>]]

## Synthesis
- [[synthesis/<slug>]]
```

## Log Entry Format

```markdown
- **YYYY-MM-DD HH:MM** — <operation>: `<filename>` → "<title>"
```

## Contradiction Block

```markdown
> ⚠️ **Contradiction** (detected: YYYY-MM-DD)
> - Previous claim: "<claim>" ([[sources/<slug>]])
> - New claim: "<claim>" ([[sources/<slug>]])
> - Note: <resolution hint>
```
