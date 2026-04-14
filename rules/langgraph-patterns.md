---
description: LangGraph rules. Loads for all projects using stateful agent workflows.
paths: ["*.py", "app/**/*.py", "agents/**/*.py", "graphs/**/*.py"]
---

# LangGraph Rules (L10)

> Anthropic SDK integration → see `anthropic-patterns.md`
> Async patterns → see `async-patterns.md`
> Credentials from env → see `security.md`

> Version: LangGraph v1.0 (GA Oct 2025). Verified April 2026.

---

## StateGraph Setup

1. Always import `START` and `END` from `langgraph.graph` — missing them causes compile failure, not runtime failure
2. Wire every path: every node must be reachable from `START` and every path must reach `END` or `__end__`
3. Call `compile()` before `invoke()` — `compile()` validates the graph structure, catches missing edges early
4. Use `async def` nodes for all I/O-bound work (LLM calls, DB queries) — never block the event loop
5. NEVER mutate the `state` dict in place inside a node — always return a new dict with only changed keys

## State Design

6. Use `Annotated[list[AnyMessage], add_messages]` for all chat history fields — `add_messages` appends and de-duplicates by message id
7. Use `MessagesState` shorthand when the only state is a message list — no need to define a custom TypedDict
8. Apply `Annotated[list[T], operator.add]` to any list field written by parallel nodes — without it, parallel nodes silently overwrite each other (last writer wins)
9. Nodes return ONLY the keys they changed — returning the full state dict on every node is wasteful and masks bugs
10. NEVER store secrets or PII in graph state — state is persisted to checkpoints; it is not ephemeral

## Checkpointing

11. Use `InMemorySaver` for dev/testing ONLY — it is lost on process restart; NEVER deploy it
12. Use `AsyncPostgresSaver` in production — requires `psycopg` v3 (NOT `psycopg2`): `pip install langgraph-checkpoint-postgres psycopg[binary]`
13. Call `await checkpointer.setup()` ONCE before first use — creates required tables; run in deploy pipeline, NOT in lifespan
14. Always pass `config = {"configurable": {"thread_id": "..."}}` when using a checkpointer — without `thread_id`, checkpointing silently does nothing
15. NEVER reuse the same `thread_id` for different users — thread isolation is total; threads cannot see each other's state
16. Run `checkpointer.setup()` in the deploy pipeline (like alembic upgrade head), NOT in application startup

## Human-in-the-Loop

17. Use `interrupt()` inside a node (not `interrupt_before` at compile time) — it's more composable, testable, and lets you pass context to the human reviewer:
    ```python
    from langgraph.types import interrupt
    decision = interrupt({"summary": state["draft"], "question": "Approve?"})
    ```
18. `interrupt()` requires a checkpointer — without one it raises an error at runtime
19. Resume an interrupted graph with `Command(resume=value)` — NOT by calling `invoke()` with `None`:
    ```python
    graph.invoke(Command(resume="approved"), config)
    ```
20. Inject modified state before resuming: `graph.update_state(config, {...})` then `invoke(Command(resume=None), config)`
21. NEVER call `interrupt()` in an edge function or router — only inside node bodies

## Conditional Edges and Routing

22. Use `Command(update={...}, goto="node_name")` when the routing decision and state update are tightly coupled in the same node — cleaner than add_conditional_edges in this case
23. Use `add_conditional_edges` when routing is based on state set by a previous node (router reads, not sets)
24. Router function return values must exactly match registered node names — a typo causes `KeyError` at runtime, not compile time
25. Always annotate router return type with `Literal["node_a", "node_b", ...]` — enables static analysis

## Streaming

26. Use `stream_mode="messages"` with `version="v2"` for token streaming — it's the 2025 standard:
    ```python
    async for chunk in graph.astream(input, config, stream_mode="messages", version="v2"):
        if chunk["type"] == "messages":
            msg_chunk, metadata = chunk["data"]
            if msg_chunk.content and metadata["langgraph_node"] == "chatbot":
                yield msg_chunk.content
    ```
27. Always filter by `metadata["langgraph_node"]` when streaming from multi-node graphs — without filtering you get tokens from every LLM call in the graph
28. Use `astream_events(version="v2")` when you need fine-grained event types (tool start/end, retriever events) — not needed for simple token streaming
29. NEVER use `version="v1"` — deprecated
30. Pass `subgraphs=True` to `astream()` when you need to see events from inside subgraphs

## Subgraphs

31. Subgraph must share at least one state key with parent — without a shared key, subgraph cannot read or write parent state
32. Use subgraphs for: logical phase separation, reusable agent components, private intermediate state
33. NEVER use subgraphs for simple sequential pipelines — just chain nodes
34. For subgraph-to-parent routing: use `Command(goto="parent_node", graph=PARENT)` — import `PARENT` from `langgraph.constants`

## RetryPolicy

35. Attach `RetryPolicy` only to nodes that make external calls (LLM, HTTP, DB) — NOT to pure computation nodes
36. Always scope `retry_on` to specific exception types — retrying `ValueError` hides logic bugs:
    ```python
    retry_policy=RetryPolicy(max_attempts=3, retry_on=(ConnectionError, TimeoutError))
    ```
