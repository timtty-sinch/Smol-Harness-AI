# mysmol

An AI agent harness builder platform focused on managing limited context windows with advanced memory. Built on LangGraph, it provides a multi-agent execution framework that decomposes tasks into plans, executes them with tools, validates results, and synthesizes a final response — all while keeping context usage in check.

## How It Works

The agent runs as a LangGraph state graph with five specialized nodes:

```
START → agent_bootstrap → agent_plan → task_manager → agent_execute → agent_validate
                                              ↑                               |
                                              └─────── (retry / next) ────────┘
                                                               ↓ (done)
                                                        agent_terminate → END
```

| Node | Role |
|------|------|
| `agent_bootstrap` | Initializes state, assigns a unique run ID |
| `agent_plan` | Uses structured output to decompose the prompt into ordered `PlanStep` tasks |
| `task_manager` | Picks the next pending task, enforces the iteration cap |
| `agent_execute` | Runs the task using bound tools; injects relevant prior results via BM25 |
| `agent_validate` | Confirms task output is meaningful; retries up to 3× with error context |
| `agent_terminate` | Synthesizes all completed results into a final response |

## Context Window Management

Every LLM call tracks input and output tokens against a configurable `CONTEXT_WINDOW_MAX` (default: 100k tokens). The live status bar shows current usage:

```
Agent is running... (context window: 12k/100k - 12%)
```

Rather than dumping all completed task results into every prompt, the memory-enabled harnesses use **BM25 retrieval** to inject only the most relevant prior results into each execution context. This keeps per-call token consumption flat regardless of how many tasks have completed.

## Files

### `harness_base.py`
The baseline harness. All completed task results are passed as raw context to each subsequent node. Good for short task lists where context accumulation is not a concern.

### `memory_harness.py`
Adds per-run memory using `InMemoryStore` (LangGraph). Validated task results are stored under a namespaced key `("task_results", run_id)`. At execution and termination, BM25 retrieval (`BM25Retriever`) surfaces the top-k most relevant results rather than the full history. Also adds SQLite LLM response caching (`.cache/llm.db`).

### `agent_latest.py`
Extends `memory_harness.py` with a `bash_execute` tool. Every bash command requires explicit user confirmation before running (interactive prompt via `rich.Confirm`). This is the recommended starting point.

## Tools

| Tool | Description |
|------|-------------|
| `website_search` | Web search via Brave Search API |
| `read_webpage` | Fetches a URL and converts HTML to Markdown |
| `fetch_rss_articles` | Parses RSS/Atom feeds; preferred over `read_webpage` for `.xml` URLs |
| `read_file` | Reads a local file by path |
| `write_file` | Writes content to a local file |
| `bash_execute` | Runs a shell command (with user approval gate in `agent_latest.py`) |
| `add_task` | Queues a new task at runtime — agents can spawn sub-tasks mid-execution |

## Dynamic Task Queuing

If the execution agent discovers new work (e.g. a search returns 5 URLs that each need to be read), it calls `add_task` to enqueue them rather than trying to handle everything in a single prompt. New tasks are appended to the plan and processed in order.

## Retry Logic

Failed validations increment a retry counter and pass the validator's reason back as error context for the next attempt. After 3 failures a task is marked `failed` and the agent moves on.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Copy `.env.example` to `.env` (or create `.env`) and set:

```env
OPENAI_BASE_URL=http://your-openai-compatible-endpoint/v1
OPENAI_API_KEY=your-key
BRAVE_SEARCH_API_KEY=your-brave-key
```

The default model is `qwen/qwen3-4b-2507` via an OpenAI-compatible endpoint. Swap `MODEL` in any harness file to use a different model.

## Running

```bash
# Latest harness (memory + bash tool + user approval)
python agent_latest.py

# Memory harness (no bash)
python memory_harness.py

# Base harness (no memory, no cache)
python harness_base.py
```

Edit the `start_prompt` at the bottom of each file to change the task.

## Dependencies

- `langgraph` — graph execution engine and memory store
- `langchain`, `langchain-openai`, `langchain-community`, `langchain-core`, `langchain-classic` — LLM bindings, tools, BM25 retriever, SQLite cache
- `rich` — terminal UI (live status, tables, trees, markdown)
- `requests`, `html2text` — web fetching
- `python-dotenv` — environment config
- `shortuuid` — run ID generation
- `typing_extensions` — TypedDict schemas
