# Microsoft Agent Framework — Agents (Python How-To)

> **Source**: [https://learn.microsoft.com/en-us/agent-framework/agents/](https://learn.microsoft.com/en-us/agent-framework/agents/)
> Last synced: April 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Agent vs Workflow — When to Use Which](#agent-vs-workflow--when-to-use-which)
3. [Core Concepts](#core-concepts)
   - [The Agent Class](#1-the-agent-class)
   - [Chat Clients (LLM Providers)](#2-chat-clients-llm-providers)
   - [Tools](#3-tools)
   - [MCP Integration](#4-mcp-integration)
4. [Running Agents](#running-agents)
   - [Non-Streaming](#non-streaming)
   - [Streaming](#streaming)
   - [Run Options](#run-options)
5. [Response & Message Types](#response--message-types)
6. [Structured Outputs](#structured-outputs)
7. [Sessions & Multi-Turn Conversations](#sessions--multi-turn-conversations)
8. [Multi-Agent Orchestration Patterns](#multi-agent-orchestration-patterns)
9. [Practical Patterns from This Project](#practical-patterns-from-this-project)
   - [Factory Functions for Agent Definitions](#factory-functions-for-agent-definitions)
   - [Pydantic Validation Between Agents](#pydantic-validation-between-agents)
   - [Handling MCP Parallel Calls](#handling-mcp-parallel-calls)
10. [Installation & Prerequisites](#installation--prerequisites)
11. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
12. [Further Resources](#further-resources)

---

## Overview

The **Agent** is the core building block of the Microsoft Agent Framework. An agent wraps an LLM with a name, a set of instructions, and optional tools. It runs autonomously: it receives a query, decides which tools to call (if any), and returns a response.

```
pip install agent-framework-core
```

The framework provides a **deterministic runtime loop** for every agent:

1. Send the user message + instructions + tool definitions to the LLM.
2. If the LLM returns a tool call → execute the tool → feed the result back to the LLM.
3. Repeat step 2 until the LLM produces a final text response.
4. Return the `AgentResponse`.

All agents derive from a common base class (`AIAgent`), which provides a consistent interface for agent composition, multi-agent orchestration, and workflow integration.

---

## Agent vs Workflow — When to Use Which

| Use an Agent when… | Use a Workflow when… |
|---|---|
| The task is open-ended or conversational | The process has well-defined steps |
| You need autonomous tool use and planning | You need explicit control over execution order |
| A single LLM call (possibly with tools) suffices | Multiple agents or functions must coordinate |

> **Rule of thumb**: if you can write a function to handle the task, do that instead of using an AI agent. If you need a graph with edges, state, and event streaming, use a **Workflow** (see [mfa_workflow_howto.md](mfa_workflow_howto.md)).

---

## Core Concepts

### 1. The Agent Class

The `Agent` class is the single most important primitive. You give it:

- **`name`** — identity for logging and multi-agent scenarios.
- **`instructions`** — a system prompt that defines the agent's behaviour.
- **`client`** — the LLM provider (OpenAI, Azure OpenAI, Foundry, Anthropic, Ollama, …).
- **`tools`** — a list of function tools and/or MCP tool proxies the agent can invoke.
- **`default_options`** — provider-specific settings (temperature, max_tokens, model override, …).

#### Minimal Example — Agent with No Tools

```python
import asyncio
import os

from agent_framework import Agent
from agent_framework.openai import OpenAIChatClient, OpenAIChatOptions
from dotenv import load_dotenv

load_dotenv()

async def main():
    client = OpenAIChatClient(
        api_key=os.environ["OPENAI_API_KEY"],
        model="gpt-4o-mini",
    )

    agent = Agent(
        client=client,
        name="ManagerAgent",
        instructions=(
            "You are a manager agent. Answer the user's query "
            "as accurately and concisely as possible."
        ),
    )

    result = await agent.run("What is a Large Language Model?")
    print(result.text)

asyncio.run(main())
```

> **Mental model**: create a client → create an agent → call `agent.run()` → print the result. Everything is `async`, so you need `asyncio.run()` as entry point.

#### Using the `as_agent()` Shorthand

Chat clients provide a convenience method to create an agent in one step:

```python
from agent_framework.openai import OpenAIChatClient

agent = OpenAIChatClient().as_agent(
    name="HelloAgent",
    instructions="You are a friendly assistant. Keep your answers brief.",
)

result = await agent.run("What is the capital of France?")
```

#### Using Azure Foundry

```python
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential

client = FoundryChatClient(
    project_endpoint="https://your-project.services.ai.azure.com/api/projects/your-project",
    model="gpt-4o",
    credential=AzureCliCredential(),
)

agent = client.as_agent(
    name="FoundryAgent",
    instructions="You are a helpful assistant.",
)
```

#### Using a Local Model (OpenAI-Compatible Endpoint)

```python
from agent_framework.openai import OpenAIChatCompletionClient

client = OpenAIChatCompletionClient(
    base_url="http://127.0.0.1:62770/v1",
    model="phi-4-mini-instruct-openvino-npu:3",
    api_key="none",
)

agent = Agent(
    client=client,
    name="LocalAgent",
    instructions="You are a manager agent.",
)
```

---

### 2. Chat Clients (LLM Providers)

The framework supports multiple LLM providers through different client classes:

| Client Class | Service | Notes |
|---|---|---|
| `OpenAIChatClient` | OpenAI (Responses API) | Supports tool approval, web search, hosted MCP |
| `OpenAIChatCompletionClient` | OpenAI / Azure OpenAI / local (Chat Completion API) | Widest compatibility |
| `FoundryChatClient` | Azure Foundry | Requires `AzureCliCredential` |
| `AnthropicChatClient` | Anthropic Claude | Via Anthropic SDK |
| `OllamaChatClient` | Ollama | Local models |

Each client exposes its own `Options` TypedDict for IDE auto-complete (e.g. `OpenAIChatOptions`, `AnthropicChatOptions`).

```python
from agent_framework.openai import OpenAIChatCompletionClient, OpenAIChatCompletionOptions

client = OpenAIChatCompletionClient(
    base_url="http://127.0.0.1:62770/v1",
    model="phi-4-mini-instruct-openvino-npu:3",
    api_key="none",
)
options = OpenAIChatCompletionOptions(temperature=0.0)

# Pass options at construction time
agent = Agent(client=client, name="Agent", instructions="...", default_options=options)

# Or override per-run
result = await agent.run("Hello", options={"temperature": 0.7, "max_tokens": 150})
```

---

### 3. Tools

Tools extend an agent's capabilities beyond text generation. The framework supports several tool types:

| Tool Type | Description |
|---|---|
| **Function Tools** | Custom Python functions decorated with `@tool` |
| **MCP Tools** | External tools discovered via MCP protocol (stdio or SSE) |
| **Code Interpreter** | Sandboxed code execution (provider-dependent) |
| **File Search** | Search through uploaded files (provider-dependent) |
| **Web Search** | Search the web (provider-dependent) |
| **Agent as Tool** | Use another agent as a function tool |

#### Function Tools with `@tool`

Turn any Python function into a tool the agent can invoke:

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

The `Annotated[str, "description"]` or `Annotated[str, Field(description="...")]` syntax documents parameters for the LLM. The framework reads these annotations and exposes them as part of the tool schema.

You can also use the `@tool` decorator with explicit metadata:

```python
@tool(name="weather_tool", description="Retrieves weather information for any location")
def get_weather(
    location: Annotated[str, Field(description="The location to get weather for")],
) -> str:
    return f"The weather in {location} is cloudy with a high of 15°C."
```

If you omit `name` and `description`, the framework uses the function name and docstring as fallback.

#### Explicit Schemas

For full control over the schema exposed to the model, pass `schema` to `@tool`:

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

#### Runtime Context in Tools (`FunctionInvocationContext`)

Use `FunctionInvocationContext` to access runtime-only values (user IDs, database connections, session) that should **not** be part of the schema:

```python
from agent_framework import FunctionInvocationContext, tool

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The location")],
    ctx: FunctionInvocationContext,
) -> str:
    """Get the weather for a given location."""
    user_id = ctx.kwargs.get("user_id", "unknown")
    print(f"Getting weather for user: {user_id}")
    return f"The weather in {location} is cloudy with a high of 15°C."

# Inject kwargs at run time
response = await agent.run(
    "What is the weather like in Amsterdam?",
    function_invocation_kwargs={"user_id": "user_123"},
)
```

#### Class with Multiple Function Tools

When tools share state or dependencies, wrap them in a class:

```python
from agent_framework import tool

class MyTools:
    def __init__(self, safe: bool = False):
        self.safe = safe

    def divide(self, a: Annotated[int, "Numerator"], b: Annotated[int, "Denominator"]) -> str:
        """Divide two numbers."""
        result = "∞" if b == 0 and self.safe else a / b
        return f"{a} / {b} = {result}"

    def add(self, x: Annotated[int, "First number"], y: Annotated[int, "Second number"]) -> str:
        return f"{x} + {y} = {x + y}"

tools = MyTools(safe=True)
add_fn = tool(description="Add two numbers.")(tools.add)

agent = Agent(client=client, name="MathAgent", instructions="Use the tools.", tools=[add_fn, tools.divide])
```

#### Providing Tools Per-Run

You can override or extend the tool list at each `run()` call:

```python
result = await agent.run("Divide 10 by 3", tools=[tools.divide])
```

---

### 4. MCP Integration

The framework has **first-class support** for [Model Context Protocol (MCP)](https://modelcontextprotocol.io/). MCP is the standard for exposing tools to AI agents. You point the framework at an MCP server command and it **auto-discovers** every tool the server exposes.

#### MCPStdioTool — Local MCP Server (stdio)

```python
from agent_framework import MCPStdioTool

jira_proxy = MCPStdioTool(
    name="jira_server",
    command="pipenv",
    args=["run", "mcp-atlassian"],
    env={
        "JIRA_URL": os.getenv("JIRA_URL"),
        "JIRA_USERNAME": os.getenv("JIRA_USERNAME"),
        "JIRA_API_TOKEN": os.getenv("JIRA_API_TOKEN"),
    },
)

agent = Agent(
    name="JiraAgent",
    instructions="You are a Jira assistant.",
    client=client,
    tools=[jira_proxy],
)

try:
    result = await agent.run("Create an epic called 'Shopping List' in SARI project")
    print(result.text)
finally:
    await jira_proxy.close()  # Always close MCP proxies
```

> **Important**: always `await proxy.close()` in a `finally` block to terminate the MCP subprocess.

#### Combining MCP Tools with Function Tools

An agent can use both MCP tools and function tools simultaneously:

```python
agent = Agent(
    name="JiraExecutor",
    instructions="Create issues on Jira, then write a summary.",
    client=client,
    tools=[jira_proxy, write_file],  # MCP + @tool function
)
```

---

## Running Agents

### Non-Streaming

```python
result = await agent.run("What is the weather like in Amsterdam?")
print(result.text)
```

### Streaming

Pattern 1 — Async iteration for real-time display:

```python
async for update in agent.run("Tell me a story", stream=True):
    if update.text:
        print(update.text, end="", flush=True)
```

Pattern 2 — Skip iteration, get the complete response:

```python
stream = agent.run("Tell me a story", stream=True)
final = await stream.get_final_response()
print(final.text)
```

Pattern 3 — Combined (iterate + finalize):

```python
stream = agent.run("Tell me a story", stream=True)

async for update in stream:
    if update.text:
        print(update.text, end="", flush=True)

final = await stream.get_final_response()
print(f"\n\nComplete: {final.text}")
```

### Run Options

Options can be set at construction time (`default_options`) or overridden per-run. Per-run options take precedence and are merged with defaults.

```python
from agent_framework.openai import OpenAIChatOptions

# Construction time
agent = client.as_agent(
    instructions="You are helpful.",
    default_options={"temperature": 0.7, "max_tokens": 500},
)

# Per-run override
result = await agent.run(
    "Tell me a detailed forecast",
    options={"temperature": 0.3, "max_tokens": 150, "model": "gpt-4o"},
)
```

Common options:

| Option | Description |
|---|---|
| `temperature` | Controls randomness (0.0 = deterministic, 1.0+ = creative) |
| `max_tokens` | Maximum tokens to generate |
| `model` | Override the model for this run |
| `top_p` | Nucleus sampling parameter |
| `response_format` | Structured output schema (Pydantic model or JSON schema) |

> **Note**: `tools` and `instructions` are keyword arguments to `run()`, not part of `options`.

---

## Response & Message Types

### AgentResponse (Non-Streaming)

```python
response = await agent.run("Hello")
print(response.text)                 # Aggregated text from all TextContent
print(len(response.messages))        # Number of messages

for message in response.messages:
    print(f"Role: {message.role}, Text: {message.text}")
```

### AgentResponseUpdate (Streaming)

```python
async for update in agent.run("Hello", stream=True):
    print(f"Text: {update.text}")
    for content in update.contents:
        if content.type == "text":
            print(f"Content: {content.text}")
```

### Content Types

Messages are composed of content items. Use the `type` property to check the content type:

| `content.type` | Description |
|---|---|
| `"text"` | Textual content (typically the agent's answer) |
| `"text_reasoning"` | Chain-of-thought reasoning (models that support it) |
| `"data"` | Binary content encoded as data URI (images, audio, video) |
| `"uri"` | URL pointing to hosted content |
| `"function_call"` | LLM requests a function tool invocation |
| `"function_result"` | Result of a function tool invocation |
| `"mcp_server_tool_call"` | Request to invoke an MCP server tool |
| `"mcp_server_tool_result"` | Result from an MCP server tool |
| `"error"` | Error information |
| `"usage"` | Token usage and billing info |

### Creating Messages Programmatically

```python
from agent_framework import Message, Content

# Simple text message
text_msg = Message(role="user", contents=["Hello!"])

# Multi-content message (text + image)
image_data = b"..."  # your image bytes
mixed_msg = Message(
    role="user",
    contents=[
        Content.from_text("Analyze this image:"),
        Content.from_data(data=image_data, media_type="image/png"),
    ],
)
```

---

## Structured Outputs

Force the agent to return data conforming to a specific schema. Pass a **Pydantic model** or a **JSON schema dict** via `response_format` in options.

### Using a Pydantic Model

```python
from pydantic import BaseModel

class PersonInfo(BaseModel):
    name: str | None = None
    age: int | None = None
    occupation: str | None = None

response = await agent.run(
    "John Smith is a 35-year-old software engineer.",
    options={"response_format": PersonInfo},
)

if response.value:
    person = response.value  # PersonInfo instance
    print(f"{person.name}, {person.age}, {person.occupation}")
```

### Using a JSON Schema Dict

```python
person_schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer"},
        "occupation": {"type": "string"},
    },
    "required": ["name", "age", "occupation"],
}

response = await agent.run(
    "John Smith is a 35-year-old software engineer.",
    options={"response_format": person_schema},
)

if response.value:
    person = response.value  # dict
    print(f"{person['name']}, {person['age']}, {person['occupation']}")
```

### Streaming + Structured Outputs

```python
stream = agent.run(query, stream=True, options={"response_format": PersonInfo})

async for update in stream:
    print(update.text, end="", flush=True)

final = await stream.get_final_response()
if final.value:
    print(f"\nParsed: {final.value.name}")
```

---

## Sessions & Multi-Turn Conversations

`AgentSession` is the conversation state container. It preserves chat history across multiple `run()` calls.

```python
session = agent.create_session()

first = await agent.run("My name is Alice.", session=session)
second = await agent.run("What is my name?", session=session)
# → "Your name is Alice."
```

### AgentSession Contents

| Property | Description |
|---|---|
| `session_id` | Local unique identifier |
| `service_session_id` | Remote service conversation ID (service-managed history) |
| `state` | Mutable dict shared with context/history providers |

### Resuming from an Existing Service Conversation

```python
session = agent.get_session(service_session_id="<service-conversation-id>")
response = await agent.run("Continue this conversation.", session=session)
```

### Serialization / Restoration

```python
serialized = session.to_dict()
resumed = AgentSession.from_dict(serialized)
```

> **Important**: sessions are agent/service-specific. Reusing a session with a different agent or provider can lead to invalid context.

---

## Multi-Agent Orchestration Patterns

### Sequential Pipeline (Plain Python)

The simplest multi-agent pattern: call `agent.run()`, pass the result to the next agent.

```python
response1 = await reader_agent.run("Read the file 'backlog.md'")
response2 = await executor_agent.run(f"Execute this backlog:\n\n{response1.text}")
```

You are the orchestrator — there is no pipeline abstraction. This gives you full control.

### Agent as a Function Tool

Use an agent as a tool for another agent. The inner agent is automatically converted to a function tool:

```python
weather_agent = client.as_agent(
    name="WeatherAgent",
    instructions="You answer questions about the weather.",
    tools=[get_weather],
)

main_agent = client.as_agent(
    name="MainAgent",
    instructions="You are a helpful assistant.",
    tools=[weather_agent.as_function_tool()],
)

result = await main_agent.run("What is the weather in Amsterdam?")
```

### Using Agents Inside Workflows

Agents can be wrapped in `AgentExecutor` to participate in workflow graphs:

```python
from agent_framework import AgentExecutor

spam_agent = AgentExecutor(
    client.as_agent(
        instructions="You are a spam detection assistant.",
        response_format=DetectionResult,
    ),
    id="spam_detection",
)

workflow = (
    WorkflowBuilder(start_executor=spam_agent)
    .add_edge(spam_agent, handler_executor)
    .build()
)
```

> For full workflow details, see [mfa_workflow_howto.md](mfa_workflow_howto.md).

---

## Practical Patterns from This Project

### Factory Functions for Agent Definitions

This project adopts a convention: each agent lives in its own module under `afw_core/agents/` and exposes a `create_agent()` factory function.

**Name and instructions are hardcoded** (they are intrinsic to the agent's identity). **Client, options, and tools are always injected** (they are infrastructure concerns).

```python
# afw_core/agents/backlog_reader.py
from agent_framework import Agent

def create_agent(client, options, tools):
    """Create the BacklogReaderAgent."""
    return Agent(
        name="BacklogReaderAgent",
        instructions=(
            "You are a backlog reader assistant. "
            "When asked, use the read_file tool to read a markdown file from the input directory. "
            "After reading, respond with ONLY a JSON object matching this schema: "
            '{"epic_count": <int>, "story_count": <int>, "description": <string>}.'
        ),
        client=client,
        default_options=options,
        tools=tools,
    )
```

```python
# afw_core/agents/jira_executor.py
from agent_framework import Agent

def create_agent(client, options, tools):
    """Create the JiraExecutorAgent."""
    return Agent(
        name="JiraExecutorAgent",
        instructions=(
            "You are a Jira execution assistant. "
            "Create issues ONE AT A TIME, never in parallel. "
            "When linking stories to an epic, first create the epic, "
            "then each story with the epic as parent. "
            "After all operations, write a summary using write_file."
        ),
        client=client,
        default_options=options,
        tools=tools,
    )
```

The entry point becomes clean orchestration logic:

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

### Pydantic Validation Between Agents

Instead of passing raw text from one agent to the next, define a **data contract** with Pydantic:

```python
# afw_core/models/backlog.py
from pydantic import BaseModel

class BacklogOutput(BaseModel):
    epic_count: int
    story_count: int
    description: str
```

```python
from afw_core.models.backlog import BacklogOutput

read_response = await reader_agent.run("Read 'backlog.md'")
backlog = BacklogOutput.model_validate_json(read_response.text)
print(f"Backlog: {backlog.epic_count} epic(s), {backlog.story_count} stories")

# Now pass validated data to the next agent
await executor_agent.run(f"Execute this backlog:\n\n{backlog.description}")
```

If the agent returns malformed JSON, Pydantic throws immediately — rather than letting corrupted data propagate downstream. This is cheap insurance for inter-agent contracts.

### Handling MCP Parallel Calls

MCP servers (like `mcp-atlassian` for Jira) often cannot handle parallel tool calls reliably. When the LLM sends multiple calls in one response, the server may fail.

**The fix**: instruct the agent explicitly:

```python
Agent(
    name="JiraExecutor",
    instructions=(
        "Create issues ONE AT A TIME, never in parallel. "
        "Wait for each to complete before the next."
    ),
    client=client,
    tools=[jira_proxy],
)
```

The agent respects this instruction. This is a general pattern: if your external system doesn't handle concurrency well, say so in the instructions.

---

## Installation & Prerequisites

| Requirement | Value |
|---|---|
| **Python** | 3.10+ |
| **Package** | `pip install agent-framework-core` |
| **Azure OpenAI** | Endpoint + deployment configured |
| **OpenAI** | API key from [platform.openai.com](https://platform.openai.com/) |
| **Authentication** | `az login` (AzureCliCredential) or API key |

Environment variables (example):

```env
# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_CHAT_MODEL=gpt-4o-mini

# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_CHAT_COMPLETION_MODEL=gpt-4o-mini
AZURE_OPENAI_API_VERSION=2024-12-01-preview

# Azure Foundry
FOUNDRY_PROJECT_ENDPOINT=https://your-project.services.ai.azure.com/api
FOUNDRY_MODEL=gpt-4o

# Jira (for MCP)
JIRA_URL=https://your-instance.atlassian.net
JIRA_USERNAME=your@email.com
JIRA_API_TOKEN=your-jira-api-token
```

> **Note**: Agent Framework does not automatically load `.env` files. Call `load_dotenv()` at the start of your script or set variables in your shell.

---

## Quick Reference Cheat Sheet

```python
from agent_framework import (
    Agent,                  # Core agent class
    AgentSession,           # Conversation state container
    AgentExecutor,          # Wraps an agent for use in workflows
    MCPStdioTool,           # MCP server bridge (stdio)
    Message,                # Chat message type
    Content,                # Message content items
    FunctionInvocationContext,  # Runtime context for tools
    tool,                   # Decorator to create function tools
)
from agent_framework.openai import (
    OpenAIChatClient,               # OpenAI Responses API
    OpenAIChatCompletionClient,     # OpenAI / Azure OpenAI Chat Completion
    OpenAIChatOptions,              # Options for Responses client
    OpenAIChatCompletionOptions,    # Options for Chat Completion client
)
from agent_framework.foundry import (
    FoundryChatClient,      # Azure Foundry client
)

# 1. Create a client
client = OpenAIChatClient(api_key="...", model="gpt-4o-mini")

# 2. Create an agent
agent = Agent(
    client=client,
    name="MyAgent",
    instructions="You are a helpful assistant.",
    tools=[my_tool, mcp_proxy],
    default_options={"temperature": 0.0},
)

# 3. Run (non-streaming)
result = await agent.run("Hello!")
print(result.text)

# 4. Run (streaming)
async for update in agent.run("Hello!", stream=True):
    if update.text:
        print(update.text, end="", flush=True)

# 5. Multi-turn conversation
session = agent.create_session()
await agent.run("My name is Alice.", session=session)
await agent.run("What is my name?", session=session)

# 6. Structured output
from pydantic import BaseModel
class Info(BaseModel):
    name: str
    age: int

result = await agent.run("John is 30.", options={"response_format": Info})
print(result.value.name)  # "John"
```

---

## Further Resources

| Resource | Link |
|---|---|
| Official overview | [learn.microsoft.com/en-us/agent-framework/overview/](https://learn.microsoft.com/en-us/agent-framework/overview/?pivots=programming-language-python) |
| Agent types | [Agents docs](https://learn.microsoft.com/en-us/agent-framework/agents/) |
| Running agents | [Running agents docs](https://learn.microsoft.com/en-us/agent-framework/agents/running-agents?pivots=programming-language-python) |
| Function tools | [Function tools docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/function-tools?pivots=programming-language-python) |
| Tools overview | [Tools docs](https://learn.microsoft.com/en-us/agent-framework/agents/tools/) |
| Structured outputs | [Structured outputs docs](https://learn.microsoft.com/en-us/agent-framework/agents/structured-outputs?pivots=programming-language-python) |
| Sessions | [Session docs](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/session?pivots=programming-language-python) |
| Middleware | [Middleware docs](https://learn.microsoft.com/en-us/agent-framework/agents/middleware/) |
| Providers | [Providers docs](https://learn.microsoft.com/en-us/agent-framework/agents/providers/) |
| Workflows how-to | [mfa_workflow_howto.md](mfa_workflow_howto.md) |
| Python samples | [github.com/.../python/samples](https://github.com/microsoft/agent-framework/tree/main/python/samples) |
| This project | [github.com/rosidotidev/MSFTAgentSample](https://github.com/rosidotidev/MSFTAgentSample) |
