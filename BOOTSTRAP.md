# BOOTSTRAP.md — LLM Wiki GHB Implementation Plan

> **Instructions for Copilot**: Read this file in its entirety before implementing.
> It contains architecture, decisions, prompt reference, and implementation steps.

---

## 1. What is this project

A reimplementation of [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) using **only native GitHub Copilot features** — zero Python, zero libraries, zero external dependencies.

Three custom agents (`.agent.md`) + three prompt templates (`.prompt.md`) + one instruction file (`.instructions.md`) replace an entire Python pipeline with workflow engine, executor, agent framework, and custom tools.

---

## 2. Architecture

### 2.1 Agents

| Agent | File | Tools | Role |
|---|---|---|---|
| wiki-ingestor | `.github/agents/wiki-ingestor.agent.md` | `[read, edit, search]` | Ingests raw documents → creates/updates wiki pages |
| wiki-querier | `.github/agents/wiki-querier.agent.md` | `[read, search, edit]` | Answers questions from the wiki (retrieval-only) |
| wiki-linter | `.github/agents/wiki-linter.agent.md` | `[read, search, execute]` | Validates wiki integrity (deterministic + semantic) |

### 2.2 Prompt Templates

| Prompt | File | Agent |
|---|---|---|
| wiki-ingest | `.github/prompts/wiki-ingest.prompt.md` | wiki-ingestor |
| wiki-query | `.github/prompts/wiki-query.prompt.md` | wiki-querier |
| wiki-lint | `.github/prompts/wiki-lint.prompt.md` | wiki-linter |

### 2.3 Instructions

| File | Purpose |
|---|---|
| `.github/instructions/wiki-schema.instructions.md` | Schema reference for all agents (already present) |

### 2.4 Python Tool → Native Copilot Mapping

| Python Tool | Copilot Tool | Notes |
|---|---|---|
| `read_wiki_page(path)` | `read` (read_file) | The agent reads .md directly |
| `write_wiki_page(path, content)` | `edit` (create_file / replace_string_in_file) | Creates or modifies .md |
| `search_wiki(query)` | `search` (grep_search) | Full-text search in workspace |
| `list_wiki_pages()` | `read` (list_dir) | Lists wiki/ directory |
| `append_log(entry)` | `edit` | Appends to wiki/log.md |
| Deterministic grep/find commands | `execute` (run_in_terminal) | For deterministic linting |

### 2.5 Data Flow

```
raw/*.md ──→ @wiki-ingestor ──→ wiki/sources/*.md
                                  wiki/entities/*.md
                                  wiki/concepts/*.md
                                  wiki/index.md (updated)
                                  wiki/log.md (appended)

User question ──→ @wiki-querier ──→ questions_pending/*.md

questions_approved/*.md ──→ @wiki-ingestor ──→ wiki/synthesis/*.md

@wiki-linter ──→ lint_pending/*.md

lint_approved/*.md ──→ @wiki-ingestor ──→ wiki pages updated
```

---

## 3. Design Decisions

1. **Monolithic ingestor**: A single agent handles extraction + integration + writing. No separate subagents.
2. **Read-only querier**: The querier does NOT modify the wiki. It only saves to `questions_pending/`. Uses `edit` only to write the output file.
3. **2-phase linter**: First deterministic phase (terminal grep/find), then semantic phase (LLM reasoning).
4. **Markdown linter output**: Human-readable report, not JSON.
5. **No hooks for v1**: Format enforcement via the system prompt.
6. **Schema from instructions file**: All agents inherit rules from `wiki-schema.instructions.md` via the `applyTo` mechanism.
7. **English-only wiki content**: All wiki pages are in English. The user interface (questions/answers) is multilingual.

---

## 4. Reference: Python Agent Prompts

These are the original prompts from the Python version. Adapt them for the `.agent.md` files.

### 4.1 SourceReaderAgent (→ wiki-ingestor, extraction phase)

```
You are a source analysis expert. You read raw documents and extract ALL information
into a structured JSON format.

When given a document's content, analyze it and respond with ONLY a valid JSON object
(no markdown fences, no preamble) matching this schema:

{
  "file_name": "<original filename>",
  "slug": "<url-friendly slug from title, lowercase, hyphens>",
  "title": "<document title>",
  "summary": "<comprehensive summary, 3-5 paragraphs covering all major points>",
  "key_takeaways": ["takeaway 1", "takeaway 2", ...],
  "claims": [
    {
      "text": "<a factual assertion made in the source>",
      "context": "<surrounding context that gives the claim meaning>",
      "entities": ["<entity names involved>"],
      "concepts": ["<concept names involved>"]
    }
  ],
  "entities": [
    {
      "name": "<Entity Name>",
      "slug": "<entity-slug>",
      "type": "<person|tool|company|project|other>",
      "description": "<one sentence>",
      "content": "<FULL paragraphs + code blocks from source about this entity>"
    }
  ],
  "concepts": [
    {
      "name": "<Concept Name>",
      "slug": "<concept-slug>",
      "definition": "<one sentence definition>",
      "content": "<FULL explanations + code blocks from source about this concept>"
    }
  ]
}

ENTITY vs CONCEPT:
- ENTITY: specific, identifiable noun (person, tool, company, class, product). Goal = entity resolution.
- CONCEPT: abstract idea, methodology, pattern, technique. Goal = knowledge synthesis.
- An item must be EITHER an entity OR a concept, never both.

CONTENT FIELD RULES:
- Copy/paste full paragraphs. Do NOT paraphrase or summarize.
- Include ALL code blocks exactly as they appear.
- The content field can be very long. That is expected and correct.
- When in doubt, INCLUDE MORE rather than less.

GENERAL RULES:
- ALWAYS produce output in English. Translate non-English sources.
- ZERO information loss.
- Slugs: lowercase, hyphens, no special characters.
- GRANULARITY: One entry per topic at the most useful level. Don't extract both general + sub-aspect.
```

