# LangGraph Complete — L10

> Version: LangGraph v1.0 (GA Oct 2025). Last verified: April 2026.
> AI connection: LangGraph wraps the Anthropic SDK with explicit state management. The `messages` key IS the conversation history you pass to `client.messages.create()`.

---

## What LangGraph Is

**Analogy:** A state machine for AI agents. You define nodes (workers), edges (transitions), and a shared state (a typed dict everyone reads/writes). LangGraph runs the machine, handles checkpointing, streaming, and human interruption. You define the shape — LangGraph runs it reliably.

Without LangGraph, your agent is a `while True:` loop with a dict. With LangGraph, that same loop has:
- Inspectable state at every step
- Pause/resume at any point
- Parallel fan-out
- Time-travel debugging
- Cross-session memory

**Key insight:** LangGraph is NOT an LLM framework. It's a graph runtime. You bring the LLM (Anthropic SDK or LangChain). LangGraph manages the flow around it.

---

## Concept 1 — StateGraph Mental Model

The `StateGraph` is a directed graph. Nodes are functions. Edges are transitions. State is a `TypedDict` shared across all nodes.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    question: str
    answer: str
    step_count: int

def ask_llm(state: State) -> dict:
    # Returns only the keys it's updating
    return {"answer": "42", "step_count": state["step_count"] + 1}

builder = StateGraph(State)
builder.add_node("ask_llm", ask_llm)
builder.add_edge(START, "ask_llm")  # wire start
builder.add_edge("ask_llm", END)    # wire end

graph = builder.compile()           # validates graph structure
result = graph.invoke({"question": "What is life?", "step_count": 0})
# result: {"question": "...", "answer": "42", "step_count": 1}
```

**Rules:**
- `compile()` validates the structure. Missing `START`/`END` wiring fails at compile time, not runtime.
- Each node returns ONLY the keys it changed — the runtime merges them into full state.
- `invoke()` → sync. `ainvoke()` → async. Same API.
- `START` and `END` are constants from `langgraph.graph`, not nodes you define.

---

## Concept 2 — State with Annotated Reducers

**Analogy:** Without a reducer, writing a key is like `dict["key"] = value` — last writer wins. With a reducer, it's like `list.append()` — all writers contribute. The `add_messages` reducer is the most important one.

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]  # appends, de-duplicates by id
    results: Annotated[list[str], operator.add]           # appends lists
    status: str                                           # last write wins (default)
```

**MessagesState shorthand** — most agents just need this:
```python
from langgraph.graph import MessagesState

# Equivalent to State with messages: Annotated[list[AnyMessage], add_messages]
builder = StateGraph(MessagesState)
```

**What `add_messages` does:**
1. Appends new messages to the list
2. If two messages share the same `id`, the new one replaces the old one (no duplicates)
3. Accepts raw dicts `{"role": "user", "content": "..."}` — auto-converts to message objects

**Custom reducer:**
```python
def merge_dicts(a: dict, b: dict) -> dict:
    return {**a, **b}

class State(TypedDict):
    metadata: Annotated[dict, merge_dicts]
```

**Critical:** Without `operator.add` on a list field, two parallel nodes writing `["item"]` will overwrite each other — you'll get the last one, not both.

---

## Concept 3 — Nodes

Nodes are plain Python functions (sync or async). They receive full state, return a dict of only the keys they're updating.

```python
# Sync
def my_node(state: State) -> dict:
    return {"status": "done"}

# Async (preferred for I/O-bound work — LLM calls, DB queries)
async def call_llm(state: State) -> dict:
    response = await anthropic_client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": m.role, "content": m.content} for m in state["messages"]],
    )
    return {"messages": [AIMessage(response.content[0].text)]}

# Register
builder.add_node("call_llm", call_llm)

# v1.0 shorthand — name inferred from function name
builder.add_node(call_llm)  # registered as "call_llm"
```

**Rules:**
- NEVER mutate `state` dict in place — return a new dict
- NEVER do blocking I/O in sync nodes when running an async graph — use `async def`
- State is a snapshot per node call — not a shared mutable object

---

## Concept 4 — Conditional Edges

Router functions read state and return the next node name. This is how branching works.

