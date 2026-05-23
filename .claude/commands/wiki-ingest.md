---
name: "wiki-ingest"
description: "Ingest raw documents into the wiki. Creates source pages AND entity/concept pages. Use: /wiki-ingest [path|scan|--RESET-ALL]"
usage: "/wiki-ingest [raw/file.md | scan | --RESET-ALL]"
trigger: "user_args"
---

# Wiki Ingest Command

## Execution

**No arguments:** Auto-scan all three sources sequentially:
1. Scan `raw/` → ingest all new files (create source + entity + concept pages)
2. Scan `questions_approved/` → create synthesis pages
3. Scan `lint_approved/` → apply approved fixes to wiki pages
4. Report combined stats (sources, entities, concepts created/updated)

**With arguments:** Extract input from `user_args` (file path, "scan", or "--RESET-ALL")
1. Detect input type and follow the corresponding workflow
2. Complete all steps sequentially
3. Report completion with detailed stats

Ingest the specified source into the wiki. Follow the full pipeline: extract → deduplicate → write pages → update index → append log.

**Input parameter**: What to ingest (file path like `raw/my-doc.md`, or "scan" to process all new files in raw/)

## Workflow

You are the Wiki Ingestor — the sole agent responsible for creating and updating wiki pages. You process raw documents, approved questions, and approved lint fixes into structured wiki content.

**ALL wiki content MUST be written in English.** Translate non-English sources.

### Input Detection

Determine what you're ingesting based on the input:

- **Raw file** (`raw/<filename>`): Full extraction pipeline (steps 1–8)
- **"scan"**: List `raw/` directory, identify files not yet in `wiki/sources/`, process each
- **Approved question** (`questions_approved/<filename>`): Create synthesis page (step 9)
- **Approved lint fix** (`lint_approved/<filename>`): Apply fixes to wiki pages (step 10)
- **`--RESET-ALL`**: Wipe all wiki content, preserving only the empty structure (see Step 11)

### Step 1 — Read Source

Use the **Read** tool to read the raw file completely. Preserve ALL code blocks verbatim.

### Step 2 — Extract

Mentally extract from the source:

- **title**: Document title
- **slug**: URL-friendly (lowercase, hyphens, no special chars)
- **summary**: Comprehensive 3–5 paragraph summary covering all major points
- **key_takeaways**: List of key points
- **entities**: Named, identifiable nouns (person, tool, company, project, framework, class, product)
- **concepts**: Abstract ideas, methodologies, patterns, techniques

**ENTITY vs CONCEPT rules:**
- ENTITY = specific, identifiable noun. Goal: entity resolution.
- CONCEPT = abstract idea, methodology, pattern, technique. Goal: knowledge synthesis.
- An item is EITHER entity OR concept, never both.

**CONTENT rules:**
- Copy/paste full paragraphs. Do NOT paraphrase or summarize.
- Include ALL code blocks exactly as they appear.
- When in doubt, INCLUDE MORE rather than less.

### Step 3 — Deduplicate

Check if the entity/concept already exists:
- Read `wiki/entities/` or `wiki/concepts/` directory
- Search for pages with overlapping slugs or titles
- If found and content is identical → **SKIP** (already ingested)
- If found but content differs → **CREATE NEW PAGE** with `-v2`, `-v3` suffix (concurrent versions)
- If NOT found → **PROCEED** to Step 4

### Step 4 — Write Source Page

Create `wiki/sources/<slug>.md`:

```yaml
---
title: "[Document Title]"
type: "source"
source_file: "[filename from raw/]"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
entities: ["entity-slug-1", "entity-slug-2"]
concepts: ["concept-slug-1", "concept-slug-2"]
---

## Summary
[Comprehensive 3-5 paragraph summary of document]

## Key Takeaways
- Takeaway 1
- Takeaway 2

## Entities Mentioned
- [[entities/entity-slug-1]] — brief role in this source
- [[entities/entity-slug-2]] — brief role in this source

## Concepts Covered
- [[concepts/concept-slug-1]] — how source relates to this
- [[concepts/concept-slug-2]] — how source relates to this
```

**Important**: The `entities` and `concepts` arrays in frontmatter MUST list ALL extracted entities/concepts. Do NOT extract individual details yet — the entity/concept pages handle that.

### Step 5 — Write/Update Entity Pages

For EACH extracted entity:

**IF NEW** (not in `wiki/index.md`):
```yaml
---
title: "[Entity Name]"
type: "entity"
entity_type: "person|tool|company|project|framework|other"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug"]
---

## Overview
[2-3 sentence description of what this entity is]

## From [[sources/<source-slug>]]
[Full detail from source — copy verbatim, preserve code blocks]

## Connections
- [[concepts/related-concept]] — relationship description
- [[entities/related-entity]] — relationship description
```

**IF EXISTS** (already in wiki):
- Open `wiki/entities/<entity-slug>.md`
- Add new frontmatter entry: append `"source-slug"` to `sources` array
- Add new section: `## From [[sources/<source-slug>]]` with all mentions from this source
- Update `updated: "YYYY-MM-DD"` with today's date
- Preserve ALL existing content (no overwriting)

### Step 6 — Write/Update Concept Pages

For EACH extracted concept:

**IF NEW** (not in `wiki/index.md`):
```yaml
---
title: "[Concept Name]"
type: "concept"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug"]
---

## Definition
[2-3 sentence concise definition based on this source]

## Deep Dive
[Full explanation from this source]

## From [[sources/<source-slug>]]
[Specific contribution and details from source — copy verbatim, preserve code blocks]

## Connections
- [[concepts/related-concept]] — relationship
- [[entities/related-entity]] — relationship
```

**IF EXISTS** (already in wiki):
- Open `wiki/concepts/<concept-slug>.md`
- Add new frontmatter entry: append `"source-slug"` to `sources` array
- Add new section: `## From [[sources/<source-slug>]]` with all mentions from this source
- Update `updated: "YYYY-MM-DD"` with today's date
- **Review** the "Definition" and "Deep Dive" sections — if new source adds important context, enhance them (do NOT remove existing content)
- Preserve ALL existing content (no overwriting)

### Step 7 — Update Index

Edit `wiki/index.md` to add ALL new pages (only if they don't already exist):

**For new source pages:**
- Add to `## Sources` section: `- [[sources/<slug>]] — one-line summary of the document`

**For new entity pages:**
- Add to `## Entities` section (alphabetical): `- [[entities/<slug>]] — one-line description (what it is, one sentence)`

**For new concept pages:**
- Add to `## Concepts` section (alphabetical): `- [[concepts/<slug>]] — one-line definition`

**Guidelines:**
- Entries are alphabetical within each section
- One-line descriptions extracted from page content (not created from scratch)
- Do NOT remove existing entries — only add new ones
- Format: `- [[category/slug]] — description`

### Step 8 — Append Log

Edit `wiki/log.md` to append a new entry:

```
## [YYYY-MM-DD HH:MM] Ingested: [source-slug]
- Source: `wiki/sources/[source-slug].md` — [one-liner description]
- Entities: [N] new, [M] updated
- Concepts: [N] new, [M] updated
- Index: [X] entries added
```

**Example:**
```
## [2026-05-23 15:12] Ingested: mfa-agents-howto
- Source: `wiki/sources/mfa-agents-howto.md` — Microsoft Agent Framework Python how-to
- Entities: 7 new (OpenAI, Azure OpenAI, Anthropic, etc.)
- Concepts: 10 new (Agent, Tools, MCP Integration, etc.)
- Index: 17 entries added (1 source + 7 entities + 10 concepts)
```

### Step 9 — Approved Question Integration

If ingesting from `questions_approved/`:
1. Read the approved answer file
2. Create `wiki/synthesis/<question-slug>.md`
3. Frontmatter: `type: "synthesis"`, cite sources
4. Update `wiki/index.md` and `wiki/log.md`

### Step 10 — Approved Lint Fixes

If ingesting from `lint_approved/`:
1. Read the approved fixes file
2. Apply each fix to the target wiki page (edit, not replace)
3. Update timestamps
4. Append to `wiki/log.md`

### Step 11 — RESET-ALL

**THIS STEP IS DESTRUCTIVE. Confirm with the user before proceeding.**

If input is `--RESET-ALL`:
1. Delete all files in `raw/`, `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`, `wiki/synthesis/`
2. Reset `wiki/index.md` to empty template
3. Reset `wiki/log.md` to empty template
4. Append to log: `## [YYYY-MM-DD HH:MM] RESET: All wiki content and source files cleared. Structure preserved.`
5. Report: "Wiki reset complete. All content and raw source files removed. Ready for fresh start."

**Note**: Normal ingestion (no arguments) will scan and ingest files from `raw/` **without** deleting them. Only `--RESET-ALL` deletes raw files.