37. NEVER add `RetryPolicy` to nodes with side effects (email, payment, message send) — retries will duplicate the action
38. Do NOT rely on `RetryPolicy` to catch `pydantic.ValidationError` — known bug (GitHub #6027, April 2026): it won't retry even if listed

## Fan-Out with Send

39. Use `Send` for map-reduce — dynamic fan-out to N parallel workers from a conditional edge:
    ```python
    from langgraph.types import Send
    def fan_out(state: State) -> list[Send]:
        return [Send("worker", {"item": x}) for x in state["items"]]
    builder.add_conditional_edges(START, fan_out)
    ```
40. Worker return dict keys MUST match parent state keys with `operator.add` reducer — without it, parallel workers silently overwrite each other
41. All `Send`'d nodes run in parallel in one superstep and block until ALL complete — use only for independent tasks

## Memory (Checkpointer vs Store)

42. Use checkpointer for within-thread state (current conversation, pause/resume) — keyed by `thread_id`
43. Use `AsyncPostgresStore` for cross-thread memory (user preferences, long-term facts) — persists across separate sessions:
    ```python
    await runtime.store.asearch(("memories", user_id), query="...")
    await runtime.store.aput(("memories", user_id), str(uuid4()), {"data": "..."})
    ```
44. Access store via `runtime: Runtime[Context]` in node signature — not via closure or global
45. Call `await store.setup()` ONCE in deploy pipeline — NOT in application startup
46. NEVER store user PII in the store without a deletion endpoint — right to be forgotten requires explicit memory delete

## Prebuilt Agent

47. Use `create_react_agent` (not `create_agent`) as of April 2026 — `create_agent` has missing features per community reports (migration incomplete):
    ```python
    from langgraph.prebuilt import create_react_agent
    ```
48. Use prebuilt agent only for: quick prototypes, simple tool-calling loops, no custom routing
49. Use custom `StateGraph` for: multi-agent orchestration, HITL at specific steps, parallel execution, production agents

## LangGraph Platform

50. Use `langgraph dev` for local development — starts LangGraph Server + Studio at localhost:8123
51. Always provide `langgraph.json` manifest — maps graph names to Python objects for the server
52. Run `checkpointer.setup()` and `store.setup()` in the deploy pipeline, NOT in the server startup hook
53. NEVER run `alembic upgrade head` and `checkpointer.setup()` inside the same lifespan — keep infra setup out of application code
54. Use `langgraph build -t my-agent:latest` to create a Docker image of your graph — same API as LangGraph Cloud but self-hosted
55. LangGraph Studio allows manual state injection at any node — use it to test edge cases by editing state mid-run without rewriting test cases
56. Background runs via LangGraph Server: `POST /runs` returns `run_id` immediately; poll `GET /runs/{run_id}` for completion — use for long-running agents where clients cannot hold open a connection
57. LangGraph Cloud auto-scales workers; self-hosted LangGraph Server uses fixed workers — size workers = max concurrent graph runs you need

## Observability (LangSmith Auto-Tracing)

58. Enable LangSmith tracing with zero code changes — set two env vars, every LangGraph run is auto-traced:
    ```bash
    LANGSMITH_TRACING=true
    LANGSMITH_API_KEY=lsv2_...
    LANGSMITH_PROJECT=my-agent   # optional — groups runs
    ```
59. NEVER skip observability in production LangGraph agents — without it you cannot debug agent failures, see token costs, or trace tool call chains
60. LangSmith traces every node, every LLM call, every tool call automatically — no `@observe` decorators needed for LangGraph (unlike raw Anthropic SDK usage)

## Context Window Management

61. Use `trim_messages` to keep the messages list within token budget for long-running chat agents:
    ```python
    from langchain_core.messages import trim_messages

    def chatbot(state: MessagesState) -> dict:
        trimmed = trim_messages(
            state["messages"],
            max_tokens=50000,
            strategy="last",           # keep most recent messages
            token_counter=ChatAnthropic(model="claude-sonnet-4-6"),
            include_system=True,       # always keep system message
            allow_partial=False,
        )
        response = model.invoke(trimmed)
        return {"messages": response}
    ```
62. NEVER pass unbounded `state["messages"]` to the LLM in a long-running agent — message lists grow unbounded with checkpointer; always trim first

## Input/Output Schema

63. Use `input` and `output` schema parameters on `StateGraph` to restrict what external callers see vs. what nodes share internally:
    ```python
    class InputState(TypedDict):
        question: str          # only this is accepted from external caller

    class OutputState(TypedDict):
        answer: str            # only this is returned to external caller

    class InternalState(InputState, OutputState):
        intermediate_result: str   # internal only — nodes can use this

    builder = StateGraph(InternalState, input=InputState, output=OutputState)
    ```
64. Use input/output schema for subgraphs with a public API — it makes the interface explicit and prevents leaking internal state to the parent graph

## Functional API (@entrypoint / @task)

65. Use the functional API (`@entrypoint`, `@task`) for simpler linear flows — less boilerplate than StateGraph when you don't need conditional routing or parallel fan-out:
    ```python
    from langgraph.func import entrypoint, task

    @task
    async def call_llm(messages: list) -> str:
        response = await model.ainvoke(messages)
        return response.content

    @task
    async def call_tool(tool_input: dict) -> str:
        return tool_fn(**tool_input)

    @entrypoint(checkpointer=InMemorySaver())
    async def pipeline(state: dict) -> dict:
        result = await call_llm(state["messages"])
        return {"answer": result}
    ```
66. Use `StateGraph` for: conditional routing, parallel fan-out, multi-agent orchestration, HITL
    Use `@entrypoint/@task` for: sequential pipelines, simple chat loops, tasks without branching

## Production Connection Pooling

67. In production, use `AsyncConnectionPool` for `AsyncPostgresSaver` — share one pool across app components instead of opening a new connection per graph run:
    ```python
    from psycopg_pool import AsyncConnectionPool
    from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

    # Create once at lifespan startup — shared across all requests
    pool = AsyncConnectionPool(
        conninfo="postgresql://user:pass@localhost:5432/mydb",
        max_size=20,
        kwargs={"autocommit": True, "prepare_threshold": 0},
    )
    checkpointer = AsyncPostgresSaver(pool)
    await checkpointer.setup()  # run once in deploy pipeline

    graph = builder.compile(checkpointer=checkpointer)
    ```
68. NEVER use `AsyncPostgresSaver.from_conn_string()` with an async context manager in a hot path — it opens and closes a connection per call; use shared pool instead

## Parallel Execution (Supersteps)

69. LangGraph runs nodes in **supersteps**: all nodes scheduled to run at the same time execute in parallel within one superstep, then block until ALL complete before the next superstep starts
70. Use `max_concurrency` on `compile()` to cap how many nodes run simultaneously — critical when parallel nodes each make LLM calls (rate limit risk):
    ```python
    graph = builder.compile(checkpointer=checkpointer, max_concurrency=5)
    ```
71. NEVER use `Send` fan-out without `max_concurrency` if workers call external APIs — unbounded parallelism will hit rate limits

## Time-Travel (update_state)

72. Always pass `as_node=` when calling `graph.update_state()` for time-travel — it tells LangGraph which node "made" this update and determines which node runs next:
    ```python
    graph.update_state(
        config,
        {"feedback": "rejected"},
        as_node="human_review",   # next node after human_review will run next
    )
    graph.invoke(None, config)
    ```
73. Without `as_node=`, LangGraph defaults to the last executed node — which is often wrong for HITL scenarios where you want execution to continue from a specific point

## Anthropic SDK Integration

74. The `messages` state key IS the conversation history — pass it directly to `client.messages.create(messages=[...])`
75. `AIMessage` from LangChain maps to `{"role": "assistant", "content": "..."}` in Anthropic SDK
76. `ToolMessage` maps to a `tool_result` content block — `tool_use_id` must match
77. Initialize `AsyncAnthropic` ONCE in lifespan, inject into graph via `Runtime[Context]` or app state — see `anthropic-patterns.md` rule 1

## ToolNode and Tool Execution

78. Use `ToolNode` from `langgraph.prebuilt` for tool execution in custom graphs — handles dispatch, parallel calls, error catching, and `ToolMessage` construction automatically:
    ```python
    from langgraph.prebuilt import ToolNode, tools_condition

    tools = [get_weather, search_web]
    tool_node = ToolNode(tools)

    builder = StateGraph(MessagesState)
    builder.add_node("agent", call_model)
    builder.add_node("tools", tool_node)
    builder.add_edge(START, "agent")
    builder.add_conditional_edges("agent", tools_condition)  # → "tools" if tool_calls, END otherwise
    builder.add_edge("tools", "agent")                       # loop back after execution
    ```
79. `tools_condition` is a prebuilt router that checks `state["messages"][-1].tool_calls` — returns `"tools"` if present, `END` otherwise; use it instead of writing your own
80. `ToolNode` handles parallel tool calls — if the LLM calls 3 tools in one turn, `ToolNode` executes them concurrently and returns all results
81. NEVER manually build `ToolMessage` objects when using `ToolNode` — it constructs them automatically; manual construction only needed when bypassing `ToolNode`

## Supervisor Multi-Agent Pattern

82. The supervisor pattern: one LLM routes between specialist subagents, each subagent returns to supervisor after completing:
    ```python
    from typing_extensions import Literal
    from pydantic import BaseModel

    class RouteDecision(BaseModel):
        next: Literal["researcher", "writer", "FINISH"]

    def supervisor(state: MessagesState) -> Command[Literal["researcher", "writer", "__end__"]]:
        decision = model.with_structured_output(RouteDecision).invoke(state["messages"])
        goto = END if decision.next == "FINISH" else decision.next
        return Command(goto=goto)

    builder = StateGraph(MessagesState)
    builder.add_node("supervisor", supervisor)
    builder.add_node("researcher", create_react_agent(model, [search_tool]))
    builder.add_node("writer", create_react_agent(model, [write_tool]))
    builder.add_edge(START, "supervisor")
    builder.add_edge("researcher", "supervisor")  # always return to supervisor
    builder.add_edge("writer", "supervisor")
    ```
83. Supervisor MUST eventually route to `END` — without a FINISH condition and max_iterations guard the supervisor loops forever

## Node Failure Recovery

84. When a node raises an unhandled exception: graph execution stops AND the checkpointer preserves all state up to that point — no completed work is lost
85. Recover from a failed node without losing progress:
    ```python
    # Inspect state at failure point
    snapshot = graph.get_state(config)
    print(snapshot.values)  # what state looked like when node failed
    print(snapshot.next)    # which node was running

    # Fix: update state to bypass bad input, then resume
    graph.update_state(config, {"bad_field": "fixed value"}, as_node="failed_node")
    graph.invoke(None, config)  # continues from that node with fixed state

    # Or: fix the code, redeploy — the checkpoint is still there, resume works
    graph.invoke(None, config)
    ```
86. NEVER run production agents without a checkpointer — an unchecked node failure with no checkpointer loses all progress and there is no recovery path

## Graph Visualization

87. Visualize graph structure during development to catch missing edges and verify routing logic:
    ```python
    # In Jupyter notebook
    from IPython.display import Image, display
    display(Image(graph.get_graph().draw_mermaid_png()))

    # Save to file
    with open("graph.png", "wb") as f:
        f.write(graph.get_graph().draw_mermaid_png())

    # Print Mermaid source (no image dependency)
    print(graph.get_graph().draw_mermaid())
    ```
88. Use `xray=True` to expand subgraph internals: `graph.get_graph(xray=True).draw_mermaid_png()`

## InjectedState and InjectedStore (Tools That Read Graph State)

89. Use `InjectedState` to pass current graph state into a tool function — the injected parameter is hidden from the LLM's tool schema (LLM doesn't see it, ToolNode injects it automatically):
    ```python
    from langgraph.prebuilt import InjectedState, InjectedStore, ToolNode
    from langchain_core.tools import tool
    from typing import Annotated

    @tool
    def get_user_context(
        query: str,
        state: Annotated[dict, InjectedState],  # hidden from LLM schema
    ) -> str:
        """Search for relevant context given a query."""
        user_id = state.get("user_id")
        return f"Context for user {user_id}: ..."

    tool_node = ToolNode([get_user_context])  # ToolNode injects state automatically
    ```