```python
from typing_extensions import Literal
from langgraph.graph import END

def route(state: State) -> Literal["node_b", "node_c", "__end__"]:
    if state["score"] > 0.8:
        return "node_b"
    elif state["needs_human"]:
        return END  # END == "__end__"
    else:
        return "node_c"

builder.add_conditional_edges(
    "decision_node",   # source: after this node, call the router
    route,             # router function
)
# No mapping dict needed if return values match exact node names
```

**With explicit mapping (when return values are not node names):**
```python
builder.add_conditional_edges(
    "decision_node",
    route,
    {"high": "node_b", "needs_review": "node_c", "done": END},
)
```

**Alternative: `Command` primitive (preferred when routing and state update are coupled):**
```python
from langgraph.types import Command

def decision_node(state: State) -> Command[Literal["node_b", "node_c"]]:
    return Command(
        update={"status": "routed"},   # state update
        goto="node_b" if state["score"] > 0.8 else "node_c",  # routing
    )
# No add_conditional_edges needed — routing is embedded in the return value
```

Use `Command` when the same node decides both what to update AND where to go next. Use `add_conditional_edges` when routing is based on state set by a previous node.

---

## Concept 5 — Checkpointing

**Analogy:** Git for agent state. Every node execution is a commit. You can roll back to any point, inspect state, branch, and replay.

Checkpointer saves full state after every node. The `thread_id` in config identifies a conversation.

```python
from langgraph.checkpoint.memory import InMemorySaver  # dev only — lost on restart

memory = InMemorySaver()
graph = builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "user-123"}}
result = graph.invoke({"messages": [...]}, config)

# Continue the same conversation
result = graph.invoke({"messages": [HumanMessage("follow-up")]}, config)
```

**Production — AsyncPostgresSaver:**
```python
# pip install langgraph-checkpoint-postgres psycopg[binary]
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/mydb"

async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
    await checkpointer.setup()  # creates tables — run ONCE in deploy pipeline

    graph = builder.compile(checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "session-abc"}}
    result = await graph.ainvoke({"messages": [...]}, config)
```

**Time-travel debugging:**
```python
# Get state history (most recent first)
history = list(graph.get_state_history(config))

# Inspect a past snapshot
snapshot = history[2]
print(snapshot.values)   # state dict at that point
print(snapshot.next)     # which node runs next

# Replay from a past checkpoint
graph.invoke(None, snapshot.config)

# Fork: modify state at a past point
fork_config = graph.update_state(
    snapshot.config,
    {"messages": [HumanMessage("different question")]}
)
graph.invoke(None, fork_config)
```

**Critical:** `AsyncPostgresSaver` requires `psycopg` v3 (not `psycopg2`). Install: `pip install psycopg[binary]`.

---

## Concept 6 — Human-in-the-Loop (HITL)

**Analogy:** A manager approval step. The agent does work, pauses, sends a summary to the human, waits for their decision, then resumes. Requires a checkpointer — the pause state must be persisted.

**Modern pattern — `interrupt()` inside a node (preferred):**
```python
from langgraph.types import interrupt, Command

def human_review(state: State) -> dict:
    # Pauses here. The value is shown to the human.
    decision = interrupt({
        "summary": state["draft"],
        "question": "Approve this draft?"
    })
    # Resumes here after Command(resume=...) is sent
    return {"approved": decision == "yes", "human_feedback": decision}

memory = InMemorySaver()
graph = builder.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "review-1"}}

# First run — pauses at interrupt()
result = graph.invoke({"draft": "Here's my plan..."}, config)
# result contains interrupt metadata — graph is paused

# Human reviews, then resumes
result = graph.invoke(Command(resume="yes"), config)
```

**Compile-time breakpoints (older pattern — still works):**
```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["human_review_node"],
)
config = {"configurable": {"thread_id": "t1"}}

graph.invoke({"value": 5}, config)

# Optionally modify state before resuming
graph.update_state(config, {"value": 99, "feedback": "change this"})

# Resume
graph.invoke(Command(resume=None), config)
```

**Key difference:** `interrupt()` is more composable — it can appear anywhere in a node body, can pass structured context to the human, and doesn't require compile-time configuration. Prefer it for production.

---

## Concept 7 — Streaming

