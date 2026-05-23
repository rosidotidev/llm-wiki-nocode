# Compatibility Analysis: SPEC_LLM_WIKI_CC vs SPEC_LLM_WIKI_GHB

**Date**: 2026-05-23  
**Status**: ✅ **COMPATIBLE** — Both specs describe the same requirements with equivalent implementations adapted for different platforms.

---

## Executive Summary

| Aspect | Result |
|--------|--------|
| **Core Requirements** | ✅ Identical |
| **System Architecture** | ✅ Equivalent (different tools, same logic) |
| **Data Structures** | ✅ Identical |
| **Workflows** | ✅ Identical |
| **Page Format** | ✅ Identical |
| **Tool Mapping** | ⚠️ Different (Claude Code tools vs Copilot tools) |
| **File Organization** | ✅ Mostly identical |
| **Functional Compatibility** | ✅ 100% compatible |

---

## 1. Core Requirements Comparison

### Functional Requirements (FR)

Both specs implement the **same 9 functional requirements**:

| FR | Requirement | CC | GHB | Notes |
|---|---|---|---|---|
| FR-1 | Cumulative integration | ✅ | ✅ | Entities/concepts merge with existing pages |
| FR-2 | Cross-referencing | ✅ | ✅ | `## Connections` sections, wikilinks |
| FR-3 | Contradiction detection | ✅ | ✅ | GHB mentions explicitly, CC implied in lint phase 2 |
| FR-4 | Retrieval-only querying | ✅ | ✅ | 3-8 page budget, citations only |
| FR-5 | Knowledge compounding | ✅ | ✅ | `*_pending/` → `*_approved/` → integration |
| FR-6 | Human gating | ✅ | ✅ | Explicit human review before integration |
| FR-7 | Evergreen content | ✅ | ✅ | Timestamps, source tracking, stale detection |
| FR-8 | Zero information loss | ✅ | ✅ | Verbatim code, full paragraphs, no summarizing |
| FR-9 | English-only content | ✅ | ✅ | All wiki content in English |

**Verdict**: ✅ **Identical functional requirements**

---

## 2. Directory Structure Comparison

### CC (Claude Code)
```
wiki/
├── index.md
├── log.md
├── sources/
├── entities/
├── concepts/
└── synthesis/

raw/
questions_pending/
questions_approved/
lint_pending/
lint_approved/
```

### GHB (GitHub Copilot)
```
wiki/
├── index.md
├── log.md
├── sources/
├── entities/
├── concepts/
└── synthesis/

raw/
questions_pending/
questions_approved/
lint_pending/
lint_approved/

docs/
.github/
├── agents/
├── instructions/
└── prompts/
```

**Differences**:
- **GHB** has additional metadata files (`docs/`, `.github/`) for documentation and agent definitions
- **CC** relies on `.claude/` directory (not shown in spec, but present in project)
- **Core wiki structure** is identical ✅

**Verdict**: ✅ **Structurally compatible** — Core wiki directories are identical

---

## 3. Page Format & Schema Comparison

### Frontmatter (All Pages)

**CC**:
```yaml
---
title: "Page Title"
type: "source|entity|concept|synthesis"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug-1", "source-slug-2"]
---
```

**GHB**:
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
- **Source pages**: `source_file`, `entities`, `concepts` (both specs)
- **Entity pages**: `entity_type` (both specs)
- **Synthesis pages**: `query` (both specs)

**Verdict**: ✅ **Identical**

### Page Body Structure

**Source Page**:
```markdown
## Summary
## Key Takeaways
## Entities Mentioned
## Concepts Covered
```
Both specs: ✅ Identical

**Entity Page**:
```markdown
## Overview
## From [[sources/<source-slug>]]
## Connections
```
Both specs: ✅ Identical

**Concept Page**:
```markdown
## Definition
## Deep Dive
## From [[sources/<source-slug>]]
## Connections
```
Both specs: ✅ Identical

**Synthesis Page**:
```markdown
# <Question Summary>
<Answer with [[wikilinks]]>
```
Both specs: ✅ Identical

**Wikilink Format**: `[[category/slug]]` (both specs) ✅ Identical

**Verdict**: ✅ **Identical page format**

---

## 4. Agent Workflows Comparison

### 4.1 Wiki Ingestor

#### CC Workflow (8 steps)
1. Read source
2. Extract
3. Deduplicate
4. Write source page
5. Write/update entity pages
6. Write/update concept pages
7. Update index
8. Append log

#### GHB Workflow (8 steps + auto-scan)
1. *(Auto-scan: optional Step 0)*
2. Read source
3. Extract
4. Deduplicate
5. Write source page
6. Write/update entities
7. Write/update concepts
8. Update index
9. Append log

**Key difference**: GHB explicitly names the auto-scan as Step 0; CC treats it as implicit when no arguments provided.

**Deduplication Logic**:
- Both: Check `wiki/index.md` for existing slugs
- Both: Skip if exact match, create `-v2`/`-v3` if content differs, proceed if not found
- Both: Respect entity vs concept distinction
- Both: Preserve ALL existing content when updating