90. Use `InjectedStore` to give tools direct access to the long-term store — the tool can read/write memories without going through node state:
    ```python
    from langgraph.store.base import BaseStore

    @tool
    async def save_preference(
        preference: str,
        store: Annotated[BaseStore, InjectedStore],
        state: Annotated[dict, InjectedState],
    ) -> str:
        """Save a user preference to long-term memory."""
        user_id = state["user_id"]
        await store.aput(("prefs", user_id), preference[:20], {"data": preference})
        return f"Saved: {preference}"
    ```
91. `InjectedState` and `InjectedStore` ONLY work when calling tools via `ToolNode` — if you call tools manually, you must inject these yourself

## ToolNode Error Handling

92. `ToolNode` catches tool exceptions by default (`handle_tool_errors=True`) and returns the error message as a `ToolMessage` — the LLM sees the error and can retry or route differently:
    ```python
    # Default — catches all exceptions, returns error string to LLM
    tool_node = ToolNode(tools)

    # Disable — exceptions propagate up (triggers RetryPolicy or fails the run)
    tool_node = ToolNode(tools, handle_tool_errors=False)

    # Custom handler — format the error yourself
    tool_node = ToolNode(tools, handle_tool_errors=lambda e: f"Tool failed: {type(e).__name__}: {e}")
    ```