Four stream modes. Use `version="v2"` — it's the unified 2025 format.

| Mode | What you get | Use for |
|------|-------------|---------|
| `"messages"` | LLM tokens + metadata | Token-by-token UI |
| `"updates"` | `{node_name: output}` per step | Debugging |
| `"values"` | Full state after each step | Chatbot state |
| `["updates","values"]` | Both combined | Monitoring |

**Token streaming (preferred in 2025-2026):**
```python
async for chunk in graph.astream(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config,
    stream_mode="messages",
    version="v2",
):
    if chunk["type"] == "messages":
        msg_chunk, metadata = chunk["data"]
        # Filter to specific node to avoid tokens from all LLM calls
        if msg_chunk.content and metadata["langgraph_node"] == "chatbot":
            print(msg_chunk.content, end="", flush=True)
```

**State updates per node:**
```python
async for chunk in graph.astream(
    {"messages": [...]},
    config,
    stream_mode="updates",
):
    for node_name, node_output in chunk.items():
        print(f"{node_name}: {node_output}")
```

**`astream_events` v2 (still available — more granular event types):**
```python
async for event in graph.astream_events(
    {"messages": [...]},
    config,
    version="v2",
):
    if event["event"] == "on_chat_model_stream":
        content = event["data"]["chunk"].content
        if content:
            print(content, end="", flush=True)
    elif event["event"] == "on_tool_start":
        print(f"\nCalling: {event['name']}")
```

**Subgraph streaming:**
```python
async for chunk in graph.astream(
    {"foo": "start"},
    stream_mode="updates",
    subgraphs=True,    # include events from subgraph nodes
    version="v2",
):
    pass
```

---

## Concept 8 — Subgraphs

A compiled `StateGraph` can be used as a node in a parent graph. This enables logical separation and reuse.

**Shared state keys (simplest):**
```python
class State(TypedDict):
    foo: str

def subgraph_node(state: State) -> dict:
    return {"foo": "processed: " + state["foo"]}

sub_builder = StateGraph(State)
sub_builder.add_node(subgraph_node)
sub_builder.add_edge(START, "subgraph_node")
sub_builder.add_edge("subgraph_node", END)
subgraph = sub_builder.compile()

parent_builder = StateGraph(State)
parent_builder.add_node("step_1", step_1)
parent_builder.add_node("sub_step", subgraph)  # compiled graph as node
parent_builder.add_edge(START, "step_1")
parent_builder.add_edge("step_1", "sub_step")
parent_builder.add_edge("sub_step", END)
parent_graph = parent_builder.compile()
```

**Private keys (subgraph has internal state parent doesn't see):**
```python
class SubState(TypedDict):
    foo: str   # shared — bridges to parent
    bar: str   # private — only inside subgraph

class ParentState(TypedDict):
    foo: str   # only the shared key

# Subgraph nodes can read/write bar internally
# Only foo is visible to parent after subgraph exits
```

**When to use subgraphs:**
- Logical separation (research phase, writing phase, review phase)
- Reusable agent components across multiple parent graphs
- When a subsystem needs private intermediate state
- Streaming from subgraphs: pass `subgraphs=True` to `astream()`

---

## Concept 9 — RetryPolicy

Per-node automatic retry for transient failures. Configured at `add_node` time.

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "call_llm",
    call_llm_node,
    retry_policy=RetryPolicy(
        max_attempts=5,
        initial_interval=1.0,    # seconds before first retry
        backoff_factor=2.0,      # 1s, 2s, 4s, 8s...
        max_interval=60.0,       # cap at 60 seconds
        jitter=True,             # randomize to avoid thundering herd
        retry_on=(               # only retry these exception types
            ConnectionError,
            TimeoutError,
        ),
    )
)
```

**Known bug (GitHub #6027, April 2026):** `RetryPolicy` does NOT retry `pydantic.ValidationError` even when explicitly listed in `retry_on`. Do not rely on it for Pydantic errors.

**Do NOT add RetryPolicy to nodes with side effects** (email sending, payment processing) — they will execute multiple times on retry.

---

## Concept 10 — Command Primitive

`Command` is the modern unified return type: combine state update + routing in one value. Replaces the pattern of: update state in node → read state in conditional edge → route.

```python
from langgraph.types import Command
from typing_extensions import Literal

