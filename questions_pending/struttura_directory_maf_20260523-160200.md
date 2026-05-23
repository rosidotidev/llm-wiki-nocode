---
title: "Struttura directory di un progetto MAF"
type: "query-answer"
query: "come è la struttura delle directory in un progetto maf?"
created: "2026-05-23"
pages_read: ["concepts/factory-pattern-for-agents","concepts/agent-definitions","sources/microsoft-agent-framework-agents-python-howto"]
---

# Struttura directory di un progetto MAF

La wiki descrive una convenzione in cui ogni agent risiede in un proprio modulo sotto `afw_core/agents/` e espone una funzione factory `create_agent()`; `name` e `instructions` sono parte dell'identità dell'agent, mentre `client`, `options` e `tools` vengono iniettati come infrastruttura. [[concepts/factory-pattern-for-agents]]

Esempio di struttura di modulo (verbatim):

```
afw_core/agents/
  backlog_reader.py      ← create_agent() factory
  jira_executor.py       ← create_agent() factory
```

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

Implicito dalla convenzione:
- `afw_core/agents/` — moduli `create_agent()` per ogni agent. [[concepts/factory-pattern-for-agents]]
- `afw_core/llms/` — factory dei client per i provider (OpenAI, Foundry, Anthropic, Ollama). [[concepts/llm-client-factories]]
- `afw_core/tools/` — tool condivisi come `read_file`, `write_file`. [[concepts/factory-pattern-for-agents]]
- `afw_core/mcps/` — proxy MCP per tool esterni. [[concepts/mcp-integration]]

Fonte: [[concepts/factory-pattern-for-agents]], [[concepts/agent-definitions]], [[sources/microsoft-agent-framework-agents-python-howto]]