93. NEVER disable `handle_tool_errors` unless you have a `RetryPolicy` on the tool node — unhandled tool exceptions will crash the graph run

## Swarm / Handoff Pattern

94. The swarm pattern: agents hand off directly to each other via `Command` — no central supervisor, each agent decides the next agent:
    ```python
    def researcher(state: MessagesState) -> Command[Literal["writer", "__end__"]]:
        response = model.invoke(state["messages"])
        if needs_writing(response):
            return Command(
                update={"messages": [response]},
                goto="writer",
            )
        return Command(update={"messages": [response]}, goto=END)

    def writer(state: MessagesState) -> Command[Literal["researcher", "__end__"]]:
        response = model.invoke(state["messages"])
        if needs_more_research(response):
            return Command(update={"messages": [response]}, goto="researcher")
        return Command(update={"messages": [response]}, goto=END)
    ```
95. Use supervisor when you need centralized control and audit trail — use swarm when agents are peers and handoff logic is local to each agent
96. Swarm graphs MUST have a termination condition in every agent — without `goto=END` paths the graph loops forever

## Custom Streaming (StreamWriter)

97. Use `get_stream_writer()` inside a node to push custom data to the stream without including it in graph state — use for progress updates in long-running nodes:
    ```python
    from langgraph.config import get_stream_writer
    import asyncio

    async def process_documents(state: State) -> dict:
        write = get_stream_writer()  # get writer for this run
        docs = state["documents"]

        results = []
        for i, doc in enumerate(docs):
            result = await process(doc)
            results.append(result)
            write({"progress": f"{i+1}/{len(docs)}", "doc": doc["id"]})  # stream immediately

        return {"results": results}

    # Client reads custom stream
    async for chunk in graph.astream(input, config, stream_mode="custom"):
        print(chunk)  # {"progress": "1/10", "doc": "doc-abc"}
    ```
98. `stream_mode="custom"` only receives `write()` calls — use `stream_mode=["custom", "updates"]` to receive both custom events and node output events

## Store: Memory Deletion (GDPR)

99. Implement memory deletion with `store.adelete()` — close the loop on rule 46 (right to be forgotten):
    ```python
    from langgraph.store.postgres.aio import AsyncPostgresStore

    async def delete_user_memories(user_id: str, store: AsyncPostgresStore) -> int:
        # List all items in user's namespace
        namespace = ("memories", user_id)
        items = await store.alist(namespace)
        # Delete each one
        for item in items:
            await store.adelete(namespace, item.key)
        return len(items)
    ```
100. `store.alist(namespace_prefix)` lists all items under a namespace — use it to enumerate before bulk delete
101. Namespace structure is a tuple of strings: `("memories", user_id)` or `("memories", user_id, "preferences")` — use multi-level namespaces for fine-grained scoping

## Testing LangGraph Graphs

