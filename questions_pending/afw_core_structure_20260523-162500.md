---
title: "Strutturazione della directory afw_core"
type: "query-answer"
query: "mi serve proprio la strtturazione della directory afw_core"
created: "2026-05-23"
pages_read: ["concepts/factory-pattern-for-agents","concepts/agent-definitions","concepts/llm-client-factories","wiki/index"]
---

# Strutturazione della directory `afw_core`

La wiki descrive una convenzione modulare per organizzare `afw_core`. Principi chiave:

- Ogni agent risiede in un proprio modulo sotto `afw_core/agents/` e espone una factory `create_agent()`; `name` e `instructions` sono parte dell'identità dell'agent mentre `client`, `options` e `tools` sono iniettati come infrastruttura. [[concepts/factory-pattern-for-agents]]

Esempio di moduli agent (verbatim):

```
afw_core/agents/
  backlog_reader.py      ← create_agent() factory
  jira_executor.py       ← create_agent() factory
```

- I client LLM seguono la convenzione `create_client()` e risiedono sotto `afw_core/llms/` (uno per provider). Esempio (verbatim):

```python
# afw_core/llms/openai.py
from agent_framework.openai import OpenAIChatClient

def create_client(api_key, model="gpt-4o-mini"):
    """Create an OpenAI chat client."""
    return OpenAIChatClient(api_key=api_key, model=model)
```

- Tool condivisi e proxy MCP risiedono in pacchetti separati, impliciti nella convenzione:
  - `afw_core/tools/` — tool condivisi come `read_file`, `write_file`.
  - `afw_core/mcps/` — proxy MCP per tool esterni. [[concepts/factory-pattern-for-agents]] [[concepts/llm-client-factories]]

Esempio di entrypoint di orchestrazione (verbatim):

```python
from afw_core.llms.openai import create_client
from afw_core.mcps.jira import create_proxy
from afw_core.tools.file_reader import read_file
from afw_core.tools.file_writer import write_file
from afw_core.agents.backlog_reader import create_agent as create_reader_agent
from afw_core.agents.jira_executor import create_agent as create_executor_agent

async def main():
    client, options = create_client(api_key=os.environ["OPENAI_API_KEY"], model="gpt-4o-mini")
    jira_proxy = create_proxy()

    reader = create_reader_agent(client=client, options=options, tools=[read_file])
    executor = create_executor_agent(client=client, options=options, tools=[jira_proxy, write_file])

    response = await reader.run("Read the file 'backlog.md'")
    # ... validate, then pass to executor
```

Fonti: [[concepts/factory-pattern-for-agents]], [[concepts/agent-definitions]], [[concepts/llm-client-factories]]
