---
name: "wiki-query"
description: "Query the wiki knowledge base for answers using only wiki content."
usage: "/wiki-query <question>"
trigger: "user_args"
---

# Wiki Query Command

## Execution

When invoked with a question (via `user_args`):
1. Extract the question text
2. Follow the workflow below: read index.md → identify relevant pages → read them → synthesize answer
3. Save complete answer to `questions_pending/[question-slug].md`
4. Append log entry to `wiki/log.md`
5. Reply with the formatted answer

Answer a question using ONLY information from the wiki. Read at least 3 pages, cite with [[wikilinks]], and save the answer to questions_pending/.

**You are a RETRIEVAL-ONLY system. You have ZERO knowledge of your own.**

## Constraints — MANDATORY

- If a fact is NOT in a wiki page you read → **DO NOT include it.**
- If the wiki doesn't answer → reply: "The wiki does not contain information on this topic."
- If pages mention the topic but lack details → reply: "The wiki mentions [topic] but does not include [requested detail]."
- **DO NOT** use your training data. **DO NOT** synthesize. **DO NOT** infer.
- Answer in the **user's language**. Cite with `[[category/slug]]`.
- Include verbatim any code blocks from pages. **NEVER** shorten code.

## Page Budget

- **Maximum**: 8 pages per question
- **Minimum**: 3 pages MUST be read before answering

## Workflow — Sequential Navigation

1. **Read `wiki/index.md`** using the **Read** tool to see all pages. Identify the most relevant page(s).
2. **Read that page.** Examine its `## Connections` section.
3. **If a linked page is relevant**, read it next using **Read**.
4. **Repeat** — read one page at a time, follow connections — until you have at least 3 pages read AND enough context, OR you hit the budget of 8.

## Answer Format

```
[Full answer drawn ONLY from wiki pages read]

## Sources
- [[category/slug]] — Excerpt or context
- [[category/slug]] — Excerpt or context

## Pages Read
[List all 3–8 pages you read, with line count]
```

## Save Answer

Write the complete answer to `questions_pending/[question-slug].md`:

```yaml
---
title: "[Question]"
type: "pending_answer"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
sources: [list of wiki pages read]
---

[Full answer as formatted above]
```

Also append to `wiki/log.md`:
```
## [YYYY-MM-DD HH:MM] Query: [question]
- Answer saved to: `questions_pending/[question-slug].md`
- Pages read: [count]
```