102. Use `InMemorySaver` (not Postgres) for test isolation — each test gets a fresh in-memory checkpointer with no shared state:
     ```python
     import pytest
     from langgraph.checkpoint.memory import InMemorySaver

     def test_chatbot_responds():
         checkpointer = InMemorySaver()
         graph = builder.compile(checkpointer=checkpointer)
         config = {"configurable": {"thread_id": "test-thread-1"}}

         result = graph.invoke(
             {"messages": [HumanMessage("Hello")]},
             config
         )
         assert result["messages"][-1].content != ""

     def test_multi_turn_memory():
         checkpointer = InMemorySaver()
         graph = builder.compile(checkpointer=checkpointer)
         config = {"configurable": {"thread_id": "test-thread-2"}}

         graph.invoke({"messages": [HumanMessage("My name is Alice")]}, config)
         result = graph.invoke({"messages": [HumanMessage("What's my name?")]}, config)
         assert "Alice" in result["messages"][-1].content
     ```
103. Use a unique `thread_id` per test — shared `thread_id` across tests will accumulate state and cause flaky tests
104. Mock LLM calls at the `ChatAnthropic` or `AsyncAnthropic` level, not at HTTP level — same pattern as `anthropic-patterns.md` rule for testing
105. Test HITL flows by asserting on `snapshot.next` after the first `invoke()` call, then calling `invoke(Command(resume=...), config)` and asserting on the final state

## Recursion Limit

106. LangGraph's default recursion limit is **25 steps** — any agent that loops (tool use, supervisor routing, reflection) will raise `GraphRecursionError` if it exceeds this; raise the limit in config:
     ```python
     config = {
         "recursion_limit": 100,          # raise from default 25
         "configurable": {"thread_id": "..."},
     }
     result = graph.invoke(input, config)
     ```
107. Catch `GraphRecursionError` explicitly and return a graceful degraded response — never let it surface as an unhandled 500:
     ```python
     from langgraph.errors import GraphRecursionError

     try:
         result = await graph.ainvoke(input, config)
     except GraphRecursionError:
         return {"messages": [AIMessage("I couldn't complete this in the allowed steps.")]}
     ```
108. Use `graph.with_config({"recursion_limit": 100})` to set a default limit on the compiled graph so you don't have to pass it on every `invoke()` call

## model.bind_tools() — Giving the LLM Tool Awareness

109. Call `model.bind_tools(tools)` on the LangChain model BEFORE using it in a node — without this the model has no knowledge of tools, generates no `tool_calls`, and `tools_condition` always routes to `END`:
     ```python
     from langchain_anthropic import ChatAnthropic

     tools = [get_weather, search_web]
     model = ChatAnthropic(model="claude-sonnet-4-6")
     model_with_tools = model.bind_tools(tools)   # bind ONCE, reuse

     async def call_model(state: MessagesState) -> dict:
         response = await model_with_tools.ainvoke(state["messages"])
         return {"messages": response}

     tool_node = ToolNode(tools)  # same tools list

     builder.add_node("agent", call_model)
     builder.add_node("tools", tool_node)
     builder.add_conditional_edges("agent", tools_condition)
     builder.add_edge("tools", "agent")
     ```
110. `model.bind_tools(tools)` and `ToolNode(tools)` must use the SAME tools list — if they differ, the model may request a tool that `ToolNode` doesn't know how to execute

## RemoveMessage — Deleting Messages from History

111. Use `RemoveMessage` to delete a specific message from the `add_messages` list by its `id` — the only way to remove a message once it's in state:
     ```python
     from langchain_core.messages import RemoveMessage

     def prune_history(state: MessagesState) -> dict:
         # Remove all messages except the last 10
         messages_to_delete = state["messages"][:-10]
         return {"messages": [RemoveMessage(id=m.id) for m in messages_to_delete]}
     ```
112. `RemoveMessage` works because `add_messages` checks: if a message with the same `id` is a `RemoveMessage`, it deletes the existing one instead of appending
113. NEVER delete the most recent `HumanMessage` or any `ToolMessage` that has a pending `tool_use_id` — the LLM will error on missing tool results

## SQLite Checkpointer (Dev with File Persistence)

114. Use `AsyncSqliteSaver` for local development when you want persistence across process restarts without running PostgreSQL:
     ```python
     # pip install langgraph-checkpoint-sqlite aiosqlite
     from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

     async with AsyncSqliteSaver.from_conn_string("./agent_state.db") as checkpointer:
         await checkpointer.setup()
         graph = builder.compile(checkpointer=checkpointer)
         result = await graph.ainvoke(input, config)
     ```
115. Checkpointer hierarchy: `InMemorySaver` (no persistence) → `AsyncSqliteSaver` (file, single process) → `AsyncPostgresSaver` (production, multi-process)

## Defining Tools (@tool decorator)

116. Define tools with `@tool` from `langchain_core.tools` — docstring is the tool description the LLM sees; type annotations become the input schema:
     ```python
     from langchain_core.tools import tool

     @tool
     def get_weather(location: str) -> str:
         """Get the current weather for a given city or location."""
         return f"72°F and sunny in {location}"

     @tool
     def search_documents(query: str, max_results: int = 5) -> list[str]:
         """Search the document store for relevant content.

         Args:
             query: Natural language search query
             max_results: Maximum number of results to return (default 5)
         """
         return vector_store.search(query, k=max_results)
     ```
117. Use `@tool(args_schema=MyPydanticModel)` when the tool has complex nested input — the Pydantic model becomes the full input schema:
     ```python
     from pydantic import BaseModel
     from langchain_core.tools import tool

     class SearchInput(BaseModel):
         query: str
         filters: dict[str, str] = {}
         limit: int = 10

     @tool(args_schema=SearchInput)
     def advanced_search(query: str, filters: dict[str, str], limit: int) -> list[str]:
         """Search with optional metadata filters."""
         return store.search(query, filters=filters, limit=limit)
     ```
