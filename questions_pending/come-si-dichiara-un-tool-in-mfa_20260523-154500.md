---
title: "Come si dichiara un tool in MFA"
type: "query-answer"
query: "come si dichiara un tool in MFA"
created: "2026-05-23"
pages_read: ["concepts/function-tools","concepts/agent-definitions","sources/microsoft-agent-framework-agents-python-howto"]
---

# Come si dichiara un tool in MFA

Per dichiarare un tool (Function Tool) nel Microsoft Agent Framework si usa il decorator `@tool` su una funzione Python, documentando i parametri con `Annotated` e `Field(description="...")`.

Esempio base (verbatim):

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

È possibile fornire metadata espliciti nel decorator (`name`, `description`):

```python
@tool(name="weather_tool", description="Retrieves weather information for any location")
def get_weather(
    location: Annotated[str, Field(description="The location to get weather for")],
) -> str:
    return f"The weather in {location} is cloudy with a high of 15°C."
```

Per controllare lo schema si può passare una `pydantic` `BaseModel` via `schema` al decorator (verbatim):

```python
from pydantic import BaseModel

class WeatherInput(BaseModel):
    """Input schema for the weather tool."""
    location: Annotated[str, Field(description="The city name")]
    unit: Annotated[str, Field(description="celsius or fahrenheit")] = "celsius"

@tool(name="get_weather", description="Get weather for a location.", schema=WeatherInput)
def get_weather(location: str, unit: str = "celsius") -> str:
    return f"The weather in {location} is 22 degrees {unit}."
```

Per accedere a valori runtime non parte dello schema si usa `FunctionInvocationContext` (verbatim):

```python
from agent_framework import FunctionInvocationContext, tool

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The location")],
    ctx: FunctionInvocationContext,
) -> str:
    user_id = ctx.kwargs.get("user_id", "unknown")
    return f"The weather in {location} is cloudy with a high of 15°C."

# Inject kwargs at run time
response = await agent.run(
    "What is the weather like in Amsterdam?",
    function_invocation_kwargs={"user_id": "user_123"},
)
```

Per aggiungere tool a un `Agent`, si passa la lista `tools` al costruttore dell'`Agent` (verbatim):

```python
Agent(
    client=...,              # LLM provider (required)
    name="...",            # Identity for logging (required)
    instructions="...",    # System prompt (required)
    tools=[...],             # Optional: function tools and/or MCP proxies
    default_options={...},   # Optional: temperature, max_tokens, etc.
)
```

Il framework supporta tre tipi principali di tool: Function Tools (`@tool`), MCP Tools (scoperti via `MCPStdioTool`) e Agent-as-Tool patterns.

[[concepts/function-tools]]
[[concepts/agent-definitions]]
[[sources/microsoft-agent-framework-agents-python-howto]]
