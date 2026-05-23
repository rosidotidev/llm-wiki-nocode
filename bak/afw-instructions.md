# Microsoft Agent Framework – Project Best Practices

This document defines the conventions, project structure, and naming rules used in this codebase when building applications with the **Microsoft Agent Framework** (`agent-framework` Python package).

Follow these guidelines when generating, modifying, or reviewing code in this project.

---

## Project Structure

```
project_root/
├── afw_core/               # All reusable components live here
│   ├── agents/             # Agent definitions
│   ├── executors/          # Workflow executors (processing units)
│   ├── workflows/          # Workflow builder definitions (graph wiring)
│   ├── tools/              # Custom function tools (@tool)
│   ├── mcps/               # MCP server proxy configurations
│   ├── llms/               # LLM client factory functions
│   └── models/             # Pydantic models for state, inputs, outputs
│
├── input/                  # Input files consumed by agents (e.g. .md backlogs)
├── output/                 # Output files produced by agents (e.g. execution reports)
│
├── main_<pipeline_name>.py # Entry point scripts (orchestration only)
├── .env                    # Environment variables (API keys, URLs)
├── Pipfile                 # Dependency management
└── setup.txt               # Pinned install commands
```

### Important: No `__init__.py` files

Do **not** generate `__init__.py` files in any directory under `afw_core/`. The `afw_core` namespace is unique enough to avoid collisions with installed packages. Python implicit namespace packages are used.

If import errors occur, the fix is to choose a non-colliding directory name (e.g. `afw_core` instead of `agents` or `tools`), **not** to add `__init__.py` files.

---

## Module Conventions

### `afw_core/agents/` — Agent Definitions

Each file defines **one agent**. The file exposes a single factory function:

```python
def create_agent(client, options, tools):
```

- `name` and `instructions` are **hardcoded** inside the function — they are intrinsic to the agent's identity.
- `client`, `options`, and `tools` are **always passed from outside** — they are infrastructure concerns.
- File naming: `snake_case` describing the agent's role (e.g. `backlog_reader.py`, `jira_executor.py`).

Example:

```python
from agent_framework import Agent

def create_agent(client, options, tools):
    return Agent(
        name="BacklogReaderAgent",
        instructions="You are a backlog reader assistant. ...",
        client=client,
        default_options=options,
        tools=tools,
    )
```

### `afw_core/tools/` — Custom Function Tools

Each file defines **one tool** (or a small group of closely related tools). Use the `@tool` decorator from `agent_framework`.

- Use `Annotated[type, "description"]` for parameter descriptions.
- Tools are standalone functions, not methods on a class.
- File naming: `snake_case` describing the tool's action (e.g. `file_reader.py`, `file_writer.py`).

Example:

```python
from typing import Annotated
from agent_framework import tool

@tool
def read_file(file_name: Annotated[str, "Name of the file to read"]) -> str:
    """Read and return the contents of a file."""
    with open(file_name, "r", encoding="utf-8") as f:
        return f.read()
```

### `afw_core/mcps/` — MCP Server Proxies

Each file configures **one MCP server** connection. The file exposes a single factory function:

```python
def create_proxy():
```

- Returns an `MCPStdioTool` instance.
- Environment variables for credentials are read inside the function via `os.getenv()`.
- File naming: the name of the external service (e.g. `jira.py`, `confluence.py`, `github.py`).

Example:

```python
import os
from agent_framework import MCPStdioTool

def create_proxy():
    return MCPStdioTool(
        name="jira_server",
        command="pipenv",
        args=["run", "mcp-atlassian"],
        env={
            "JIRA_URL": os.getenv("JIRA_URL"),
            "JIRA_USERNAME": os.getenv("JIRA_USERNAME"),
            "JIRA_API_TOKEN": os.getenv("JIRA_API_TOKEN"),
        },
    )
```

### `afw_core/llms/` — LLM Client Factories

Each file configures **one LLM provider**. The file exposes a single factory function:

```python
def create_client(api_key, model, options=DEFAULT_OPTIONS):
```