118. Tool docstring quality directly affects LLM tool selection accuracy — be specific: describe what it does, when to use it, and what it returns

## Accessing configurable Values Inside Nodes

119. Access runtime configurable values inside a node via `RunnableConfig` type hint — the standard way to pass user_id, tenant_id, or feature flags without graph state:
     ```python
     from langchain_core.runnables import RunnableConfig

     async def personalized_node(state: MessagesState, config: RunnableConfig) -> dict:
         user_id = config["configurable"].get("user_id", "anonymous")
         tenant = config["configurable"].get("tenant_id")
         # use user_id for personalized behavior
         return {"messages": [...]}

     # Caller passes these at invoke time
     config = {
         "configurable": {
             "thread_id": "session-1",
             "user_id": "user-42",
             "tenant_id": "acme-corp",
         }
     }
     result = graph.invoke(input, config)
     ```
120. `RunnableConfig` and `Runtime[Context]` both work — `RunnableConfig` is the older pattern, widely used in tutorials; `Runtime[Context]` is v1.0+, provides type safety. Either is correct.

## checkpoint_id — Direct Checkpoint Replay

121. Use `checkpoint_id` in config to replay from a specific checkpoint, not just the latest state in a thread:
     ```python
     # Get history and pick a specific point
     history = list(graph.get_state_history(config))
     target = history[3]  # e.g., 4th checkpoint from most recent

     # Replay from EXACTLY that checkpoint
     replay_config = {
         "configurable": {
             "thread_id": "session-1",
             "checkpoint_id": target.config["configurable"]["checkpoint_id"],
         }
     }
     result = graph.invoke(None, replay_config)
     ```
122. Without `checkpoint_id`, `invoke(None, config)` always continues from the LATEST checkpoint in that thread — `checkpoint_id` is required for true point-in-time replay

## Store Semantic Search Configuration

123. `store.asearch(namespace, query="...")` only performs **semantic similarity** search if the store is configured with an embedding function — without it, search falls back to keyword/exact matching:
     ```python
     from langchain_anthropic import ChatAnthropic
     from langgraph.store.memory import InMemoryStore

     # Dev: InMemoryStore with embeddings
     store = InMemoryStore(
         index={
             "embed": ChatAnthropic(model="claude-sonnet-4-6").as_embeddings(),
             "dims": 1536,
             "fields": ["data"],   # which fields to embed
         }
     )

     # Production: AsyncPostgresStore with embeddings
     # from langgraph.store.postgres.aio import AsyncPostgresStore
     # store = AsyncPostgresStore.from_conn_string(DB_URI, index={...})
     ```
124. Without embedding configuration, `asearch(query="what is the user's name?")` will NOT find semantically related memories — it only matches exact text; always configure embeddings for memory agents

## filter_messages — Selecting Message Types

125. Use `filter_messages` to select specific message types from the history — useful when you only want to pass human/AI turns to a summarization or classification node:
     ```python
     from langchain_core.messages import filter_messages, HumanMessage, AIMessage

     def summarize_node(state: MessagesState) -> dict:
         # Only pass human and AI turns, exclude tool messages
         conversation = filter_messages(
             state["messages"],
             include_types=[HumanMessage, AIMessage],
         )
         summary = summarizer.invoke(conversation)
         return {"summary": summary.content}
     ```
126. `filter_messages(messages, exclude_types=[ToolMessage])` is equivalent — use whichever is more readable for your case

## graph.abatch() — Parallel Graph Runs

127. Use `graph.abatch(inputs, configs)` to run multiple inputs through the graph in parallel — more efficient than sequential `ainvoke` calls:
     ```python
     inputs = [
         {"messages": [HumanMessage("Summarize doc 1")]},
         {"messages": [HumanMessage("Summarize doc 2")]},
         {"messages": [HumanMessage("Summarize doc 3")]},
     ]
     configs = [
         {"configurable": {"thread_id": f"batch-{i}"}}
         for i in range(len(inputs))
     ]
     results = await graph.abatch(inputs, configs)
     ```
128. Each input in `abatch()` must have its own unique `thread_id` config — sharing a thread_id across batch inputs causes checkpoint conflicts

## graph.as_tool() — Wrapping a Graph as a Tool

129. Use `graph.as_tool()` to expose a compiled graph as a LangChain tool so a parent agent can call it — the standard pattern for nested multi-agent architectures:
     ```python
     # Research subgraph
     research_graph = research_builder.compile()

     research_tool = research_graph.as_tool(
         name="research_agent",
         description="Research a topic thoroughly. Input: the research question as a string.",
         arg_types={"input": str},  # what the parent agent passes
     )

     # Parent orchestrator uses research_graph as just another tool
     orchestrator = create_react_agent(
         model,
         tools=[research_tool, write_tool, summarize_tool],
     )
     ```
130. `graph.as_tool()` is preferred over calling `graph.ainvoke()` manually inside a `@tool` function — it keeps tool metadata consistent and handles streaming correctly

## FastAPI + LangGraph Streaming Integration

131. Wire `graph.astream()` to a FastAPI SSE endpoint — the standard pattern for serving LangGraph agents over HTTP:
     ```python
     from fastapi import FastAPI
     from fastapi.responses import StreamingResponse
     import json

     app = FastAPI()

     @app.post("/chat")
     async def chat(request: ChatRequest):
         config = {
             "recursion_limit": 50,
             "configurable": {"thread_id": request.thread_id},
         }

         async def generate():
             async for chunk in graph.astream(
                 {"messages": [HumanMessage(request.message)]},
                 config,
                 stream_mode="messages",
                 version="v2",
             ):
                 if chunk["type"] == "messages":
                     msg_chunk, metadata = chunk["data"]
                     if msg_chunk.content and metadata["langgraph_node"] == "chatbot":
                         yield f"data: {json.dumps({'content': msg_chunk.content})}\n\n"
             yield "data: [DONE]\n\n"

         return StreamingResponse(generate(), media_type="text/event-stream")
     ```
