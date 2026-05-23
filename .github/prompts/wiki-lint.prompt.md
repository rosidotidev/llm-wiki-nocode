---
description: "Run a full health check on the wiki (broken links, orphans, contradictions, etc.)."
agent: "wiki-linter"
---

# Wiki Lint

Run a full wiki health check. Execute deterministic validations first (broken wikilinks, orphan pages, missing frontmatter), then perform semantic analysis (contradictions, stale claims, missing pages, missing cross-references). Save the report to lint_pending/.