**Input types** (both specs):
- No arguments: auto-scan
- `raw/file.md`: specific file
- `scan`: explicit scan
- `questions_approved/file`: synthesis page
- `lint_approved/file`: apply fixes
- `--RESET-ALL`: destructive reset

**Verdict**: ✅ **Identical workflows**

### 4.2 Wiki Querier

#### CC Workflow
1. Read `wiki/index.md`
2. Navigate via connections
3. Read 3-8 pages contextually
4. Answer by quoting wiki content
5. Save to `questions_pending/`
6. Append log

#### GHB Workflow
1. Read `wiki/index.md`
2. Identify and read most relevant page
3. Examine `## Connections` for related pages
4. Follow links reading one at a time
5. Repeat until min (3) met or max (8) hit
6. Compose answer with citations
7. Save to `questions_pending/`
8. Append log

**Key constraints** (both specs):
- ✅ Retrieval-only (no external knowledge)
- ✅ Min 3, max 8 pages
- ✅ Cite with `[[wikilinks]]`
- ✅ No synthesis or inference
- ✅ Preserve code blocks verbatim

**Verdict**: ✅ **Identical retrieval constraints**

### 4.3 Wiki Linter

#### CC Phases
**Phase 1 — Deterministic Checks**:
- Broken wikilinks
- Orphan pages
- Missing frontmatter
- Empty sections

**Phase 2 — Semantic Checks**:
- Contradictions
- Stale claims
- Missing pages
- Missing cross-references
- Suggested questions

#### GHB Phases
**Phase 1 — Deterministic Checks** (via terminal):
- Broken wikilinks
- Orphan pages
- Missing frontmatter
- Empty sections

**Phase 2 — Semantic Checks** (LLM reasoning):
- Contradictions
- Stale claims
- Missing pages
- Missing cross-references
- Suggested questions

**Output** (both specs):
- Report to `lint_pending/lint-report_<timestamp>.md`
- Severity levels: 🔴 🟡 🟢
- Append to `wiki/log.md`

**Key constraint** (both specs):
- Never modify wiki pages directly
- Only report and suggest fixes

**Verdict**: ✅ **Identical lint logic**

---

## 5. Tool Mapping Comparison

### CC (Claude Code Tools)

| Capability | Tool(s) |
|---|---|
| Read files | **Read** |
| Write files | **Write** |
| Edit files | **Edit** |
| List/find files | **Glob** |
| Search content | **Grep** |
| Terminal commands | **Bash** |

### GHB (Copilot Native Tools)

| Capability | Tool(s) |
|---|---|
| Read files/directories | `read` (read_file, list_dir) |
| Create/modify files | `edit` (create_file, replace_string_in_file) |
| Full-text search | `search` (grep_search) |
| Run terminal commands | `execute` (run_in_terminal) |

### Mapping Equivalence

| GHB Tool | CC Equivalent | Compatibility |
|---|---|---|
| `read` | **Read**, **Glob** | ✅ Equivalent (CC split into 2 tools) |
| `edit` | **Edit**, **Write** | ✅ Equivalent (CC split into 2 tools) |
| `search` | **Grep**, **Bash** | ✅ Equivalent (CC has more options) |
| `execute` | **Bash** | ✅ Equivalent |

**Verdict**: ✅ **Functionally equivalent** — Different tool names/APIs, same capabilities

---

## 6. Data Flow Comparison

### Ingest Flow
**Both specs**:
```
raw/<filename>.md → sources/<slug>.md
                 → entities/<slug>.md (new or updated)
                 → concepts/<slug>.md (new or updated)
                 → index.md (updated)
                 → log.md (appended)
```
✅ Identical

### Query Flow
**Both specs**:
```
User question → questions_pending/<slug>_<timestamp>.md
             → log.md (appended)
```
✅ Identical

### Lint Flow
**Both specs**:
```
(no input) → lint_pending/lint-report_<timestamp>.md
          → log.md (appended)
```
✅ Identical

### Human-Gated Compounding
**Both specs**:
```
questions_pending/ ──[human review]──→ questions_approved/ ──→ integration
lint_pending/      ──[human review]──→ lint_approved/      ──→ updates
```
✅ Identical

**Verdict**: ✅ **Identical data flows**

---

## 7. Key Constraints & Rules

### Ingestion Constraints
| Constraint | CC | GHB | Status |
|---|---|---|---|
| Auto-scan by default | ✅ | ✅ | ✅ Identical |
| Preserve code blocks verbatim | ✅ | ✅ | ✅ Identical |
| Translate non-English to English | ✅ | ✅ | ✅ Identical |
| Create NEW page on ambiguous dedup | ✅ | ✅ | ✅ Identical |
| Respect entity vs concept distinction | ✅ | ✅ | ✅ Identical |