132. Always set `recursion_limit` in the FastAPI handler config — unbounded agent loops will hold the HTTP connection open indefinitely

## SystemMessage Injection Pattern

133. Prepend a `SystemMessage` to messages at call time inside the node — do NOT store it in graph state (it would accumulate with every turn):
     ```python
     from langchain_core.messages import SystemMessage

     async def chatbot(state: MessagesState, config: RunnableConfig) -> dict:
         user_id = config["configurable"].get("user_id", "anonymous")
         system = SystemMessage(
             f"You are a helpful assistant. User ID: {user_id}. Today: {date.today()}."
         )
         # System message prepended at call time, NOT saved to state
         response = await model_with_tools.ainvoke([system] + state["messages"])
         return {"messages": response}
     ```
134. NEVER append `SystemMessage` to `state["messages"]` — it accumulates across turns and inflates the context; always inject dynamically in the node body

## store.aget() — Exact Key Retrieval

135. Use `store.aget(namespace, key)` for exact key lookup — faster than `asearch()` when you know the exact key:
     ```python
     async def load_profile(state: MessagesState, runtime: Runtime[AppContext]) -> dict:
         user_id = runtime.context.user_id

         # Exact lookup — returns Item or None
         item = await runtime.store.aget(("profiles", user_id), "main")
         if item:
             profile = item.value  # the dict you stored with aput()
         else:
             profile = {"name": "Unknown", "preferences": {}}

         return {"profile": profile}
     ```
136. Store API summary: `aput()` write, `aget()` exact read, `asearch()` semantic/keyword search, `adelete()` delete, `alist()` list namespace

## LCEL Chains as Nodes

137. Any LangChain `Runnable` (including LCEL `|` chains) can be used directly as a node — LangGraph calls `.invoke()` on it with the state dict:
     ```python
     from langchain_core.prompts import ChatPromptTemplate
     from langchain_core.output_parsers import StrOutputParser

     # LCEL chain
     summarize_chain = (
         ChatPromptTemplate.from_template("Summarize this in one sentence: {text}")
         | ChatAnthropic(model="claude-haiku-4-5-20251001")
         | StrOutputParser()
     )

     # Add directly as a node — LangGraph passes state dict as input
     # Chain receives {"text": "...", "messages": [...]} — use only what it needs
     builder.add_node("summarizer", summarize_chain)
     ```
138. LCEL chain nodes receive the FULL state dict as input — use `itemgetter` or a `RunnableLambda` wrapper if the chain only needs specific keys:
     ```python
     from operator import itemgetter
     from langchain_core.runnables import RunnableLambda

     # Extract only the "text" field before passing to the chain
     summarize_node = (
         RunnableLambda(lambda state: {"text": state["text"]})
         | summarize_chain
         | RunnableLambda(lambda result: {"summary": result})
     )
     builder.add_node("summarizer", summarize_node)
     ```

## Middleware — LangChain Callback System

139. LangGraph has no built-in middleware layer — use **LangChain callbacks** for cross-cutting concerns (logging, timing, cost tracking, error alerting):
     ```python
     from langchain_core.callbacks import AsyncCallbackHandler
     import time

     class AgentMetricsHandler(AsyncCallbackHandler):
         async def on_llm_start(self, serialized, messages, **kwargs):
             self._start = time.monotonic()
             logger.info(f"LLM call: {serialized.get('name')}")

         async def on_llm_end(self, response, **kwargs):
             duration = time.monotonic() - self._start
             tokens = response.llm_output.get("token_usage", {})
             logger.info(f"LLM done: {duration:.2f}s, tokens={tokens}")

         async def on_tool_start(self, serialized, input_str, **kwargs):
             logger.info(f"Tool: {serialized['name']}({input_str[:80]})")

         async def on_tool_error(self, error, **kwargs):
             logger.error(f"Tool error: {error}")
     ```
140. Attach callbacks per-run via config — they fire on every LLM call and tool call within that graph run:
     ```python
     config = {
         "callbacks": [AgentMetricsHandler(), LangSmithCallbackHandler()],
         "configurable": {"thread_id": "session-1"},
     }
     result = await graph.ainvoke(input, config)
     ```
141. Attach callbacks at compile time via `graph.with_config()` to apply them to ALL runs of that graph — useful for global observability:
     ```python
     instrumented_graph = graph.with_config({
         "callbacks": [AgentMetricsHandler()],
         "recursion_limit": 50,
     })
     # Now every invoke/astream on instrumented_graph uses these defaults
     result = await instrumented_graph.ainvoke(input, config)
     ```
142. Key callback events for agents: `on_llm_start`, `on_llm_end`, `on_llm_error`, `on_tool_start`, `on_tool_end`, `on_tool_error`, `on_chain_start`, `on_chain_end`
143. NEVER do blocking I/O in `BaseCallbackHandler` methods — use `AsyncCallbackHandler` for any async operations (DB writes, HTTP calls); sync handlers block the event loop

## What Makes LangGraph Exceptional (Design Principles)