- `api_key` and `model` are **always passed from outside** (typically from environment variables in the entry point).
- `options` has a sensible default defined as a module-level constant.
- Returns a tuple: `(client, options)`.
- File naming: the provider name (e.g. `openai.py`, `azure.py`, `foundry.py`).

Example:

```python
from agent_framework.openai import OpenAIChatClient, OpenAIChatOptions

DEFAULT_OPTIONS = OpenAIChatOptions(temperature=0.0)

def create_client(api_key: str, model: str, options: OpenAIChatOptions = DEFAULT_OPTIONS):
    client = OpenAIChatClient(api_key=api_key, model=model)
    return client, options
```

### `afw_core/executors/` — Workflow Executors

Each file defines **one executor** — the processing unit of a workflow. An executor is a class that extends `Executor`, receives a typed message via a `@handler` method, and either forwards results downstream (`ctx.send_message`) or produces final output (`ctx.yield_output`).

- File naming: `snake_case` describing what the executor does (e.g. `data_extractor.py`, `report_formatter.py`).
- Class naming: `PascalCaseExecutor` (e.g. `DataExtractorExecutor`).
- Constructor receives dependencies (client, options, config) and stores them as instance attributes.
- The executor `id` (passed to `super().__init__(id="...")`) is used in event streaming to identify which executor is running.

#### Executor Categories

Executors fall into three categories based on how they perform their work:

| Category | Uses LLM? | When to use |
|---|---|---|
| **Agent-wrapping** | Yes | The executor delegates to an `Agent` from `afw_core/agents/`. Use when the task requires LLM reasoning, tool calling, or natural language generation. |
| **Pure-Python** | No | The executor performs deterministic logic (parsing, transforming, file I/O, regex, API calls). Prefer this whenever the transformation is rule-based. |
| **Direct API** | Yes (no agent) | The executor calls an LLM API directly (e.g. `AsyncOpenAI`) without going through an `Agent`. Use for batch/parallel LLM calls where the agent abstraction adds overhead. |

**Principle: minimise LLM usage.** If a step can be implemented deterministically in Python with equivalent quality, do not use an LLM. Reserve LLM calls for tasks that genuinely require language understanding, generation, or vision.

#### Agent-Wrapping Executor

```python
from agent_framework import Executor, WorkflowContext, handler
from afw_core.agents.my_agent import create_agent
from afw_core.tools.my_tool import my_tool

class MyAgentExecutor(Executor):

    def __init__(self, client, options):
        super().__init__(id="my_agent")
        self._agent = create_agent(client=client, options=options, tools=[my_tool])

    @handler
    async def handle(self, input_data: str, ctx: WorkflowContext[str]) -> None:
        response = await self._agent.run(input_data)
        await ctx.send_message(response.text)
```

#### Pure-Python Executor

```python
import json
from agent_framework import Executor, WorkflowContext, handler

class DataTransformerExecutor(Executor):

    def __init__(self):
        super().__init__(id="data_transformer")

    @handler
    async def handle(self, raw_json: str, ctx: WorkflowContext[str]) -> None:
        data = json.loads(raw_json)
        transformed = self._process(data)
        await ctx.send_message(json.dumps(transformed))

    def _process(self, data: dict) -> dict:
        # Deterministic transformation — no LLM needed
        ...
```

#### Direct API Executor

```python
import asyncio
from openai import AsyncOpenAI
from agent_framework import Executor, WorkflowContext, handler

class ParallelVisionExecutor(Executor):

    def __init__(self, max_concurrent: int = 5):
        super().__init__(id="parallel_vision")
        self._semaphore = asyncio.Semaphore(max_concurrent)

    @handler
    async def handle(self, images_json: str, ctx: WorkflowContext[str]) -> None:
        aclient = AsyncOpenAI()
        tasks = [self._describe(aclient, img) for img in json.loads(images_json)]
        results = await asyncio.gather(*tasks)
        await ctx.send_message(json.dumps(results))

    async def _describe(self, aclient, image: dict) -> dict:
        async with self._semaphore:
            response = await aclient.chat.completions.create(...)
            return {"path": image["path"], "description": response.choices[0].message.content}
```