### Query Constraints
| Constraint | CC | GHB | Status |
|---|---|---|---|
| Retrieval-only (no external knowledge) | ✅ | ✅ | ✅ Identical |
| Min 3, max 8 pages | ✅ | ✅ | ✅ Identical |
| Cite with wikilinks | ✅ | ✅ | ✅ Identical |
| No synthesis or inference | ✅ | ✅ | ✅ Identical |

### Lint Constraints
| Constraint | CC | GHB | Status |
|---|---|---|---|
| Phase 1 before Phase 2 | ✅ | ✅ | ✅ Identical |
| Never modify wiki pages | ✅ | ✅ | ✅ Identical |
| Use severity levels | ✅ | ✅ | ✅ Identical |

**Verdict**: ✅ **All constraints identical**

---

## 8. Naming Conventions

| Convention | CC | GHB | Status |
|---|---|---|---|
| Slugs | Lowercase, hyphens only | Lowercase, hyphens only | ✅ Identical |
| Timestamps | YYYY-MM-DD, YYYY-MM-DD HH:MM | YYYY-MM-DD, YYYY-MM-DD HH:MM | ✅ Identical |
| Wikilinks | `[[category/slug]]` | `[[category/slug]]` | ✅ Identical |
| Page type values | source\|entity\|concept\|synthesis | source\|entity\|concept\|synthesis | ✅ Identical |
| Entity types | person\|tool\|company\|project\|framework\|other | person\|tool\|company\|project\|framework\|other | ✅ Identical |

**Verdict**: ✅ **Naming conventions identical**

---

## 9. Differences & Clarifications

### Where They Differ

1. **Platform Context**
   - **CC**: Claude Code (integrated IDE tool)
   - **GHB**: GitHub Copilot (VS Code extension)
   - **Impact**: None — both use same concepts, different UI

2. **Tool APIs**
   - **CC**: Dedicated Read, Write, Edit, Glob, Grep, Bash tools
   - **GHB**: Native Copilot `read`, `edit`, `search`, `execute` tools
   - **Impact**: None — functionally equivalent, different syntax

3. **Project Documentation**
   - **GHB**: More detailed (agent definitions, gap analysis, design decisions section)
   - **CC**: More focused (integrated into `.claude/` structure)
   - **Impact**: None — documentation level differs, requirements identical

4. **Explicit Mentions**
   - **GHB**: Explicitly mentions contradiction blocks, design decisions section
   - **CC**: Implicit in lint phase 2, omits some design rationale
   - **Impact**: Minor — both specs support same features

### Where They're Identical

✅ Core requirements  
✅ Directory structure  
✅ Page format and schema  
✅ Frontmatter fields  
✅ Body structure  
✅ Wikilink format  
✅ Agent workflows  
✅ Data flows  
✅ Human-gated compounding  
✅ All constraints  
✅ Naming conventions  
✅ Conflict resolution (dedup logic)  
✅ Entity vs concept rules  

---

## 10. Migration/Compatibility Path

### Can I use CC with GHB documentation?
**Yes** ✅ — The specs are semantically identical. If you follow CC's `/wiki-ingest`, `/wiki-query`, `/wiki-lint` commands, you'll build a wiki that is **100% compatible** with GHB specifications.

### Can I migrate a wiki from CC to GHB?
**Yes** ✅ — All wiki pages, index, and log are format-identical. You would only need to reimplement the three agents using Copilot tools instead of Claude Code tools.

### Can a wiki built by GHB be read by CC?
**Yes** ✅ — All page formats are identical, so CC can read and extend GHB-built wikis without modification.

### Are there breaking changes?
**No** ❌ — Neither spec is "breaking" the other. They are parallel implementations of the same concept with different tool APIs.

---

## 11. Verdict

| Question | Answer |
|---|---|
| **Do they describe the same requirements?** | ✅ Yes, identical (9 FRs, same workflows, same constraints) |
| **Are they compatible?** | ✅ Yes, 100% compatible |
| **Can you use CC and GHB together?** | ✅ Yes, on the same wiki (tools are swappable) |
| **Should you use one over the other?** | ⚠️ Depends on your environment (Claude Code vs VS Code) |
| **Will CC output be readable by GHB?** | ✅ Yes, page format is identical |

---

## Recommendations

### For Your Project (Currently Using CC)

1. **No changes needed** — Your CC implementation is 100% spec-compliant with GHB
2. **Reference GHB for clarification** — When CC spec is ambiguous, GHB provides more detail (e.g., design decisions, contradiction handling, gap analysis)
3. **Use GHB gap analysis** — SPEC_LLM_WIKI_GHB.md references `BACKLOG_KARPATHY_GAPS.md` for a prioritized list of unimplemented features relative to Karpathy's original spec

### For Documentation

- **CC spec** is good for quick reference and practical usage
- **GHB spec** is better for understanding design decisions and original vision
- Keep both open for comprehensive understanding

---

**Conclusion**: Both SPEC files are **fully compatible implementations** of the same LLM Wiki concept. They differ only in platform (Claude Code vs Copilot) and presentation style, not in requirements, data structures, or workflows.