def router(state: State) -> Command[Literal["fast_path", "slow_path"]]:
    if state["score"] > 0.9:
        return Command(update={"path": "fast"}, goto="fast_path")
    else:
        return Command(update={"path": "slow"}, goto="slow_path")

# No add_conditional_edges needed — Command handles routing
```

**Command from subgraph to parent:**
```python
from langgraph.constants import PARENT

def subgraph_node(state: SubState) -> Command:
    return Command(
        update={"result": "done"},
        goto="parent_node_name",
        graph=PARENT,
    )
```

**Command to resume HITL:**
```python
graph.invoke(Command(resume="human approval text"), config)
```

---

## Concept 11 — Send API (Fan-Out / Map-Reduce)

`Send` creates dynamic fan-out — N parallel copies of a node, each with its own input. Results aggregate back via a reducer.

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END

class OverallState(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]  # reducer aggregates results

class WorkerState(TypedDict):
    subject: str

def generate_joke(state: WorkerState) -> dict:
    return {"jokes": [f"Why did the {state['subject']} cross the road? To get to the other side!"]}

def fan_out(state: OverallState) -> list[Send]:
    # Return list of Send objects — one per parallel worker
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

builder = StateGraph(OverallState)
builder.add_node("generate_joke", generate_joke)
builder.add_conditional_edges(START, fan_out)    # fan_out IS the edge function
builder.add_edge("generate_joke", END)

graph = builder.compile()
result = graph.invoke({"subjects": ["cat", "dog", "fish"]})
# All 3 workers run in parallel. result["jokes"] has 3 items.
```

**Rules:**
- Worker return dict keys must match parent state keys with `operator.add` reducer — otherwise parallel results overwrite each other silently
- All `Send`'d nodes run in parallel in the same superstep, then block until ALL complete
- Use only for independent tasks — sequential dependencies gain nothing

---

## Concept 12 — Prebuilt Agent

**Status (April 2026):** `create_react_agent` is deprecated in favor of `create_agent` from `langchain.agents`, but `create_react_agent` still works with a deprecation warning. `create_agent` has missing features per community reports — use `create_react_agent` for full feature parity until migration is complete.

```python
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

@tool
def get_weather(location: str) -> str:
    """Get the current weather for a location."""
    return f"72°F and sunny in {location}"

model = ChatAnthropic(model="claude-sonnet-4-6")
memory = InMemorySaver()

agent = create_react_agent(
    model,
    tools=[get_weather],
    prompt="You are a helpful assistant.",  # system prompt
    checkpointer=memory,
)

config = {"configurable": {"thread_id": "session-1"}}
result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Weather in Tokyo?"}]},
    config
)
```

**Prebuilt vs custom graph:**
- Use `create_react_agent`: quick prototype, simple tool loop, no custom routing
- Use custom `StateGraph`: multi-agent orchestration, HITL at specific steps, parallel tool execution, custom routing logic, production agents

---

## Concept 13 — Persistence and Memory (Checkpointer vs Store)

**Analogy:** Checkpointer = RAM (fast, per-session, lost when session ends). Store = disk (slower, cross-session, persists for the user).

| | Checkpointer | Store |
|---|---|---|
| **Stores** | Full graph state per step | Arbitrary key-value data |
| **Scope** | Per `thread_id` | Cross-thread (user-level) |
| **Use for** | Conversation history, pause/resume | User preferences, long-term memory |
| **Dev class** | `InMemorySaver` | `InMemoryStore` |
| **Prod class** | `AsyncPostgresSaver` | `AsyncPostgresStore` |

