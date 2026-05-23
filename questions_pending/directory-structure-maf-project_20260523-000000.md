---
title: "Directory structure of a MAF project"
type: "query-answer"
query: "qual è la struttura delle directory di un progetto con maf?"
created: "2026-05-23"
pages_read: ["concepts/factory-pattern-for-agents", "concepts/llm-client-factories", "concepts/multi-agent-orchestration", "concepts/pydantic-validation-between-agents"]
---

# Qual è la struttura delle directory di un progetto con MAF?

Il wiki non descrive esplicitamente un albero di directory completo, ma dai percorsi di import nei code block emerge una struttura ricorrente in tutti gli esempi pratici. [[concepts/factory-pattern-for-agents]]

```
project-root/
│
├── main_<pipeline>.py              ← entry point: orchestrazione pura
│
└── afw_core/
    ├── agents/                     ← un modulo per agent, espone create_agent()
    │   ├── backlog_reader.py
    │   └── jira_executor.py
    │
    ├── llms/                       ← un modulo per provider LLM, espone create_client()
    │   ├── openai.py
    │   ├── azure.py
    │   └── foundry.py
    │
    ├── mcps/                       ← un modulo per MCP server, espone create_proxy()
    │   └── jira.py
    │
    ├── tools/                      ← function tools (@tool)
    │   ├── file_reader.py
    │   └── file_writer.py
    │
    └── models/                     ← contratti Pydantic inter-agent
        └── backlog.py
```

**`afw_core/agents/`** — ogni file ospita un singolo agent e la sua factory `create_agent(client, options, tools)`. Nome e instructions sono hardcoded (identità dell'agent); client, options e tools sono sempre iniettati. [[concepts/factory-pattern-for-agents]]

**`afw_core/llms/`** — ogni file configura un provider LLM ed espone `create_client(api_key, model)` che restituisce la tupla `(client, options)`. Il naming segue il nome del provider (`openai.py`, `azure.py`, `foundry.py`). [[concepts/llm-client-factories]]

**`afw_core/mcps/`** — factory per i proxy MCP (`create_proxy()`). [[concepts/multi-agent-orchestration]]

**`afw_core/tools/`** — funzioni decorate con `@tool` (es. `read_file`, `write_file`) importate dall'entry point e iniettate agli agent. [[concepts/factory-pattern-for-agents]]

**`afw_core/models/`** — modelli Pydantic che fungono da contratti tipizzati tra agent (es. `BacklogOutput`), validati con `model_validate_json()`. [[concepts/pydantic-validation-between-agents]]

**`main_<pipeline>.py`** — entry point che crea client/options, istanzia agent tramite le factory, e orchestra la sequenza di `agent.run()`. [[concepts/multi-agent-orchestration]]