#### Executor Communication Patterns

| Pattern | Method | Use |
|---|---|---|
| Forward to next executor | `ctx.send_message(data)` | Pass data along the graph edge |
| Produce workflow output | `ctx.yield_output(data)` | Return data to the workflow caller |
| Share state across steps | `ctx.set_state(key, val)` / `ctx.get_state(key)` | Cross-executor data that doesn't flow through edges |

Use `send_message` for data that flows along graph edges. Use `set_state` for large data that multiple downstream executors need without re-sending through each edge.

---

### `afw_core/workflows/` — Workflow Definitions

Each file defines **one workflow** by wiring executors together using `WorkflowBuilder`. The file exposes a single factory function:

```python
def build_<workflow_name>_workflow(...):
```

- The function instantiates executors, connects them with edges, and returns the built `Workflow` object.
- Parameters are the dependencies that executors need (e.g. `client`, `options`, `output_dir`). Only pass what is actually required — if no executor uses an LLM, the workflow builder should not require `client`/`options`.
- File naming: `snake_case` describing the workflow's purpose (e.g. `doc_ingest.py`, `report_generation.py`).
- The graph topology should be documented in the docstring.

Example:

```python
from agent_framework import WorkflowBuilder

from afw_core.executors.extractor import ExtractorExecutor
from afw_core.executors.transformer import TransformerExecutor
from afw_core.executors.writer import WriterExecutor


def build_etl_workflow(client, options, output_dir: str):
    """Build an ETL workflow.

    Graph:  Extractor ──▶ Transformer ──▶ Writer
    """
    extractor = ExtractorExecutor(client, options)
    transformer = TransformerExecutor()
    writer = WriterExecutor(output_dir)

    return (
        WorkflowBuilder(start_executor=extractor)
        .add_edge(extractor, transformer)
        .add_edge(transformer, writer)
        .build()
    )
```

#### Workflow Versioning

When evolving a workflow, create a new file with a descriptive suffix rather than modifying the existing one. This preserves previous versions for comparison and rollback:

```
afw_core/workflows/
├── doc_ingest.py               # v1: basic pipeline
├── doc_ingest_tpl.py           # v2: adds template pattern
├── doc_ingest_tpl_multi.py     # v3: adds multi-doc scaling
└── doc_ingest_tpl_multi_opt.py # v4: LLM-minimal variant
```

Each version gets its own `main_<name>.py` entry point.

#### Scaling Patterns

When a workflow must handle variable-size input (multiple files, long documents), apply these patterns inside executors:

| Pattern | Problem it solves | Implementation |
|---|---|---|
| **Parallel I/O** | Sequential API calls are slow | `asyncio.gather` with `asyncio.Semaphore` for bounded concurrency |
| **Chunking** | Single LLM call exceeds context window | Split input at logical boundaries (e.g. headings), process each chunk with a separate LLM call, concatenate results |
| **Deterministic assembly** | LLM adds cost/latency for rule-based transforms | Replace the LLM call with a Python function that maps input types to output format |

---

### `afw_core/models/` — Pydantic Models

Each file defines Pydantic models for structured data exchange between agents or for validating agent outputs.

- Use `pydantic.BaseModel` as the base class.
- File naming: the domain concept (e.g. `backlog.py`, `execution_result.py`).
- Models can be used with `model_validate_json()` to parse and validate LLM text responses.

Example:

```python
from pydantic import BaseModel

class BacklogOutput(BaseModel):
    epic_count: int
    story_count: int
    description: str
```

---

## Entry Point Scripts

Files named `main_<pipeline_name>.py` at the project root serve as **orchestration-only** entry points.

- Import agents, tools, mcps, llms, models, and workflows from `afw_core.*`.
- Create shared resources (client, options, proxies).
- Compose agents with their tools, or build workflows.
- Define the execution pipeline (sequential agent calls or workflow streaming).
- Handle cleanup (e.g. `await proxy.close()`).

### Agent-Based Entry Point

Import pattern:

