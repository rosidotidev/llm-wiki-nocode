---
description: "Ingest raw documents, approved questions, and approved lint fixes into the wiki."
agent: "wiki-ingestor"
---

# Wiki Ingest

Run the wiki ingest pipeline. The recommended usage is to leave the input empty: the agent will automatically scan all three source directories (`raw/`, `questions_approved/`, `lint_approved/`) and process anything not yet ingested — no questions asked.

Specifying a single file or a specific command is supported for advanced use only and is generally discouraged — prefer the auto-scan.

${input:source?Leave empty for auto-scan (recommended). Advanced only: specific file (e.g. raw/my-doc.md) or --RESET-ALL.}
