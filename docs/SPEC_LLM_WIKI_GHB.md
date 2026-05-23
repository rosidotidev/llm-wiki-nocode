# LLM Wiki GHB — Technical Specification

> A pure GitHub Copilot implementation of Andrej Karpathy's LLM Wiki concept.
> Zero code. Zero dependencies. Only agents, instructions, and prompts.

---

## 1. Introduction & Motivation

This project implements [Andrej Karpathy's LLM Wiki concept](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — the idea that an LLM can maintain a personal, evergreen knowledge base that grows and evolves over time by ingesting sources, connecting ideas, detecting contradictions, and answering questions grounded exclusively in its own content.

**What we took from Karpathy's vision:**

- A wiki that **integrates** new knowledge in the context of what already exists (not a flat extraction)
- A system that **connects** entities and concepts across sources, building a graph of knowledge
- **Contradiction detection** — when new information conflicts with existing claims, the system flags it
- **Query-grounded answers** — the system answers questions exclusively from wiki content, never from training data
- **Compounding knowledge** — answers and corrections flow back into the wiki, making it richer over time

**Our adaptation:**

Karpathy's gist describes the concept abstractly. We implemented it as a **zero-code, Copilot-only system** — three custom agents running inside VS Code replace what would traditionally require a Python pipeline with workflow engines, vector stores, and custom tooling. The entire orchestration lives in agent definitions (`.agent.md`), instruction files (`.instructions.md`), and prompt templates (`.prompt.md`).

---

## 2. Functional Requirements

### 2.1 How We Interpreted the Requirements

Karpathy's core insight is that an LLM wiki is NOT a database of mechanically extracted cards. It is a **cumulative intellectual artifact** — an organism that evolves. We translated this into the following functional requirements:

| # | Requirement | Interpretation |
|---|---|---|
| FR-1 | **Cumulative integration** | Each new source is integrated in the context of what already exists. New entities/concepts merge with or extend existing pages — they don't create isolated duplicates. |
| FR-2 | **Cross-referencing** | Every page connects to related pages via explicit `## Connections` sections. The wiki forms a navigable graph, not a flat list. |
| FR-3 | **Contradiction detection** | When a new source makes a claim that conflicts with an existing page, the system adds a visible contradiction block rather than silently overwriting. |
| FR-4 | **Retrieval-only querying** | Questions are answered exclusively from wiki content. The LLM navigates the wiki like a researcher — reading index, following links, gathering evidence — and cites every claim. |
| FR-5 | **Knowledge compounding** | Query answers and lint suggestions can be promoted back into the wiki as synthesis pages or page corrections, creating a feedback loop. |
| FR-6 | **Human gating** | No auto-generated content enters the wiki without passing through a human review stage (`*_pending/` → `*_approved/`). |
| FR-7 | **Evergreen content** | Pages have `updated` dates, sources are tracked, and the linter detects stale claims — ensuring the wiki doesn't decay silently. |
| FR-8 | **Zero information loss** | Ingest preserves all code blocks verbatim, copies full paragraphs rather than summarizing, and errs on the side of including more. |
| FR-9 | **English-only content** | All wiki pages are written in English regardless of source language. The user interface (questions, commands) is multilingual. |

### 2.2 Wiki Structure Design

To support these requirements, we structured the wiki around **four page types**, each serving a distinct purpose:

| Page Type | Directory | Purpose |
|---|---|---|
| **Source** | `wiki/sources/` | One page per ingested document — summary, key takeaways, lists of entities and concepts extracted. Acts as provenance record. |
| **Entity** | `wiki/entities/` | Pages for named, identifiable nouns (people, tools, companies, frameworks). Accumulates information from multiple sources. |
| **Concept** | `wiki/concepts/` | Pages for abstract ideas, methodologies, patterns. Includes definition, deep dive, and per-source contributions. |
| **Synthesis** | `wiki/synthesis/` | Pages created from promoted query answers. Represents the wiki's own conclusions grounded in existing content. |

The **entity vs concept** distinction is critical: an entity has a specific identity (e.g., "OpenAI", "Pydantic"), while a concept is abstract (e.g., "structured outputs", "multi-agent orchestration"). An item is always one or the other, never both.

---

## 3. Directory Layout

```
<project-root>/
│
├── wiki/                           ← LLM-generated wiki content
│   ├── index.md                    ← Navigable catalog of all pages (entry point for agents)
│   ├── log.md                      ← Append-only timeline of all operations
│   ├── sources/                    ← One page per ingested source document
│   ├── entities/                   ← Named entity pages (person, tool, company, etc.)
│   ├── concepts/                   ← Abstract concept pages
│   └── synthesis/                  ← Pages from promoted query answers
│
├── raw/                            ← Source documents to ingest (immutable, never modified)
│
├── questions_pending/              ← Query answers (auto-generated by querier)
├── questions_approved/             ← User-promoted answers (ready for integration)
│
├── lint_pending/                   ← Lint reports (auto-generated by linter)
├── lint_approved/                  ← User-approved fixes (ready for integration)
│
├── docs/                           ← Project documentation
├── .github/
│   ├── agents/                     ← Agent definitions (.agent.md)
│   ├── instructions/               ← Shared instruction files (.instructions.md)
│   └── prompts/                    ← User-facing prompt templates (.prompt.md)
│
└── README.md
```

### 3.1 Key Files

**`wiki/index.md`** — The primary navigation tool for all agents. Contains a wikilink and one-line summary for every page, organized by type. Agents read this first to understand what exists and plan their navigation.

**`wiki/log.md`** — Append-only audit trail. Every operation (ingest, query, lint, reset) is logged with timestamp, operation type, and pages touched.

---

## 4. Architecture Overview

### 4.1 Zero-Code Design

The entire system runs inside VS Code with GitHub Copilot. There is no Python code, no installed dependencies, no build step. Three files types compose the architecture:

| File Type | Location | Role |
|---|---|---|
| `.agent.md` | `.github/agents/` | Defines an agent's identity, tools, and complete behavioral instructions |
| `.instructions.md` | `.github/instructions/` | Shared format rules inherited by all agents via `applyTo` |
| `.prompt.md` | `.github/prompts/` | User-facing entry points that delegate to the correct agent |

### 4.2 Tool Mapping

Agents use native Copilot tools. No custom tooling is needed:

| Capability | Copilot Tool | Usage |
|---|---|---|
| Read files/directories | `read` (read_file, list_dir) | Read wiki pages, index, raw sources |
| Create/modify files | `edit` (create_file, replace_string_in_file) | Write wiki pages, save answers, append log |
| Full-text search | `search` (grep_search) | Search across wiki for keywords, verify links |
| Run terminal commands | `execute` (run_in_terminal) | Deterministic lint checks (grep, find) |

### 4.3 Composition Model

```
User ──→ prompt template (.prompt.md) ──→ agent (.agent.md)
                                              │
                                              ├── inherits format rules from .instructions.md
                                              ├── uses native Copilot tools
                                              └── reads schema.md for page conventions
```

---

## 5. Agents

### 5.1 Wiki Ingestor (`@wiki-ingestor`)

**Purpose**: The sole agent responsible for creating and updating wiki pages. Transforms raw documents, approved questions, and approved lint fixes into structured wiki content.

**Tools**: `read`, `edit`, `search`, `execute`

**Responsibility**: Full pipeline from source comprehension to page creation:

| Step | Action | Description |
|---|---|---|
| 0 | Auto-scan | List `wiki/sources/` and `wiki/synthesis/`, compare against `raw/` and `questions_approved/` by slug — process anything new without asking |
| 1 | Read source | Read the raw file completely, preserving all code blocks |
| 2 | Extract | Identify title, slug, summary, key takeaways, entities, concepts |
| 3 | Deduplicate | Read `wiki/index.md`, check if extracted slugs match existing pages |
| 4 | Write source page | Create `wiki/sources/<slug>.md` with summary, takeaways, entity/concept lists |
| 5 | Write/update entities | Create new entity pages OR append to existing ones |
| 6 | Write/update concepts | Create new concept pages OR append to existing ones |
| 7 | Update index | Add new entries to `wiki/index.md` in alphabetical order |
| 8 | Append log | Record the operation in `wiki/log.md` |

**Input types**:
- *(no argument)* — auto-scan all three source directories and process everything new (Step 0)
- `raw/<filename>` — process a specific source document
- `"scan"` — alias for no-argument auto-scan
- `questions_approved/<filename>` — create a synthesis page
- `lint_approved/<filename>` — apply approved corrections to wiki pages
- `--RESET-ALL` — wipe all wiki content, preserving empty structure

**Key constraints**:
- ALL content in English (translate non-English sources)
- ZERO information loss — copy full paragraphs, preserve code blocks verbatim
- When unsure about slug deduplication, create a NEW page (duplicates are cheaper than wrong merges)
- Entity vs concept distinction must be respected (never both)
- Each new source can touch many existing pages (updates, new connections, contradictions)

---

### 5.2 Wiki Querier (`@wiki-querier`)

**Purpose**: A retrieval-only system that answers questions exclusively from wiki content, navigating pages like a researcher.

**Tools**: `read`, `edit`

**Responsibility**: Navigate the wiki, gather evidence, compose an answer with citations, save the result.

**Workflow**:

| Step | Action |
|---|---|
| 1 | Read `wiki/index.md` to see all available pages |
| 2 | Identify and read the most relevant page |
| 3 | Examine `## Connections` section for related pages |
| 4 | Follow links, reading pages one at a time |
| 5 | Repeat until minimum reads (3) met and enough context gathered, or budget (8) hit |
| 6 | Compose answer by quoting/closely paraphrasing, cite with `[[wikilinks]]` |
| 7 | Save answer to `questions_pending/<slug>_<timestamp>.md` |
| 8 | Append to `wiki/log.md` |

**Why `edit` tool?** — Used ONLY for saving the answer file and appending the log entry. The querier NEVER modifies wiki pages.

**Key constraints**:
- Page budget: minimum 3, maximum 8 pages per question
- If a fact is not in a wiki page → do not include it
- If the wiki doesn't cover the topic → say so explicitly
- No training data, no synthesis, no inference beyond what's written
- Answer in the user's language, but cite with standard `[[category/slug]]` format
- Code blocks from wiki pages must be included verbatim, never shortened

---

### 5.3 Wiki Linter (`@wiki-linter`)

**Purpose**: Validates wiki health and integrity through a two-phase check — deterministic validation (objective, pass/fail) followed by semantic analysis (requires LLM reasoning).

**Tools**: `read`, `search`, `execute`

**Responsibility**: Detect structural and semantic issues, produce an actionable report.

#### Phase 1 — Deterministic Checks (via terminal)

These are objective validations run via terminal commands:

| Check | What it detects |
|---|---|
| Broken wikilinks | `[[category/slug]]` references where the target `.md` file doesn't exist |
| Orphan pages | Pages not referenced by any other page and not in the index |
| Missing frontmatter | Pages lacking required fields (`title`, `type`, `created`, `updated`, `sources`) |
| Empty sections | Section headers with no content below them |

#### Phase 2 — Semantic Checks (LLM reasoning)

These require understanding and cross-page comparison:

| Check | What it detects |
|---|---|
| Contradictions | Claims in one page that conflict with claims in another |
| Stale claims | Information from older sources that newer sources may have superseded |
| Missing pages | Entity/concept names mentioned in text but lacking a dedicated page |
| Missing cross-references | Pages that discuss related topics but don't link to each other |
| Suggested questions | Knowledge gaps that a new source or research could fill |

**Output**: A structured markdown report saved to `lint_pending/lint-report_<timestamp>.md` with severity levels (🔴 high, 🟡 medium, 🟢 low) and suggested fixes for each issue.

**Key constraints**:
- Phase 2 must NOT repeat Phase 1 checks
- The linter never modifies wiki pages — it only reports
- Issues include actionable suggested fixes (create page, add link, flag contradiction)

---

## 6. Instructions & Prompts

### 6.1 Schema Instructions

**File**: `.github/instructions/wiki-schema.instructions.md`

A shared instruction file that defines wiki page format conventions. All agents inherit these rules via the `applyTo` mechanism. It covers:

- Directory layout
- Frontmatter requirements (all pages)
- Page body templates (source, entity, concept, synthesis)
- Wikilink format
- Index and log format
- Contradiction block format

This ensures consistent page structure regardless of which agent creates or reads a page.

### 6.2 Prompt Templates

Three user-facing entry points that delegate to the correct agent:

| Prompt | File | Agent | User Input |
|---|---|---|---|
| `/wiki-ingest` | `.github/prompts/wiki-ingest.prompt.md` | wiki-ingestor | Empty (recommended — auto-scan all sources) or advanced-only: specific file path / `--RESET-ALL` |
| `/wiki-query` | `.github/prompts/wiki-query.prompt.md` | wiki-querier | A question |
| `/wiki-lint` | `.github/prompts/wiki-lint.prompt.md` | wiki-linter | (none — runs full check) |

Prompts are minimal — they provide the user's input and a one-line instruction. The agent's `.agent.md` file contains the complete behavioral specification. Their sole purpose is UX: prompt files appear in the VS Code Copilot command palette under the `/` syntax, making the wiki operations discoverable without requiring the user to know the `@agent-name` syntax.

### 6.3 Composition

```
User types "/wiki-query What is an Agent?"
    │
    ▼
wiki-query.prompt.md (passes question to agent)
    │
    ▼
wiki-querier.agent.md (full behavioral instructions)
    │
    ├── inherits: wiki-schema.instructions.md (format rules)
    ├── uses tools: read, edit
    └── reads: wiki/index.md → wiki pages → ## Connections → answer
```

---

## 7. Data Flow & Human-Gated Compounding

### 7.1 Ingest Flow

```
raw/<filename>.md ──→ @wiki-ingestor ──→ wiki/sources/<slug>.md
                                          wiki/entities/<slug>.md (new or updated)
                                          wiki/concepts/<slug>.md (new or updated)
                                          wiki/index.md (updated)
                                          wiki/log.md (appended)
```

### 7.2 Query Flow

```
User question ──→ @wiki-querier ──→ questions_pending/<slug>_<timestamp>.md
                                     wiki/log.md (appended)
```

### 7.3 Lint Flow

```
@wiki-linter ──→ lint_pending/lint-report_<timestamp>.md
                  wiki/log.md (appended)
```

### 7.4 Human-Gated Compounding Loop

```
questions_pending/ ──[human review]──→ questions_approved/ ──→ @wiki-ingestor ──→ wiki/synthesis/
lint_pending/      ──[human review]──→ lint_approved/      ──→ @wiki-ingestor ──→ wiki pages updated
```

This is the compounding mechanism: the wiki's own answers and corrections feed back into it as first-class content, but only after human validation. This prevents hallucination propagation while enabling the wiki to grow from its own intellectual activity.

---

## 8. Gap Analysis

A detailed gap analysis comparing our implementation against Karpathy's original spec is maintained in [BACKLOG_KARPATHY_GAPS.md](BACKLOG_KARPATHY_GAPS.md).

**Current status**: 8 of 28 identified gaps completed, 20 pending.

**Key completed items**:
- Parseable log format with timestamps and pages touched
- Query and lint operations logged
- Wiki reset workflow
- Index with one-liner summaries
- English-only content rules
- Query agent sequential navigation workflow

**Key pending items** (by priority):
- P1: Lint detection of new contradictions, stale claims, missing pages
- P1: Lint remediation (auto-fixing broken links, applying semantic suggestions)
- P2: Human-in-the-loop confirmation before writing
- P2: Git auto-commit after operations
- P3: Image handling, web search in lint, BM25/vector search upgrade

---

## 9. Design Decisions

| Decision | Rationale |
|---|---|
| **Monolithic ingestor** | A single agent handles extraction + dedup + writing. Splitting into sub-agents adds complexity without clear benefit at this scale. |
| **Read-only querier** | The querier NEVER modifies the wiki. It saves answers to `questions_pending/` for human review. This prevents contamination of the knowledge base with unverified content. |
| **2-phase linter** | Deterministic checks (broken links, orphans) are fast and objective — no LLM needed. Semantic checks (contradictions, stale claims) require reasoning. Separating them avoids wasting LLM calls on things grep can catch. |
| **English-only wiki** | Consistent language enables cross-source comparison and concept merging. The user interface remains multilingual. |
| **No Python code** | The entire system runs via Copilot's native capabilities. This eliminates dependency management, environment setup, and API key configuration for the wiki runtime. |
| **Schema via instructions file** | Format rules live in `.instructions.md` (inherited automatically) rather than enforced by post-hoc hooks. This is simpler and leverages Copilot's native `applyTo` mechanism. |
| **Dedup prefers NEW** | When slug matching is ambiguous, creating a new page is always safer than merging with the wrong existing page. Duplicates are cheap to merge later; incorrect merges corrupt information. |
| **Human gating** | Auto-generated content (answers, lint fixes) always lands in `*_pending/` first. This prevents hallucination propagation and gives the user editorial control over what enters the knowledge base. |
| **Index as navigation hub** | Agents read `index.md` first to understand what exists. This avoids expensive directory traversals and gives agents a content-oriented view (with one-liner summaries) rather than a raw file listing. |
| **Directory-listing idempotency** | The ingestor compares slug sets between input directories (`raw/`, `questions_approved/`) and output directories (`wiki/sources/`, `wiki/synthesis/`) to detect new files. O(1) per file — no log parsing, no frontmatter reads. |

---

## References

- [Karpathy's LLM Wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — Original concept
- [BOOTSTRAP.md](../BOOTSTRAP.md) — Implementation plan
- [schema.md](../schema.md) — Wiki page format reference
- [BACKLOG_KARPATHY_GAPS.md](BACKLOG_KARPATHY_GAPS.md) — Gap analysis vs Karpathy spec
- [PIPELINE_WIKI_LLM.md](PIPELINE_WIKI_LLM.md) — Python pipeline architecture (reference implementation)