```python
from afw_core.llms.openai import create_client
from afw_core.mcps.jira import create_proxy
from afw_core.tools.file_reader import read_file
from afw_core.agents.backlog_reader import create_agent as create_reader_agent
from afw_core.models.backlog import BacklogOutput
```

When importing multiple agents, alias them:

```python
from afw_core.agents.reader import create_agent as create_reader_agent
from afw_core.agents.executor import create_agent as create_executor_agent
```

### Workflow-Based Entry Point

For workflow pipelines, the entry point builds the workflow and consumes the event stream:

```python
import asyncio
import logging
import os

from dotenv import load_dotenv

from afw_core.llms.openai import create_client
from afw_core.workflows.my_pipeline import build_my_pipeline_workflow

logging.basicConfig(level=logging.WARNING, format="%(name)s %(levelname)s %(message)s")
logging.getLogger("agent_framework").setLevel(logging.INFO)

load_dotenv()


async def main():
    client, options = create_client(
        api_key=os.environ["OPENAI_API_KEY"],
        model=os.environ.get("OPENAI_CHAT_MODEL", "gpt-4o-mini"),
    )

    workflow = build_my_pipeline_workflow(client, options)

    async for event in workflow.run("input data", stream=True):
        if event.type == "executor_invoked":
            print(f"Starting {event.executor_id}...")
        elif event.type == "executor_completed":
            print(f"Done: {event.executor_id}")
        elif event.type == "output":
            print(f"Result: {event.data}")


if __name__ == "__main__":
    asyncio.run(main())
```

Use a label dictionary to map executor IDs to human-readable descriptions in the event stream output.

---

## Naming Conventions Summary

| Element | Convention | Example |
|---|---|---|
| Directories | `snake_case`, under `afw_core/` | `afw_core/agents/` |
| Files | `snake_case`, one concept per file | `backlog_reader.py` |
| Agent factory | `create_agent(client, options, tools)` | Always this signature |
| MCP factory | `create_proxy()` | Always this name |
| LLM factory | `create_client(api_key, model, options)` | Always this signature |
| Tools | `@tool` decorated functions | `read_file()`, `write_file()` |
| Models | PascalCase Pydantic classes | `BacklogOutput` |
| Executor class | `PascalCaseExecutor` in `afw_core/executors/` | `DataExtractorExecutor` |
| Executor `id` | `snake_case`, unique within the workflow | `"data_extractor"` |
| Workflow builder | `build_<name>_workflow(...)` in `afw_core/workflows/` | `build_etl_workflow()` |
| Entry points | `main_<pipeline_name>.py` | `main_backlog_from_md.py` |

---

## Agent Behavior Rules

- When an agent needs to create multiple items via MCP tools (e.g. Jira issues), instruct it to **create them one at a time, never in parallel**. Parallel MCP tool calls to the same server can cause errors.
- Include this constraint in the agent's `instructions`, not in the user query.
- Use agent `instructions` for behavioral constraints (how). Use the user query for task specifics (what).

---

## Logging

Use Python's standard `logging` module with the `agent_framework` logger:

```python
import logging
logging.basicConfig(level=logging.WARNING, format="%(name)s %(levelname)s %(message)s")
logging.getLogger("agent_framework").setLevel(logging.DEBUG)  # or INFO for less noise
```

Set the base level to `WARNING` to silence `httpcore`, `httpx`, and `openai` HTTP noise. Set `agent_framework` logger independently.

---

## Key Framework Defaults

| Setting | Default | Location |
|---|---|---|
| `max_iterations` (tool-call loop) | 40 | `agent_framework._tools` |
| `max_consecutive_errors_per_request` | 3 | `agent_framework._tools` |

These can be overridden via `client.function_invocation_configuration` if needed.

---

## Dependencies

Managed via `pipenv`. Only direct dependencies are listed:

- `agent-framework` — core framework
- `agent-framework-openai` — OpenAI provider (installed as sub-package)
- `python-dotenv` — `.env` file loading
- Install additional MCP server packages or tool libraries as needed for your specific use case

`pydantic` is a transitive dependency of `agent-framework` and does not need to be installed separately.
