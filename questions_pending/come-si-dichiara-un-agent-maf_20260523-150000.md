---
title: "Come si dichiara un agent MAF?"
type: "query-answer"
query: "come si dichiara un agent MAF?"
created: "2026-05-23"
pages_read: ["concepts/agent-definitions", "entities/microsoft-agent-framework", "concepts/chat-clients"]
---

# Come si Dichiara un Agent in MAF?

In [[concepts/agent-definitions]], un **Agent** nel Microsoft Agent Framework si dichiara fornendo 5 parametri:

```python
Agent(
    client=...,              # LLM provider (required)
    name="...",              # Identity for logging (required)
    instructions="...",      # System prompt (required)
    tools=[...],             # Optional: function tools and/or MCP proxies
    default_options={...},   # Optional: temperature, max_tokens, etc.
)
```

## Metodo Rapido: `as_agent()`

Se usi un client chat, puoi usare il shorthand [[concepts/agent-definitions]]:

```python
agent = OpenAIChatClient().as_agent(
    name="HelloAgent",
    instructions="You are a friendly assistant.",
)
```

## Client LLM Disponibili

Da [[concepts/chat-clients]]:

| Client | Provider |
|--------|----------|
| `OpenAIChatClient` | OpenAI (Responses API) |
| `OpenAIChatCompletionClient` | OpenAI / Azure OpenAI / local |
| `FoundryChatClient` | Azure Foundry |
| `AnthropicChatClient` | Anthropic Claude |
| `OllamaChatClient` | Ollama (local) |

## Runtime Loop

Da [[entities/microsoft-agent-framework]], tutti gli agent seguono un **deterministic runtime loop**:
1. Invia message + instructions + tool definitions all'LLM
2. Se l'LLM chiama un tool → esegui il tool → torna il risultato
3. Ripeti finché l'LLM produce una risposta finale
4. Ritorna l'`AgentResponse`
