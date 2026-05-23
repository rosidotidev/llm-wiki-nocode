---
title: "Come si aggancia un tool a un agent e dove si dichiara?"
type: "query-answer"
query: "come si aggancia un tool a un agent e dove si dichiara?"
created: "2026-05-23"
pages_read: ["concepts/agent-definitions", "concepts/function-tools", "concepts/mcp-integration"]
---

# Come Si Aggancia un Tool a un Agent e Dove Si Dichiara

## Risposta

Un tool si aggancia a un agent passandolo nel parametro `tools=[]` al momento della costruzione dell'Agent. Esistono due tipi principali di tool, ciascuno dichiarato diversamente.

### 1. Function Tools — Dichiarazione con `@tool`

I function tools sono funzioni Python decorati con `@tool`. Si dichiarano sopra la funzione:

```python
from typing import Annotated
from agent_framework import tool
from pydantic import Field

@tool
def get_weather(
    location: Annotated[str, Field(description="The city name to get weather for")],
) -> str:
    """Get the weather for a given location."""
    return f"The weather in {location} is cloudy with a high of 15°C."
```

Poi si aggancia all'agent passandolo nel parametro `tools`:

```python
agent = Agent(
    client=client,
    name="WeatherAgent",
    instructions="You are a weather assistant.",
    tools=[get_weather],  # Qui si aggancia il tool
)
```

Secondo [[concepts/function-tools]], il decorator `@tool` accetta parametri opzionali come `name` e `description`. I parametri della funzione devono essere documentati con `Annotated[type, Field(description="...")]` per esporre lo schema all'LLM.

### 2. MCP Tools — Dichiarazione con `MCPStdioTool`

Per tool esterni (ad es. Jira via MCP — Model Context Protocol), si dichiara un proxy MCP:

```python
from agent_framework import MCPStdioTool

jira_proxy = MCPStdioTool(
    name="jira_server",
    command="pipenv",
    args=["run", "mcp-atlassian"],
    env={"JIRA_URL": "...", "JIRA_USERNAME": "...", "JIRA_API_TOKEN": "..."},
)

agent = Agent(
    name="JiraAgent",
    instructions="You are a Jira assistant.",
    client=client,
    tools=[jira_proxy],  # Qui si aggancia il tool MCP
)
```

Secondo [[concepts/mcp-integration]], il framework auto-discover tutti i tool esposti dal server MCP. **Importante**: sempre chiudere il proxy in un blocco `finally`: `await jira_proxy.close()`.

### 3. Combinare Entrambi i Tipi

Un agent può usare sia function tools che MCP tools contemporaneamente:

```python
agent = Agent(
    name="JiraExecutor",
    instructions="Create issues and write summaries.",
    client=client,
    tools=[jira_proxy, get_weather, write_file],  # Mix di tool types
)
```

### 4. Override Per-Run

Si possono anche override o estendere i tool al momento della chiamata `run()`:

```python
result = await agent.run("Divide 10 by 3", tools=[tools.divide])
```

## Riepilogo

Secondo [[concepts/agent-definitions]], l'Agent class prende i tool nel parametro `tools=[]`. Dove si dichiarano:
- **Function tools**: decorator `@tool` sopra la funzione
- **MCP tools**: istanza di `MCPStdioTool`

Entrambi si passano come lista nel parametro `tools` del costruttore Agent.