### 4.2 SlugMapperAgent (→ wiki-ingestor, deduplication phase)

```
You are a wiki deduplication assistant. Given a list of EXISTING wiki page slugs
and NEW slugs extracted from a source, map each new slug to the most semantically
equivalent existing slug, or mark it as "NEW" if no good match exists.

Rules:
- Match only when the new item clearly covers the SAME topic as the existing page.
- Partial keyword overlap is NOT enough. "mcp-integration" and "mcp-server-proxies"
  are different topics unless they truly cover the same concept.
- Plural/singular variants are a match (e.g., "tool" ↔ "tools").
- A more specific slug can match a more general one if they cover the same ground.
- When in doubt, return "NEW". Creating a duplicate is cheaper than merging unrelated topics.
```

### 4.3 WikiQuerierAgent (→ wiki-querier)

```
You are a RETRIEVAL-ONLY system. You have ZERO knowledge. You answer EXCLUSIVELY by
copying relevant passages from wiki pages you read with your tools.

PAGE BUDGET: you may read at most 8 pages per question.
MINIMUM READS: you MUST read at least 3 pages before answering.

WORKFLOW — SEQUENTIAL NAVIGATION:
1. Read wiki/index.md to see all pages. Identify the most relevant page.
2. Read that page. Examine its ## Connections section.
3. If a linked page is relevant, read it next.
4. Repeat — read one page at a time, follow connections — until you have
   at least 3 pages read AND enough context, OR you hit the budget.
5. Optionally use grep_search with keywords if index/connections don't suffice.
6. Answer by QUOTING or CLOSELY PARAPHRASING what you read. Cite with [[category/slug]].
   Include verbatim any code blocks from pages. NEVER shorten code.

CONSTRAINTS — MANDATORY:
- If a fact is NOT in a wiki page you read → DO NOT include it.
- If the wiki doesn't answer → reply: "The wiki does not contain information on this topic."
- If pages mention the topic but lack details → reply: "The wiki mentions [topic] but does not include [requested detail]."
- DO NOT use your training data. DO NOT synthesize. DO NOT infer.
- Answer in the user's language. Cite with [[category/slug]].
```

### 4.4 WikiLinterAgent (→ wiki-linter, semantic phase)

```
You are the Wiki Linter. You perform SEMANTIC health checks on the wiki that
require reasoning — things that deterministic code cannot catch.

NOTE: Broken wikilinks, orphan pages, and missing frontmatter are already
checked deterministically BEFORE you run. Do NOT repeat those checks.

YOUR SEMANTIC CHECKS:
1. CONTRADICTIONS: claims in one page that conflict with claims in another.
2. STALE CLAIMS: information from older sources that newer sources may have superseded.
3. MISSING PAGES: important entities/concepts mentioned in text but lacking their own page.
4. MISSING CROSS-REFERENCES: pages that should link to each other but don't.
5. SUGGESTED QUESTIONS: gaps in knowledge that a new source or research question could fill.

WORKFLOW:
1. Read wiki/index.md to see all pages.
2. Read pages that seem related or potentially contradictory.
3. Compare claims across pages looking for conflicts or staleness.
4. Look for entity/concept names in page text without their own page.
5. Produce findings as structured suggestions.

OUTPUT FORMAT — Markdown report with sections:
## Contradictions
## Stale Claims
## Missing Pages
## Missing Cross-References
## Suggested Questions

Each item: severity (🔴 high / 🟡 medium / 🟢 low), description, pages involved, suggested action.
If no issues found in a section, write "None found."
```

---

## 5. Implementation Steps

### Step 1: Create `.github/agents/wiki-ingestor.agent.md`

Frontmatter:
```yaml
---
description: "Ingest raw documents into the wiki. Use when: ingest, process raw file, add source to wiki, scan for new files."
tools: [read, edit, search]
---
```

Body: Combine the logic of SourceReaderAgent + SlugMapperAgent + WriterExecutor into a single flow:

1. **Input**: The user provides a path to a raw file (e.g. `raw/my-doc.md`) or says "scan" to process all new files in `raw/`.
2. **Extraction**: Read the raw file. Mentally extract: title, slug, summary, key_takeaways, entities, concepts, claims.
3. **Deduplication**: Read `wiki/index.md`. For each extracted entity/concept, check if a page with a similar/equivalent slug already exists. Apply SlugMapperAgent rules.
4. **Write Source Page**: Create `wiki/sources/<slug>.md` with frontmatter and sections: Summary, Key Takeaways, Entities Mentioned, Concepts Covered.
5. **Write/Update Entity Pages**: For each entity:
   - If NEW: create `wiki/entities/<slug>.md` with frontmatter, Overview, "From [[sources/<source-slug>]]" section, Connections.
   - If EXISTS: append "From [[sources/<source-slug>]]" section to the existing page, update Connections if needed.
6. **Write/Update Concept Pages**: Same as entities.
7. **Update Index**: Add new pages to `wiki/index.md` in the appropriate sections.
8. **Append Log**: Add entry to `wiki/log.md` with format: `- **YYYY-MM-DD HH:MM** — ingest: \`<filename>\` → "<title>"`
9. **Handle questions_approved/**: If the file is in `questions_approved/`, create a `wiki/synthesis/<slug>.md` with `query` frontmatter and answer content.
10. **Handle lint_approved/**: If the file is in `lint_approved/`, apply the suggested fix to the involved pages.

Constraints:
- MUST follow wiki-schema.instructions.md (frontmatter, wikilinks, page structure)
- ALL content in English (translate if source is in another language)
- Preserve ALL code blocks verbatim
- When in doubt on slug mapping, create NEW page (cheaper than wrong merge)

### Step 2: Create `.github/agents/wiki-querier.agent.md`

Frontmatter:
```yaml
---
description: "Answer questions using the wiki as knowledge base. Use when: query wiki, ask about wiki content, search wiki for information."
tools: [read, search, edit]
---
```

Body: Adapt the WikiQuerierAgent prompt. The `edit` tool is ONLY for saving the answer to `questions_pending/`.

Workflow:
1. Read `wiki/index.md`
2. Navigate pages (max 8, min 3) following Connections
3. Answer with [[wikilink]] citations
4. Save answer: create file `questions_pending/<slug>_<YYYYMMDD-HHMMSS>.md` with `query` frontmatter and answer body

### Step 3: Create `.github/agents/wiki-linter.agent.md`

Frontmatter:
```yaml
---
description: "Validate wiki health and integrity. Use when: lint wiki, check wiki health, find broken links, audit wiki quality."
tools: [read, search, execute]
---
```

Body: Two phases.

**Phase 1 — Deterministic** (via terminal):
- Broken wikilinks: grep for `[[...]]`, verify each target exists as a file
- Orphan pages: pages not referenced by any other page nor by the index
- Missing frontmatter: pages without `title`, `type`, `created`, `updated`, `sources`

**Phase 2 — Semantic** (LLM reasoning):
- Contradictions, stale claims, missing pages, missing cross-refs, suggested questions

Output: save markdown report to `lint_pending/lint-report_<YYYYMMDD-HHMMSS>.md`

### Step 4: Create the 3 prompt templates

- `.github/prompts/wiki-ingest.prompt.md` → agent: wiki-ingestor, argument-hint
- `.github/prompts/wiki-query.prompt.md` → agent: wiki-querier, argument-hint
- `.github/prompts/wiki-lint.prompt.md` → agent: wiki-linter, argument-hint

### Step 5: Test

1. `@wiki-querier "What is an Agent?"` — should read pages and answer with citations
2. `@wiki-ingestor` + point to a raw/ file — should create the pages
3. `@wiki-linter` — should produce a report in lint_pending/

---

## 6. Quick Format Reference

### Required frontmatter (all wiki pages)

```yaml
---
title: "Page Title"
type: "source|entity|concept|synthesis"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: ["source-slug-1", "source-slug-2"]
---
```

### Wikilinks

Always `[[category/slug]]` — e.g. `[[entities/openai]]`, `[[concepts/rag]]`

### Log entry format

```
- **YYYY-MM-DD HH:MM** — <operation>: `<filename>` → "<title>"
```

### Contradiction block

```markdown
> ⚠️ **Contradiction** (detected: YYYY-MM-DD)
> - Previous claim: "<claim>" ([[sources/<slug>]])
> - New claim: "<claim>" ([[sources/<slug>]])
> - Note: <resolution hint>
```

---

## 7. What NOT To Do

- Do not create Python code
- Do not install dependencies
- Do not use MCP servers
- Do not create hooks for v1
- Do not duplicate wiki pages when the slug match is clear
- Do not answer from training data in the querier — ONLY from the wiki

---

## 8. Future Evolution (v2+)

- [ ] PostToolUse hooks to automatically validate created pages
- [ ] Dedicated `wiki-approver` agent for the pending→approved flow
- [ ] Git auto-commit after each ingest/lint
- [ ] Lint remediation: the linter proposes fixes, the ingestor applies them
- [ ] Overview and Comparison page types
- [ ] Image support (multimodal)
