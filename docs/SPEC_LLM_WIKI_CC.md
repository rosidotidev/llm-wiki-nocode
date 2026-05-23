# SPEC_LLM_WIKI_CC.md — LLM Wiki for Claude Code

**Version**: 1.0  
**Date**: 2026-05-23  
**System**: Claude Code (Andrej Karpathy's LLM Wiki concept)

---

## Overview

LLM Wiki CC is a knowledge management system for Claude Code that ingests documents, structures them into entities and concepts, and allows retrieval via natural language queries. The system is powered by three integrated skills that ensure zero training data pollution and structured knowledge.

**Core principle**: ALL wiki operations are pure retrieval and synthesis — no external knowledge, only what's in the wiki.

---

## Skills

### 1. `/wiki-ingest`

**Purpose**: Ingest raw documents into the wiki structure, creating source pages and entity/concept pages

**Input**: 
- No arguments: Auto-scan `raw/`, `questions_approved/`, `lint_approved/` and ingest everything new
- File path: `raw/my-doc.md` to ingest a specific file
- Special: `--RESET-ALL` to clear entire wiki (destructive)

**Workflow for raw documents**:
1. **Read source** — Read raw document completely (preserve ALL code blocks verbatim)
2. **Extract** — Identify:
   - `title`: Document title
   - `slug`: URL-friendly identifier (lowercase, hyphens only)
   - `summary`: Comprehensive 3-5 paragraph summary
   - `key_takeaways`: Bullet list of main points
   - `entities`: List of named, identifiable nouns (people, tools, companies, frameworks, projects)
   - `concepts`: List of abstract ideas, methodologies, patterns, techniques
3. **Deduplicate** — Check `wiki/index.md` for existing entity/concept slugs
   - If exact match found → skip
   - If content differs → create new page with `-v2`, `-v3` suffix (concurrent versions)
   - If not found → proceed
4. **Create source page** — `wiki/sources/<slug>.md` with summary, takeaways, entity/concept lists (all verbatim)
5. **Create/update entity pages** — For each extracted entity:
   - If NEW: Create `wiki/entities/<entity-slug>.md` with overview, connections
   - If EXISTS: Append new section `## From [[sources/<source-slug>]]`
6. **Create/update concept pages** — For each extracted concept:
   - If NEW: Create `wiki/concepts/<concept-slug>.md` with definition, deep dive, connections
   - If EXISTS: Append new section `## From [[sources/<source-slug>]]`
7. **Update index** — Add entries to `wiki/index.md` (alphabetical, with one-line descriptions)
8. **Append log** — Record operation in `wiki/log.md` with timestamp, files created/modified

**Output**: Source page + entity pages + concept pages created/updated, index updated, log entry appended

**Example**:
```
/wiki-ingest raw/my-agent-guide.md
```

---

### 2. `/wiki-query`

**Purpose**: Answer questions using ONLY wiki content (retrieval-only mode)

**Input**: Natural language question

**Constraints**:
- ❌ NO external knowledge
- ❌ NO synthesis or inference
- ❌ NO training data
- ✅ ONLY wiki pages
- ✅ Minimum 3 pages read
- ✅ Maximum 8 pages read
- ✅ Cite with `[[wikilinks]]`

**Workflow**:
1. Read `wiki/index.md` to see all pages
2. Navigate via connections (no search allowed)
3. Read 3-8 pages contextually
4. Answer by quoting wiki content
5. Save answer to `questions_pending/`
6. Append log entry

**Output**: Answer with citations, saved to `questions_pending/`

**Example**:
```
/wiki-query What are the main concepts in agent orchestration?
```

---

### 3. `/wiki-lint`

**Purpose**: Validate wiki health and integrity

**Phase 1 — Deterministic Checks**:
- Broken wikilinks (do all `[[category/slug]]` targets exist?)
- Orphan pages (pages not in index or referenced anywhere?)
- Missing frontmatter (all pages have title/type/created/updated/sources?)
- Empty sections (headers with no content?)

**Phase 2 — Semantic Checks**:
- Contradictions (conflicting claims in related pages?)
- Stale claims (superseded information?)
- Missing pages (entities/concepts mentioned without pages?)
- Missing cross-references (related pages not linking?)
- Suggested questions (knowledge gaps to fill?)

**Output**: Comprehensive report saved to `lint_pending/` with severity levels (🔴 🟡 🟢)

**Example**:
```
/wiki-lint
```

---

## Directory Structure

```
wiki/
├── index.md              # Catalog of all pages
├── log.md                # Chronological operation log
├── sources/              # One page per ingested source
│   └── <slug>.md
├── entities/             # Named, identifiable nouns
│   └── <slug>.md
├── concepts/             # Abstract patterns & methodologies
│   └── <slug>.md
└── synthesis/            # Approved query answers
    └── <slug>.md

raw/                       # Immutable source documents (before ingestion)
├── <filename>.md
└── ...

questions_pending/         # Auto-generated query answers (awaiting approval)
├── <slug>_YYYYMMDD-HHMMSS.md
└── ...

questions_approved/        # User-promoted answers (ready to integrate)
├── <slug>.md
└── ...

lint_pending/              # Auto-generated lint reports (awaiting review)
├── lint-report_YYYYMMDD-HHMMSS.md
└── ...

lint_approved/             # Approved lint fixes (ready to apply)
├── fix-<slug>.md
└── ...
```

---

## Page Format (All Pages)

### Frontmatter (Required for all pages)

```yaml
---
title: "Page Title"
type: "source|entity|concept|synthesis"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug-1", "source-slug-2"]
---
```

**Additional fields**:
- **Source pages**: Add `source_file`, `entities`, `concepts`
- **Entity pages**: Add `entity_type` (person|tool|company|project|framework|other)
- **Concept pages**: No additional fields
- **Synthesis pages**: Add `query` (the original question)

### Body Structure

**Source page**:
```markdown
## Summary
<3-5 paragraphs covering all major points>

## Key Takeaways
- Point 1
- Point 2

## Entities Mentioned
- [[entities/<slug>]] — description

## Concepts Covered
- [[concepts/<slug>]] — description
```

**Entity page**:
```markdown
## Overview
<one-sentence description>

## From [[sources/<source-slug>]]
<content about this entity from the source>

## Connections
- [[entities/<related>]] — relationship
- [[concepts/<related>]] — relationship
```

**Concept page**:
```markdown
## Definition
<one-sentence definition>

## Deep Dive
<full explanation>

## From [[sources/<source-slug>]]
<source-specific content>

## Connections
- [[entities/<related>]] — relationship
- [[concepts/<related>]] — relationship
```

**Synthesis page**:
```markdown
# <Question Summary>

<Full answer with [[wikilinks]] citations>
```

---

## Content Rules

### English Only
All wiki content must be in English, regardless of source language. Translate non-English sources.

### Zero Information Loss
- Copy/paste full paragraphs (never paraphrase)
- Include ALL code blocks exactly as they appear
- Never shorten, paraphrase, or simplify code
- When in doubt, include more rather than less

### Naming Conventions
- **Slugs**: Lowercase, hyphens only, no special chars
- **Timestamps**: ISO 8601 format (YYYY-MM-DD, YYYY-MM-DD HH:MM)
- **Wikilinks**: Always `[[category/slug]]` format

### Entity vs Concept Distinction
- **Entity** (Specific, Identifiable): People, tools, companies, projects, frameworks, products
- **Concept** (Abstract, General): Patterns, methodologies, techniques, ideas
- **Rule**: Each item is EITHER entity OR concept, never both

---

## Workflow: Human-Gated Compounding

The wiki uses a three-stage compounding process:

### Stage 1: Pending (Auto-Generated)
- `/wiki-ingest` creates pages in `wiki/`
- `/wiki-query` saves answers to `questions_pending/`
- `/wiki-lint` saves reports to `lint_pending/`

**These are drafts — user reviews before approval.**

### Stage 2: Approved (User Curated)
- User moves file from `*_pending/` to `*_approved/`
- Signals that content is validated and ready for integration

### Stage 3: Integration (Automated)
- `/wiki-ingest` processes `*_approved/` folders
- Integrates into wiki structure
- Clears the approved folder

---

## Skill Invocation Reference

| Skill | Syntax | Input | Output |
|-------|--------|-------|--------|
| **Ingest** | `/wiki-ingest` | `raw/file.md` or `scan` | Wiki pages + index + log |
| **Query** | `/wiki-query` | Natural language question | Answer with citations + `questions_pending/` |
| **Lint** | `/wiki-lint` | (no input) | Report with issues + `lint_pending/` |

---

## GitHub → Claude Code Mapping

| GitHub File | Claude Code File | Status | Tool Changes |
|---|---|---|---|
| `.github/agents/wiki-ingestor.agent.md` | `.claude/skills/wiki-ingest.skill.md` | ✅ Adapted | read→Read, edit→Edit, execute→Bash |
| `.github/agents/wiki-querier.agent.md` | `.claude/skills/wiki-query.skill.md` | ✅ Adapted | read→Read, edit→Edit |
| `.github/agents/wiki-linter.agent.md` | `.claude/skills/wiki-lint.skill.md` | ✅ Adapted | execute→Bash, search→Grep/Bash |
| `.github/instructions/wiki-schema.instructions.md` | `.claude/instructions/wiki-schema.instructions.md` | ❌ Identical | No changes |
| `.github/prompts/*.prompt.md` | — | ℹ️ Integrated | Prompts now in skill definitions |

**Reusability**: ~95% of logic recycled from GitHub version, adapted for Claude Code tools.

---

## `.claude/` File Inventory

| File | Purpose | Mapping |
|------|---------|---------|
| `CLAUDE.md` | Entry point and overview | — |
| `SPEC_LLM_WIKI_CC.md` | This specification document | — |
| `MAPPING.md` | Detailed GitHub→Claude Code mapping | — |
| `skills/wiki-ingest.skill.md` | Ingest skill definition | `.github/agents/wiki-ingestor.agent.md` |
| `skills/wiki-query.skill.md` | Query skill definition | `.github/agents/wiki-querier.agent.md` |
| `skills/wiki-lint.skill.md` | Lint skill definition | `.github/agents/wiki-linter.agent.md` |
| `instructions/wiki-schema.instructions.md` | Page format schema (universal) | `.github/instructions/wiki-schema.instructions.md` |
| `instructions/claude-code-usage.md` | Practical usage guide | — |

---

## Quick Start

### 1. Ingest Your First Document

Place a document in `raw/my-guide.md`, then:
```
/wiki-ingest raw/my-guide.md
```

Skill will extract entities/concepts and create wiki pages.

### 2. Query the Wiki

Ask a question:
```
/wiki-query What is the main pattern for this topic?
```

Skill will read 3-8 pages and answer using only wiki content.

### 3. Check Wiki Health

Run a lint check:
```
/wiki-lint
```

Skill will validate structure and suggest improvements.

### 4. Approve and Integrate

After review:
1. Move `questions_pending/answer.md` → `questions_approved/`
2. `/wiki-ingest questions_approved/answer.md` to integrate

---

## Key Constraints

✅ **Ingestion**:
- Auto-scan mode: process all new sources
- Preserve code blocks verbatim
- Translate non-English to English
- Create NEW page on ambiguous dedup
- Respect entity vs concept distinction

✅ **Query**:
- RETRIEVAL-ONLY: use only wiki content
- Read min 3, max 8 pages
- Cite with `[[wikilinks]]`
- No external knowledge

✅ **Lint**:
- Phase 1: deterministic checks first
- Phase 2: semantic checks only after
- Never modify wiki pages directly
- Use severity levels: 🔴 🟡 🟢

---

## System Health

The wiki tracks its own health via:
- `wiki/log.md` — Chronological operation audit trail
- `lint_pending/lint-report_*.md` — Health check reports
- `wiki/index.md` — Current state of all pages

Run `/wiki-lint` regularly to catch:
- Broken wikilinks
- Orphaned pages
- Missing frontmatter
- Contradictions
- Stale claims
- Missing cross-references

---

## Philosophy

**Knowledge without pollution**: Every answer comes from the wiki. No hallucinations, no synthesis, no training data. What you read is what was ingested.

**Structured for retrieval**: Entities, concepts, and sources are distinct. You know where knowledge came from and can trace relationships.

**Human-gated**: Before anything integrates, a human reviews it. This prevents drift and keeps the wiki authoritative.

**Audit trail**: Every operation is logged in `wiki/log.md`. You can trace what was added, when, and from where.

---

**For more details, see**:
- `CLAUDE.md` — System overview
- `MAPPING.md` — GitHub → Claude Code mapping
- `instructions/wiki-schema.instructions.md` — Page format details
- `instructions/claude-code-usage.md` — Practical workflow guide