**Using both in production:**
```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.store.postgres.aio import AsyncPostgresStore  # pip install langgraph-store-postgres
from langgraph.runtime import Runtime
from dataclasses import dataclass
import uuid

@dataclass
class AppContext:
    user_id: str

async def agent_node(state: MessagesState, runtime: Runtime[AppContext]) -> dict:
    user_id = runtime.context.user_id

    # Query cross-thread memories for this user
    memories = await runtime.store.asearch(
        ("memories", user_id),
        query=str(state["messages"][-1].content)
    )
    memory_text = "\n".join(m.value["data"] for m in memories)

    response = await anthropic_client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=f"User context:\n{memory_text}",
        messages=[{"role": m.type, "content": m.content} for m in state["messages"]],
    )

    # Optionally save new memory
    await runtime.store.aput(
        ("memories", user_id),
        str(uuid.uuid4()),
        {"data": "User prefers concise answers"}
    )

    return {"messages": [AIMessage(response.content[0].text)]}

DB_URI = "postgresql://user:pass@localhost:5432/mydb"

async with (
    AsyncPostgresStore.from_conn_string(DB_URI) as store,
    AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
):
    await store.setup()        # run ONCE in deploy pipeline
    await checkpointer.setup() # run ONCE in deploy pipeline

    graph = builder.compile(checkpointer=checkpointer, store=store)

    config = {"configurable": {"thread_id": "thread-1"}}
    result = await graph.ainvoke(
        {"messages": [...]},
        config,
        context=AppContext(user_id="user-42"),
    )
```

---

## Concept 14 — LangGraph Platform

Brief overview — full ops layer for deploying graphs as services.

**LangGraph Server:** Wraps your compiled graph in a REST API. Key endpoints:
- `POST /runs/stream` — stream execution
- `POST /threads` — create thread
- `GET /threads/{id}/state` — get state

**Deploy manifest (`langgraph.json`):**
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent.py:graph"
  },
  "env": ".env"
}
```

**Local dev server:**
```bash
pip install langgraph-cli
langgraph dev  # starts server + Studio UI at localhost:8123
```

**LangGraph Studio:** Visual debugger — shows graph structure, state per node, allows manual state injection and replay from any checkpoint. Use it for debugging complex agent behavior.

**SDK for external access:**
```python
from langgraph_sdk import get_sync_client

client = get_sync_client(url="https://your-deployment.langsmith.com", api_key="...")
for chunk in client.runs.stream(None, "agent", input={"messages": [...]}, stream_mode="updates"):
    print(chunk.event, chunk.data)
```

---

## Key Imports Reference

```python
# Graph building
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.graph.message import add_messages

# Types
from langgraph.types import Command, Send, interrupt, RetryPolicy

# Checkpointing — dev
from langgraph.checkpoint.memory import InMemorySaver

# Checkpointing — production
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver  # pip install langgraph-checkpoint-postgres psycopg[binary]

# Store — dev
from langgraph.store.memory import InMemoryStore

# Store — production
from langgraph.store.postgres.aio import AsyncPostgresStore  # pip install langgraph-store-postgres

# Runtime context (v1.0+)
from langgraph.runtime import Runtime

# LangChain Anthropic integration
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage, AnyMessage

# Standard
from typing import Annotated
import operator
from typing_extensions import TypedDict, Literal
```

---

## Breaking Changes (as of April 2026)

| Old | New | Status |
|-----|-----|--------|
| `MemorySaver` | `InMemorySaver` | Alias still works |
| `create_react_agent` | `create_agent` (langchain.agents) | Deprecated, not broken. Migration incomplete — use `create_react_agent` until `create_agent` reaches feature parity |
| `astream_events(version="v1")` | `astream(stream_mode="messages", version="v2")` | v1 deprecated |
| `interrupt_before` at compile | `interrupt()` inside node | Both work; `interrupt()` preferred |
| `config["configurable"]` for context | `context_schema=` + `Runtime[Context]` | Both work; new pattern preferred |

---

## Connection Map

```
CORE                     ADVANCED                 PRODUCTION
  StateGraph               Command (routing)        AsyncPostgresSaver
  State + reducers         Send (fan-out)           AsyncPostgresStore
  add_messages             Subgraphs                interrupt() + HITL
  Nodes (async)            RetryPolicy              astream (v2)
  Conditional edges        Prebuilt agent           LangGraph Platform
  Checkpointing
```

**Anthropic SDK connection:**
- `messages` state key → passed directly as `messages=` in `client.messages.create()`
- `AIMessage` from LangChain ↔ assistant content block from Anthropic SDK
- `ToolMessage` ↔ `tool_result` content block
- LangGraph handles the loop; Anthropic SDK handles the actual LLM call inside each node
