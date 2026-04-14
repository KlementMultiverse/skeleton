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

## Anthropic SDK Integration

54. The `messages` state key IS the conversation history — pass it directly to `client.messages.create(messages=[...])`
55. `AIMessage` from LangChain maps to `{"role": "assistant", "content": "..."}` in Anthropic SDK
56. `ToolMessage` maps to a `tool_result` content block — `tool_use_id` must match
57. Initialize `AsyncAnthropic` ONCE in lifespan, inject into graph via `Runtime[Context]` or app state — see `anthropic-patterns.md` rule 1
