# LLM Wiki: Copilot → Claude Code Mapping

This document maps every artifact in the Copilot implementation to its Claude Code equivalent. It serves as the blueprint for the port.

---

## File & Directory Mapping

| Copilot Artifact | Location | Claude Code Equivalent | Location | Notes |
|---|---|---|---|---|
| **Agents** | `.github/agents/` | **Slash Commands** | `.claude/commands/` | 3 agents → 3 commands |
| `wiki-ingestor.agent.md` | `.github/agents/` | `/wiki-ingest` | `.claude/commands/wiki-ingest.md` | Full agent behavior becomes command content |
| `wiki-querier.agent.md` | `.github/agents/` | `/wiki-query` | `.claude/commands/wiki-query.md` | Full agent behavior becomes command content |
| `wiki-linter.agent.md` | `.github/agents/` | `/wiki-lint` | `.claude/commands/wiki-lint.md` | Full agent behavior becomes command content |
| **Instructions** | `.github/instructions/` | **Project Instructions** | `.claude/CLAUDE.md` | Shared format rules |
| `wiki-schema.instructions.md` | `.github/instructions/` | `.claude/CLAUDE.md` | `.claude/` directory | Schema + format conventions |
| **Prompts** | `.github/prompts/` | **N/A** | **N/A** | Prompt logic merged into slash commands |
| `wiki-ingest.prompt.md` | `.github/prompts/` | Merged into `/wiki-ingest` | `.claude/commands/wiki-ingest.md` | User-facing description + input spec |
| `wiki-query.prompt.md` | `.github/prompts/` | Merged into `/wiki-query` | `.claude/commands/wiki-query.md` | User-facing description + input spec |
| `wiki-lint.prompt.md` | `.github/prompts/` | Merged into `/wiki-lint` | `.claude/commands/wiki-lint.md` | User-facing description + input spec |

---

## Functional Requirements Mapping

Every functional requirement from the Copilot spec is preserved in Claude Code:

| Requirement | Copilot Location | Claude Code Location | Implementation |
|---|---|---|---|
| **FR-1: Cumulative integration** | `wiki-ingestor.agent.md` Step 5–6 | `/wiki-ingest` command | Append-to-existing logic preserved |
| **FR-2: Cross-referencing** | `wiki-schema.instructions.md` | `.claude/CLAUDE.md` | Wikilink format enforced |
| **FR-3: Contradiction detection** | `wiki-linter.agent.md` Phase 2.1 | `/wiki-lint` command | Semantic check section |
| **FR-4: Retrieval-only querying** | `wiki-querier.agent.md` | `/wiki-query` command | RETRIEVAL-ONLY constraint replicated |
| **FR-5: Knowledge compounding** | `wiki-ingestor.agent.md` Step 9–10 | `/wiki-ingest` command | Questions_approved / lint_approved flows |
| **FR-6: Human gating** | `wiki-ingestor.agent.md` Step 0 | `/wiki-ingest` command | Pending/approved directory logic |
| **FR-7: Evergreen content** | `wiki-schema.instructions.md` | `.claude/CLAUDE.md` | Updated dates, source tracking |
| **FR-8: Zero information loss** | `wiki-ingestor.agent.md` Step 2 | `/wiki-ingest` command | "Copy full paragraphs, preserve code blocks verbatim" |
| **FR-9: English-only content** | `wiki-ingestor.agent.md` | `/wiki-ingest` command | Translation requirement enforced |

---

## Tool Mapping

Copilot native tools → Claude Code equivalents:

| Copilot Tool | Claude Code Equivalent | Usage in Commands | Notes |
|---|---|---|---|
| `read` (read_file) | `Read` tool | All three commands | Read wiki files, index, schema |
| `read` (list_dir) | `Bash` tool + `ls` | `/wiki-ingest` (auto-scan) | Directory listing for dedup logic |
| `edit` (create_file) | `Write` tool | `/wiki-ingest`, `/wiki-query` | Create new wiki pages, answer files |
| `edit` (replace_string_in_file) | `Edit` tool | `/wiki-ingest`, `/wiki-query` | Update existing pages, append log |
| `search` (grep_search) | `Grep` tool | `/wiki-linter` Phase 1 | Extract wikilinks, find orphans |
| `execute` (run_in_terminal) | `Bash` tool | `/wiki-linter` Phase 1 | Terminal validation (find, grep) |

