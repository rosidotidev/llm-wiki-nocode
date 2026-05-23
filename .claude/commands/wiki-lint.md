---
name: "wiki-lint"
description: "Run a comprehensive health check on the wiki for broken links, orphaned pages, and inconsistencies."
usage: "/wiki-lint"
trigger: "no_args"
---

# Wiki Lint Command

## Execution

When invoked (no arguments required):
1. Execute Phase 1 deterministic checks first (use Bash for speed)
2. Only proceed to Phase 2 semantic checks after Phase 1 is complete
3. Save complete report to `lint_pending/lint-report-[YYYY-MM-DD-HHmm].md`
4. Append summary to `wiki/log.md`
5. Display report with severity levels and actionable fixes

Run a full wiki health check. Execute deterministic validations first (broken wikilinks, orphan pages, missing frontmatter), then perform semantic analysis (contradictions, stale claims, missing pages, missing cross-references). Save the report to lint_pending/.

## Phase 1 — Deterministic Checks (via Bash)

Run these checks using the **Bash** tool. These are objective, pass/fail validations.

### 1.1 Broken Wikilinks

Find all `[[category/slug]]` references across wiki pages and verify each target file exists:

```bash
grep -roh "\[\[[^]]*\]\]" wiki/ | sort -u
```

For each extracted wikilink `[[category/slug]]`, verify `wiki/category/slug.md` exists using **Read** or **Glob**.

### 1.2 Orphan Pages

Find pages that are:
- NOT referenced by any other wiki page
- NOT listed in `wiki/index.md`

```bash
find wiki/ -name "*.md" ! -name "index.md" ! -name "log.md"
```

For each page, search for backlinks using **Grep**.

### 1.3 Missing Frontmatter

Every `.md` file in `wiki/` MUST have YAML frontmatter with: `title`, `type`, `created`, `updated`, `sources`.

```bash
find wiki/ -name "*.md" -exec grep -L "^---" {} \;
```

### 1.4 Invalid Frontmatter

Check for:
- Missing required fields
- Invalid `type` values (must be: `source`, `entity`, `concept`, `synthesis`)
- Malformed dates (must be YYYY-MM-DD)
- Missing `sources` array

## Phase 2 — Semantic Checks

After deterministic checks pass, perform manual semantic analysis:

### 2.1 Contradictions

Read pages and check for:
- Conflicting claims about the same entity/concept
- Definition disagreement across pages
- Outdated information (check dates)

### 2.2 Stale Claims

- Are timestamps realistic? (e.g., `created: 2025-01-01` but should be earlier?)
- Has the source been updated but the wiki page hasn't?

### 2.3 Missing Pages

- Do entities/concepts mentioned in pages have their own pages?
- Are there references to entities that should exist but don't?

### 2.4 Missing Cross-References

- Should pages link to each other but don't?
- Are related entities/concepts connected via wikilinks?

## Report Format

Save to `lint_pending/lint-report-[YYYY-MM-DD-HHmm].md`:

```yaml
---
title: "Wiki Lint Report"
type: "lint_report"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
---

# Wiki Lint Report — [Date]

## Phase 1: Deterministic Checks

### 🟢 Broken Wikilinks
- All wikilinks verified. No broken links found.

### 🟡 Orphan Pages
- `wiki/entities/foo.md` — not referenced in any page. Recommended: link from [[concept/bar]] or delete.

### 🟢 Missing Frontmatter
- All pages have valid frontmatter.

### 🟢 Invalid Frontmatter
- All frontmatter fields valid.

## Phase 2: Semantic Checks

### 🟡 Contradictions
- Page `entities/foo.md` claims "X", but `concepts/bar.md` claims "not X". Needs clarification.

### 🟡 Stale Claims
- `entities/foo.md` (updated 2024-03-15) mentions "latest version is 1.2" — check if this is still true.

### 🔴 Missing Pages
- Concept `machine-learning` is mentioned in 3 pages but has no dedicated page. Recommendation: create `concepts/machine-learning.md`.

### 🟡 Missing Cross-References
- `entities/pytorch.md` mentions "TensorFlow" but doesn't link it. Recommendation: add [[entities/tensorflow]] wikilink.

## Actionable Fixes

For each issue, provide:
1. **File**: Path to the page
2. **Fix**: Specific action (edit, create, delete, link)
3. **Severity**: 🔴 (critical), 🟡 (warning), 🟢 (info)

## Append to Log

Also edit `wiki/log.md` to append:
```
## [YYYY-MM-DD HH:MM] Lint Check Completed
- Report saved to: `lint_pending/lint-report-[timestamp].md`
- Issues found: [count]
- Critical (🔴): [count]
- Warnings (🟡): [count]
- Info (🟢): [count]
```
