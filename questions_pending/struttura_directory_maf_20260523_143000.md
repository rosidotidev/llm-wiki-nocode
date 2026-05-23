---
title: "Struttura directory di un progetto con MAF"
type: "query-answer"
query: "qual è la struttura a directory di un progetto con MAF"
created: "2026-05-23"
pages_read: ["sources/microsoft-agent-framework-agents-python-howto", "entities/microsoft-agent-framework", "concepts/factory-pattern-for-agents", "concepts/llm-client-factories", "concepts/multi-agent-orchestration"]
---

# Struttura directory di un progetto con MAF

La struttura consigliata per un progetto con il Microsoft Agent Framework segue il pattern factory con moduli separati:

```
afw_core/
  agents/
    backlog_reader.py      ← create_agent() factory
    jira_executor.py       ← create_agent() factory
    ...                    ← altri agenti
  llms/
    openai.py              ← OpenAI client factory
    azure.py               ← Azure Foundry client factory
    ...                    ← altri provider
  mcps/
    jira.py                ← MCP proxy per Jira
    ...                    ← altri MCP servers
  tools/
    file_reader.py         ← funzioni strumento personalizzate
    file_writer.py         ← funzioni strumento personalizzate
    ...                    ← altri tool
  models/
    backlog.py             ← modelli Pydantic per validazione dati
    ...                    ← altri modelli
```

## Principi di Organizzazione

### Agents (afw_core/agents/)
Secondo [[concepts/factory-pattern-for-agents]], ogni agente vive nel suo modulo e espone una funzione factory `create_agent(client, options, tools)`. Nome e istruzioni sono hardcodati (identità intrinseca dell'agente), mentre client, options e tools sono iniettati (infrastruttura).

### LLM Clients (afw_core/llms/)
Secondo [[concepts/llm-client-factories]], ogni provider LLM (OpenAI, Azure, Anthropic, etc.) ottiene il suo modulo con una factory `create_client()`. Questo centralizza l'inizializzazione client, le credenziali e le opzioni provider-specifiche.

### Tools e MCP (afw_core/tools/ e afw_core/mcps/)
I tool personalizzati e i proxy MCP sono organizzati in moduli separati e importati dagli agenti durante la creazione.

### Data Models (afw_core/models/)
Secondo [[concepts/multi-agent-orchestration]], modelli Pydantic sono utilizzati per validazione dati tra agenti e definire contratti inter-agente.