---

## Workflow & Architecture Mapping

| Copilot Concept | Claude Code Equivalent |
|---|---|
| User runs `/wiki-ingest` command | User invokes `/wiki-ingest` command via slash menu |
| Agent reads frontmatter via `read` tool | Command uses `Read` tool to parse frontmatter |
| Agent auto-scans `raw/` directory | Command uses `Bash` with `ls` / `find` to list files |
| Agent deduplicates via `search` (grep) | Command uses `Grep` to search `wiki/index.md` |
| Agent creates/updates pages via `edit` | Command uses `Write` for new, `Edit` for updates |
| Agent appends log via `edit` | Command uses `Edit` to append to `wiki/log.md` |
| User promotes answer to `questions_approved/` | User moves file; command detects in auto-scan |
| Linter runs deterministic checks via `execute` | Linter command uses `Bash` for terminal checks |
| Linter performs semantic checks | Linter command uses native LLM reasoning |
| Report saved via `edit` | Linter command uses `Write` for report file |

---

## Directory Structure (No Changes)

The wiki's directory structure remains identical:

```
<project-root>/
├── wiki/
│   ├── index.md
│   ├── log.md
│   ├── sources/
│   ├── entities/
│   ├── concepts/
│   └── synthesis/
├── raw/
├── questions_pending/
├── questions_approved/
├── lint_pending/
├── lint_approved/
├── docs/
└── .github/  ← BEING REPLACED BY .claude/
```

---

## Implementation Gaps & Workarounds

| Gap | Issue | Workaround |
|---|---|---|
| **No `applyTo` in Claude Code** | Copilot's `.instructions.md` can be inherited by agents. Claude Code has no equivalent. | Move schema instructions into `.claude/CLAUDE.md` at project level. Commands reference it explicitly in their descriptions. |
| **No built-in grep_search** | Copilot's `search` tool is high-level. Claude Code has `Grep` (ripgrep). | `Grep` is more powerful; use regex patterns from linter requirements. Minor syntax differences only. |
| **No built-in execute** | Copilot's `execute` runs terminal. Claude Code has `Bash`. | `Bash` is equivalent; PowerShell commands from Copilot agent translate to bash for compatibility. |
| **Manual auto-scan** | Copilot's agents can run auto-scan from the background. Claude Code slash commands are user-triggered. | User explicitly invokes `/wiki-ingest` (no argument) for auto-scan, or system uses a scheduled task if background automation is needed. |
| **Prompts merged into commands** | Copilot has separate `.prompt.md` files. Claude Code slash commands combine description + behavior. | User-facing description becomes command frontmatter; agent instructions become command body. |

---

## Summary: Complete 1:1 Parity

✅ **3 Copilot agents** → **3 Claude Code slash commands**  
✅ **1 shared instructions file** → **1 `.claude/CLAUDE.md` + command descriptions**  
✅ **3 user-facing prompts** → **3 slash commands** (with integrated descriptions)  
✅ **All tools** → **Claude Code equivalents**  
✅ **All workflows** → **Exact replication**  
✅ **All functional requirements** → **Preserved 1:1**  

---

## Next Steps

1. **STEP 2**: Draft `docs/SPEC_LLM_WIKI_CC.md` (mirror of GHB spec, written for Claude Code)
2. **STEP 3**: Create `.claude/CLAUDE.md` (project instructions)
3. **STEP 4**: Implement `.claude/commands/wiki-ingest.md`, `wiki-query.md`, `wiki-lint.md`
4. **STEP 5**: Final review (confirm every mapped item is implemented)
