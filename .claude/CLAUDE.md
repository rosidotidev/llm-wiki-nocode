# LLM Wiki — Claude Code

## Overview

This is an implementation of Andrej Karpathy's LLM Wiki concept using Claude Code. All wiki operations are exposed as three integrated commands: `/wiki-ingest`, `/wiki-query`, and `/wiki-lint`. These instructions define the shared schema, format rules, and constraints that all commands must follow.

---

## Command Invocation

To invoke a wiki command, use the `/wiki-*` syntax in Claude Code:

- **`/wiki-ingest`** — Ingest a source document into the wiki
- **`/wiki-query`** — Query the wiki (retrieval-only, no training data)
- **`/wiki-lint`** — Run a comprehensive health check on the wiki

Each command executes the workflow defined in `.claude/commands/` with full access to wiki logic and zero training data pollution.

---

## Quick Start

### 1. Ingest a Document
```
/wiki-ingest raw/my-document.md
```

### 2. Ask a Question
```
/wiki-query What are the main concepts?
```

### 3. Check Wiki Health
```
/wiki-lint
```

---

## Complete Specification

**See [`SPEC_LLM_WIKI_CC.md`](SPEC_LLM_WIKI_CC.md) for the complete specification including**:
- Detailed command workflows
- Directory structure
- Page format schema
- Content rules
- GitHub→Claude Code mapping
- Human-gated compounding process

---

## Directory Layout

```
wiki/                      # Main wiki content
├── index.md               # Catalog of all pages
├── log.md                 # Chronological operation log
├── sources/               # Ingested source documents
├── entities/              # Named entities (people, tools, companies, etc.)
├── concepts/              # Abstract patterns and methodologies
└── synthesis/             # Approved query answers

raw/                       # Raw documents awaiting ingestion
questions_pending/         # Auto-generated answers (awaiting approval)
questions_approved/        # User-approved answers (ready to integrate)
lint_pending/              # Auto-generated lint reports (awaiting review)
lint_approved/             # Approved lint fixes (ready to apply)
```

---

## Frontmatter (All Pages)

Every wiki page MUST have complete YAML frontmatter:

```yaml
---
title: "Page Title"
type: "source|entity|concept|synthesis"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug-1", "source-slug-2"]
---
```

---

## Content Rules

### English-Only
ALL wiki content must be in English, regardless of source language.

### Zero Information Loss
- Copy/paste full paragraphs rather than summarizing
- Preserve ALL code blocks exactly as they appear
- Never shorten, paraphrase, or simplify code
- When in doubt, include more rather than less

### Naming Conventions

**Slugs:** Lowercase, hyphens only, no special chars

**Timestamps:** ISO 8601 format (YYYY-MM-DD, YYYY-MM-DD HH:MM)

---

## Entity vs Concept Distinction

**Entity** (Specific, Identifiable): People, tools, companies, projects, frameworks, products

**Concept** (Abstract, General): Patterns, methodologies, techniques, ideas

Never both - each item is EITHER entity OR concept.

---

## Tool Usage by Command

### `/wiki-ingest` command
Tools: Read, Edit, Write, Glob, Grep, Bash
- Use Glob to list directories
- Use Grep to search for existing pages
- Use Read for source files and wiki pages
- Use Write to create new pages
- Use Edit to append to existing files
- Use Bash for file operations

### `/wiki-query` command
Tools: Read, Edit, Write
- Read ONLY wiki pages (never modify)
- Write answer to questions_pending/
- Edit to append to wiki/log.md

### `/wiki-lint` command
Tools: Read, Grep, Bash, Write, Edit
- Grep for wikilinks and patterns
- Read for semantic checks
- Bash for deterministic checks
- Write to save report
- Edit to append to log

---

## Behavioral Constraints

### Ingest Command
- Auto-scan mode: Process all new sources without asking
- Preserve all code blocks verbatim
- Translate all non-English to English
- Create NEW page on ambiguous dedup
- Respect entity vs concept distinction

### Query Command
- RETRIEVAL-ONLY: Use only wiki content
- Read min 3, max 8 pages per question
- Cite every claim with [[wikilinks]]
- Include code blocks verbatim
- Save answer to questions_pending/

### Lint Command
- Phase 1: Deterministic checks first
- Phase 2: Semantic checks only after
- Never modify wiki pages
- Include actionable fixes
- Use severity levels: 🔴 🟡 🟢

---

## Human-Gated Compounding

1. **Pending stage:** Auto-generated content in `*_pending/`
2. **Approved stage:** User reviews and moves to `*_approved/`
3. **Integration:** `/wiki-ingest` processes and integrates into wiki

---

## Documentation Files

| File | Purpose |
|------|---------|
| **SPEC_LLM_WIKI_CC.md** | Complete specification (commands, workflows, schema) |
| **MAPPING.md** | GitHub→Claude Code mapping and changes |
| **instructions/wiki-schema.instructions.md** | Page format reference (universal) |
| **instructions/claude-code-usage.md** | Practical workflow guide |

---

**System created**: 2026-05-23  
**Status**: Ready for use  
**Commands**: 3 (ingest, query, lint)