144. LangGraph's core value over a plain `while True:` loop: **Durable** (survives process crashes), **Observable** (every step traced), **Controllable** (pause at any point), **Resumable** (any checkpoint → continue from there), **Debuggable** (time-travel replay), **Parallelizable** (supersteps) — these are not add-ons, they are the design
145. LangGraph is NOT an LLM framework — it is a **graph execution engine**. The LLM is just one kind of node. You can build workflows with zero LLM calls and LangGraph still provides all guarantees.
146. The checkpointer is what transforms an agent from a **disposable process** into a **persistent entity** — with a checkpointer, a thread is a long-lived identity that can be paused for days and resumed exactly where it left off
147. LangGraph enforces **explicit control flow** — unlike ReAct/AgentExecutor (black box), every routing decision is your code, every state transition is visible, every failure is recoverable. This is what "full control" means.

## Superstep State Isolation (Critical Correctness Rule)

148. **All parallel nodes in a superstep receive the state BEFORE that superstep started** — parallel nodes cannot see each other's writes until the NEXT step:
     ```
     State before superstep: {count: 0}
     node_a runs → returns {count: 1}   ←── both nodes started from count=0
     node_b runs → returns {count: 1}   ←── NOT count=2, they don't see each other
     After superstep: reducers merge → {count: 2}  (if using operator.add)
     ```
149. This isolation is intentional — it makes parallel execution deterministic and prevents race conditions; design your state reducers to merge parallel writes correctly (operator.add, not replace)

## Streaming + HITL Combined

150. When `interrupt()` fires during a streaming run, the stream emits the interrupt event and pauses — clients must detect this and surface it to the user:
     ```python
     async for chunk in graph.astream(input, config, stream_mode="updates"):
         if "__interrupt__" in chunk:
             # Graph is paused — surface interrupt to user
             interrupt_value = chunk["__interrupt__"][0].value
             user_decision = await get_human_input(interrupt_value)
             # Resume — pass the decision back
             async for resume_chunk in graph.astream(
                 Command(resume=user_decision),
                 config,
                 stream_mode="updates",
             ):
                 yield resume_chunk
             break  # break the outer loop — resume_chunk loop handles rest
         else:
             yield chunk
     ```
151. NEVER discard `__interrupt__` chunks in streaming — if you filter them out, the agent appears hung to the client and the interrupt is never resolved

## Durable Long-Running Agent Pattern

152. Design long-running agents (minutes to hours) as durable runs: start the run, store the `thread_id`, return to the client immediately, let the agent run in the background, poll or stream results later:
     ```python
     # Start — returns immediately
     @app.post("/agents/start")
     async def start_agent(request: AgentRequest):
         thread_id = str(uuid4())
         asyncio.create_task(
             graph.ainvoke(
                 {"task": request.task},
                 {"configurable": {"thread_id": thread_id}, "recursion_limit": 200}
             )
         )
         return {"thread_id": thread_id, "status": "running"}

     # Poll state
     @app.get("/agents/{thread_id}/state")
     async def get_agent_state(thread_id: str):
         config = {"configurable": {"thread_id": thread_id}}
         snapshot = graph.get_state(config)
         return {"values": snapshot.values, "next": snapshot.next}
     ```
153. Durable agents require `AsyncPostgresSaver` — `InMemorySaver` is lost if the background task's process restarts
154. Set `recursion_limit` high for long-running agents (100-500) — the default 25 is for short interactive agents

## Reflexion — Self-Correction Loop

155. The reflexion pattern: agent generates output → evaluator scores it → if quality below threshold, regenerate with feedback — implement as a conditional edge:
     ```python
     class ReflexionState(TypedDict):
         task: str
         draft: str
         critique: str
         iteration: int

     def evaluate(state: ReflexionState) -> dict:
         score = evaluator_chain.invoke({"draft": state["draft"]})
         return {"critique": score.critique, "iteration": state["iteration"] + 1}

     def should_revise(state: ReflexionState) -> Literal["revise", "__end__"]:
         if state["iteration"] >= 3:       # max iterations guard — REQUIRED
             return END
         if "APPROVED" in state["critique"]:
             return END
         return "revise"

     builder.add_conditional_edges("evaluate", should_revise)
     ```
156. ALWAYS add a max iterations guard to reflexion loops — an evaluator that never gives APPROVED will loop forever; `iteration >= N` → force END

## Routing Based on Tool Output Content

157. After `ToolNode` runs, inspect `ToolMessage` content to route to different nodes — not just whether a tool was called, but what it returned:
     ```python
     def route_on_tool_result(state: MessagesState) -> Literal["handle_error", "handle_success", "agent"]:
         last = state["messages"][-1]
         if not hasattr(last, "tool_call_id"):
             return "agent"  # not a tool result — back to agent
         if last.status == "error":
             return "handle_error"
         if "not found" in last.content.lower():
             return "handle_not_found"
         return "agent"  # success — let agent continue

     builder.add_conditional_edges("tools", route_on_tool_result)
     ```
158. `ToolMessage.status` is `"success"` or `"error"` — `ToolNode` sets this automatically based on whether the tool raised an exception

## Determinism and Predictability

159. Use `temperature=0` on all LLM calls inside routing/classification nodes — routing decisions must be deterministic; non-zero temperature introduces random branching
160. Use structured output (`model.with_structured_output(Schema)`) for all routing nodes — free text routing ("go to node_a") fails on slight phrasing variations; typed output (`Literal["node_a", "node_b"]`) never fails
161. Document every conditional edge with a comment explaining the exact routing logic — conditional edges are the most common source of agent behavior bugs; make intent explicit
