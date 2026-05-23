---
title: "Come si dichiara un agent con MAF - sintassi base"
type: "query-answer"
query: "come si dichiara un agent con maf,solo sintassi base"
created: "2026-05-23"
pages_read: ["concepts/agent-definitions", "sources/microsoft-agent-framework-agents-python-howto", "concepts/chat-clients"]
---

# Come si dichiara un agent in MFA — Sintassi Base

## Risposta

Un agent in MFA si dichiara con la classe `Agent`, passando quattro parametri principali: client, name, instructions, e opzionalmente tools e default_options.

### Forma completa

```python
from agent_framework import Agent
from agent_framework.openai import OpenAIChatClient

client = OpenAIChatClient(api_key="...", model="gpt-4o-mini")

agent = Agent(
    client=client,
    name="ManagerAgent",
    instructions="You are a manager agent. Answer the user's query as accurately and concisely as possible.",
)

result = await agent.run("What is a Large Language Model?")
```

Secondo [[concepts/agent-definitions]], il `Agent` class è il primitivo più importante: riceve **name** (identità per logging), **instructions** (system prompt), **client** (LLM provider), **tools** (opzionali), e **default_options** (impostazioni provider-specifiche).

### Forma shorthand (più veloce)

I client chat forniscono il metodo `as_agent()` per creare un agent in un unico passaggio:

```python
from agent_framework.openai import OpenAIChatClient

agent = OpenAIChatClient().as_agent(
    name="HelloAgent",
    instructions="You are a friendly assistant. Keep your answers brief.",
)
```

### Con modelli locali (Ollama)

Usa [[concepts/chat-clients]] per accedere a modelli locali via endpoint OpenAI-compatibile:

```python
from agent_framework.openai import OpenAIChatCompletionClient

client = OpenAIChatCompletionClient(
    base_url="http://127.0.0.1:62770/v1",
    model="phi-4-mini-instruct",
    api_key="none",
)

agent = Agent(client=client, name="LocalAgent", instructions="...")
```

### Parametri supportati

[[concepts/chat-clients]] espone per ogni client una TypedDict `Options` (es. `OpenAIChatCompletionOptions`) per l'IDE auto-complete. Opzioni comuni:
- `temperature` — Controls randomness (0.0 = deterministic, 1.0+ = creative)
- `max_tokens` — Maximum tokens to generate
- `response_format` — Structured output schema (Pydantic or JSON schema)

Le opzioni si impostano al momento della costruzione (`default_options`) oppure si override per-run nel metodo `run()`.

Tutti gli agent ereditano da una classe base comune (`AIAgent`) che fornisce un'interfaccia coerente per composizione, multi-agent orchestration e workflow integration [[sources/microsoft-agent-framework-agents-python-howto]].
