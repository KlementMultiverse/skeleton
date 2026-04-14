# L14: LLM Observability — Langfuse, Tracing, Evals, Cost Control

## The one thing to remember

An LLM call is a black box. Observability is the window you cut into that box. Without it you are
flying blind: you do not know which prompt version is failing in production, why costs spiked at
2 AM, or whether your RAG pipeline's answers are actually grounded in the retrieved context.

---

## Concept 1 — Why LLM Observability Is Different From Regular APM

**Analogy:** Traditional APM (Application Performance Monitoring) is like monitoring a vending
machine — you track inputs, outputs, latency, and errors. LLM observability is like monitoring a
chef: the same ingredients can produce wildly different dishes depending on the recipe version,
the model's mood, and whether the prep cook retrieved the right context. You need to inspect the
*reasoning*, not just the plumbing.

The key differences from regular distributed tracing:

| Concern | Regular APM | LLM Observability |
|---|---|---|
| Input payload | Request body size | Prompt text + token count |
| Output payload | Response bytes | Completion text + stop reason |
| Cost unit | CPU/memory | Input tokens × price + output tokens × price |
| Quality signal | HTTP status code | Faithfulness, relevancy, hallucination rate |
| Debugging unit | Stack trace | Prompt version + model + context window contents |
| Regression signal | Latency p99 | Eval score drop on golden dataset |

You need both layers. Regular APM for infrastructure health. LLM observability for *quality*.

---

## Concept 2 — The Langfuse Data Model

**Analogy:** Think of a Trace as a court case file. A case has a docket number (trace_id), a
plaintiff (user_id), a case type (name), and metadata. Inside that file are individual hearings
(Spans). Each hearing might involve an expert witness (a Generation — the actual LLM call).
After the case closes, a judge reviews it and writes a verdict (Score). The file itself does not
do anything — it just organizes everything that happened.

```
Trace
├── Span  (e.g., "retrieve-context")
│   └── Generation  (e.g., "embed-query")
├── Span  (e.g., "rerank")
└── Generation  (e.g., "final-answer")
    └── Score  (e.g., faithfulness = 0.91)
```

### Trace

The root entity. Represents one logical unit of work — typically one user request.

Required fields:
- `id` — your own request_id, propagated from HTTP middleware
- `name` — human-readable label, e.g. `"chat-completion"`, `"rag-query"`
- `user_id` — always bind after auth resolves; enables per-user cost attribution
- `session_id` — groups multiple traces for the same conversation
- `metadata` — arbitrary dict; include `prompt_version`, `model`, `env`

```python
from langfuse import Langfuse

langfuse = Langfuse()

trace = langfuse.trace(
    id=request_id,                    # your HTTP request ID
    name="rag-query",
    user_id=str(current_user.id),
    session_id=session_id,
    metadata={
        "prompt_version": 3,
        "model": "claude-3-5-sonnet-20241022",
        "env": "production",
        "tenant_id": str(tenant_id),
    },
    tags=["rag", "production"],
)
```

### Span

A timed unit of work within a trace. Use for everything that is *not* an LLM call: retrieval,
reranking, tool execution, DB queries. Spans can be nested.

```python
retrieval_span = trace.span(
    name="retrieve-context",
    input={"query": user_query, "top_k": 5},
)
# ... do the retrieval ...
retrieval_span.end(
    output={"chunks": chunks, "chunk_count": len(chunks)},
    metadata={"retriever": "pgvector", "similarity_threshold": 0.75},
)
```

### Generation

A specialised Span specifically for LLM API calls. Captures token usage, model name, cost,
latency, and stop reason. This is what Langfuse uses to compute per-model cost dashboards.

Required fields per generation:
- `model` — exact model string, e.g. `"claude-3-5-sonnet-20241022"`
- `input` — the messages array (prompt)
- `output` — the completion text
- `usage.input` — input token count
- `usage.output` — output token count
- `metadata.latency_ms` — wall-clock time for the LLM call
- `metadata.stop_reason` — e.g. `"end_turn"`, `"max_tokens"`, `"tool_use"`
- association with `trace_id` and optionally `parent_observation_id`

```python
import time

generation = trace.generation(
    name="final-answer",
    model="claude-3-5-sonnet-20241022",
    model_parameters={"max_tokens": 1024, "temperature": 0.3},
    input=messages,
)

t0 = time.monotonic()
response = anthropic_client.messages.create(
    model="claude-3-5-sonnet-20241022",
    messages=messages,
    max_tokens=1024,
)
latency_ms = int((time.monotonic() - t0) * 1000)

generation.end(
    output=response.content[0].text,
    usage={
        "input": response.usage.input_tokens,
        "output": response.usage.output_tokens,
        "unit": "TOKENS",
    },
    metadata={
        "latency_ms": latency_ms,
        "stop_reason": response.stop_reason,
    },
)
```

### Score

An evaluation metric attached to a trace or a specific observation. Scores are the bridge
between raw traces and quality dashboards. They can be created by:
1. A human annotator in the Langfuse UI
2. An automated LLM-as-judge pipeline
3. Your application code (e.g., user thumbs up/down)
4. RAGAS or other eval frameworks

```python
# Attach score to a trace
langfuse.score(
    trace_id=trace.id,
    name="faithfulness",
    value=0.91,                        # 0.0 to 1.0
    comment="All claims verified against retrieved context",
)

# Attach score to a specific observation (e.g., the generation span)
langfuse.score(
    trace_id=trace.id,
    observation_id=generation.id,
    name="answer_relevancy",
    value=0.87,
)
```

---

## Concept 3 — The `@observe` Decorator

**Analogy:** The `@observe` decorator is like CCTV cameras automatically installed in every room
of your function call graph. You do not have to wire up start/end timing manually — it wraps
each decorated function in a span, records input/output, and chains parent/child relationships
automatically.

```python
from langfuse.decorators import langfuse_context, observe

@observe()                             # creates a Span automatically
def retrieve_context(query: str, top_k: int = 5) -> list[str]:
    chunks = vector_db.search(query, top_k=top_k)
    return chunks

@observe()                             # parent span
def answer_question(user_query: str, user_id: str) -> str:
    # Bind user to the trace — call this once per request, after auth
    langfuse_context.update_current_trace(
        user_id=user_id,
        session_id=get_session_id(user_id),
        metadata={"prompt_version": 3},
    )

    chunks = retrieve_context(user_query)        # child span auto-nested

    # Update current observation (the answer_question span itself)
    langfuse_context.update_current_observation(
        input={"query": user_query, "chunk_count": len(chunks)},
        metadata={"retriever": "pgvector"},
    )

    response = call_llm(user_query, chunks)      # another child span
    return response
```

Key `langfuse_context` methods:

| Method | What it does |
|---|---|
| `update_current_trace(...)` | Set trace-level fields: user_id, session_id, name, tags |
| `update_current_observation(...)` | Set span-level fields: input, output, metadata |
| `get_current_trace_id()` | Read the trace ID (needed to attach scores externally) |
| `get_current_observation_id()` | Read the span ID (for observation-level scores) |

---

## Concept 4 — Prompt Versioning

**Analogy:** Prompt versioning is like Git for your prompts. You commit a new version, you can
roll back, and your traces tell you which commit (version) produced each response. Without this,
when a prompt change degrades quality you cannot tell which deploy introduced the regression.

```python
from langfuse import Langfuse

langfuse = Langfuse()

# Fetch production-labelled prompt (default)
prompt = langfuse.get_prompt("rag-system-prompt")

# Fetch a specific version by number — for canary / A/B testing
prompt_v3 = langfuse.get_prompt("rag-system-prompt", version=3)

# Fetch a label — use labels for staging / shadow traffic
prompt_staging = langfuse.get_prompt("rag-system-prompt", label="staging")

# Compile: replaces {{variable}} placeholders
compiled = prompt_v3.compile(
    context="\n".join(chunks),
    user_query=user_query,
)

# Link the generation to the prompt version — shows in Langfuse UI
generation = trace.generation(
    name="answer",
    model="claude-3-5-sonnet-20241022",
    input=compiled,
    prompt=prompt_v3,                  # links trace to prompt version
)
```

Prompt versioning workflow:
1. Create prompt in Langfuse UI or via SDK
2. Label one version `production` — all clients fetch it by default
3. Deploy new version with label `staging` first, run evals
4. Promote to `production` label when eval scores pass threshold
5. Traces automatically record which version produced each response

---

## Concept 5 — Datasets and Golden Evals

**Analogy:** A golden dataset is like a driving test exam. The questions are fixed, the correct
answers are known, and you run every candidate (every prompt version, every model change) through
the same exam. A candidate passes when their score exceeds the threshold. A regression is
detected when a previously-passing candidate now fails.

```python
from langfuse import Langfuse
import pytest

langfuse = Langfuse()

def test_rag_quality_regression():
    """CI gate: eval scores must not drop below baseline."""
    dataset = langfuse.get_dataset("rag-golden-v2")

    run_name = f"ci-{os.environ['GIT_SHA'][:8]}"
    results = []

    for item in dataset.items:
        # Run your application against the golden input
        answer = answer_question(
            user_query=item.input["question"],
            user_id="ci-test-user",
        )

        # Link execution trace to the dataset item
        trace_id = langfuse_context.get_current_trace_id()
        item.link(trace_id, run_name, metadata={"model": "claude-3-5-sonnet-20241022"})

        # Score against expected output
        score = compute_faithfulness(answer, item.expected_output)
        langfuse.score(trace_id=trace_id, name="faithfulness", value=score)
        results.append(score)

    avg_faithfulness = sum(results) / len(results)

    # Hard regression gate
    assert avg_faithfulness >= 0.85, (
        f"Faithfulness regression: {avg_faithfulness:.3f} < 0.85 baseline"
    )

    langfuse.flush()
```

Creating a golden dataset from production traces:

```python
# Promote high-quality production traces to golden dataset
dataset = langfuse.create_dataset(name="rag-golden-v2")

for trace in high_quality_traces:            # curated from production
    langfuse.create_dataset_item(
        dataset_name="rag-golden-v2",
        input={"question": trace.input["user_query"]},
        expected_output=trace.output,        # human-verified answer
        metadata={"source_trace_id": trace.id},
    )
```

---

## Concept 6 — Flushing

**Analogy:** Langfuse batches telemetry events in memory and sends them asynchronously — like a
postal worker who collects all mail for an hour before driving to the post office. If the
process exits before the trip happens, your mail is lost. `flush()` forces the trip immediately.

```python
# ALWAYS call before process exit
langfuse.flush()

# In FastAPI lifespan — ensures no data loss on shutdown
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    langfuse.flush()                   # drain the buffer on shutdown

app = FastAPI(lifespan=lifespan)

# In background tasks — flush after the task completes
from fastapi import BackgroundTasks

async def evaluate_in_background(trace_id: str):
    score = await run_llm_judge(trace_id)
    langfuse.score(trace_id=trace_id, name="quality", value=score)
    langfuse.flush()                   # flush at end of background task

@app.post("/chat")
async def chat(background_tasks: BackgroundTasks, ...):
    response = await answer_question(...)
    trace_id = langfuse_context.get_current_trace_id()
    background_tasks.add_task(evaluate_in_background, trace_id)
    return response
```

Rule: every code path that creates Langfuse observations must have a `flush()` before the
process or task ends. Missing a flush in a serverless function means every observation is lost.

---

## Concept 7 — LangSmith

**Analogy:** If Langfuse is a standalone observability platform you host yourself (or use as
SaaS), LangSmith is the observability platform *built into the LangChain ecosystem*. It is like
the difference between a third-party IDE plugin and the IDE's own built-in debugger. LangSmith
is tighter for LangChain/LangGraph; Langfuse is more flexible for any LLM stack.

### Automatic tracing with LangChain

```python
import os

# Set these in your environment — automatic tracing, zero code changes
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls_..."
os.environ["LANGCHAIN_PROJECT"] = "my-rag-app"

# Every LangChain / LangGraph invocation is now traced automatically
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
response = llm.invoke([HumanMessage(content="What is RAG?")])
# Trace appears in LangSmith dashboard — no other code needed
```

### `@traceable` for non-LangChain code

```python
from langsmith import traceable
from langsmith.wrappers import wrap_openai
import openai

# Wrap third-party clients to capture token usage automatically
client = wrap_openai(openai.Client())

@traceable(name="retrieve-context", run_type="retriever")
def retrieve(query: str) -> list[str]:
    return vector_db.search(query)

@traceable(name="rag-pipeline", run_type="chain", tags=["production"])
def rag_pipeline(user_query: str, user_id: str) -> str:
    chunks = retrieve(user_query)          # child span, auto-nested
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": build_system_prompt(chunks)},
            {"role": "user", "content": user_query},
        ],
    )
    return response.choices[0].message.content
```

### LangSmith datasets and evaluators

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

def my_evaluator(run, example) -> dict:
    """Custom evaluator: returns score between 0 and 1."""
    prediction = run.outputs.get("answer", "")
    reference = example.outputs.get("answer", "")
    score = 1.0 if reference.lower() in prediction.lower() else 0.0
    return {"key": "answer_match", "score": score}

results = evaluate(
    rag_pipeline,
    data="rag-golden-dataset",
    evaluators=[my_evaluator],
    experiment_prefix=f"ci-{git_sha[:8]}",
    metadata={"model": "claude-3-5-sonnet-20241022", "prompt_version": 3},
)
```

### LangSmith vs Langfuse — when to choose which

| Criterion | Choose LangSmith | Choose Langfuse |
|---|---|---|
| Primary framework | LangChain / LangGraph | Any (Anthropic, OpenAI direct, custom) |
| Prompt management | Not primary feature | First-class feature with versioning |
| Self-hosting | Not supported | Full self-hosted Docker option |
| Dataset/eval UI | Excellent | Excellent |
| Cost | Per-trace pricing | Open source, free self-hosted tier |
| Auto-instrumentation | LangChain ecosystem only | Integrations for OpenAI, Anthropic, LiteLLM |
| Data residency | US/EU on LangChain cloud | Full control when self-hosted |

Rule: if your stack is LangChain-heavy, LangSmith's zero-config tracing (`LANGCHAIN_TRACING_V2`)
is a strong advantage. If you run raw Anthropic/OpenAI calls or want self-hosting, Langfuse wins.
Never run both simultaneously on the same call — double-tracing wastes bandwidth and money.

---

## Concept 8 — Request ID Propagation

**Analogy:** Request ID propagation is like a shipping label on a package. The label is printed at
the post office (HTTP gateway), attached at the front door, and scanned at every sorting facility
(middleware, service layer, LLM call). Without the label, when a package is lost you cannot trace
which route it took. Without a request ID in every LLM trace, you cannot correlate a user
complaint to a specific LLM call.

```python
import uuid
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

app = FastAPI()
app.add_middleware(RequestIDMiddleware)
```

Propagate the request ID into every LLM trace:

```python
from langfuse.decorators import langfuse_context, observe

@observe()
async def answer_question(
    user_query: str,
    user_id: str,
    request_id: str,               # injected from request.state
    tenant_id: str,
) -> str:
    # Bind the HTTP request ID as the Langfuse trace ID
    langfuse_context.update_current_trace(
        id=request_id,             # makes traces findable by request ID
        user_id=user_id,
        metadata={
            "tenant_id": tenant_id,
            "request_id": request_id,
        },
    )
    # ... rest of logic
```

---

## Concept 9 — Cost Attribution

**Analogy:** Cost attribution is like expense reports in a company. Without them, the accounting
department just sees a large "cloud services" line item. With them, each team, user, and project
sees exactly what they spent. You cannot optimise what you cannot see.

Token pricing (as of 2025, prices change — always verify):
- Claude 3.5 Sonnet: ~$3.00 / 1M input tokens, ~$15.00 / 1M output tokens
- GPT-4o: ~$2.50 / 1M input, ~$10.00 / 1M output
- GPT-4o-mini: ~$0.15 / 1M input, ~$0.60 / 1M output

Cost calculation pattern:

```python
from dataclasses import dataclass
from decimal import Decimal

MODEL_PRICES = {
    "claude-3-5-sonnet-20241022": {
        "input": Decimal("0.000003"),    # $ per token
        "output": Decimal("0.000015"),
    },
    "gpt-4o-mini": {
        "input": Decimal("0.00000015"),
        "output": Decimal("0.0000006"),
    },
}

def compute_cost(model: str, input_tokens: int, output_tokens: int) -> Decimal:
    prices = MODEL_PRICES[model]
    return (
        Decimal(input_tokens) * prices["input"]
        + Decimal(output_tokens) * prices["output"]
    )

# In your generation logging:
cost = compute_cost(model, response.usage.input_tokens, response.usage.output_tokens)

generation.end(
    output=response.content[0].text,
    usage={
        "input": response.usage.input_tokens,
        "output": response.usage.output_tokens,
        "input_cost": float(cost * Decimal(input_tokens) / Decimal(input_tokens + output_tokens)),
        "output_cost": float(cost * Decimal(output_tokens) / Decimal(input_tokens + output_tokens)),
        "unit": "TOKENS",
    },
    metadata={"tenant_id": tenant_id, "user_id": user_id},
)
```

Langfuse aggregates cost by user_id, session_id, model, and prompt_version automatically once
`usage.input_cost` and `usage.output_cost` (or just `usage.input` and `usage.output` with a
configured model price) are provided.

---

## Concept 10 — Multi-Step Agent Traces

**Analogy:** A multi-step agent trace is like a project management timeline. The project
(trace) has a start date and end date. Inside are phases (parent spans), and each phase has
tasks (child spans / generations). You can see the critical path — which step took the longest —
and where quality degraded.

```python
from langfuse.decorators import observe, langfuse_context

@observe(name="tool-call")
def execute_tool(tool_name: str, tool_input: dict) -> dict:
    langfuse_context.update_current_observation(
        input={"tool": tool_name, "args": tool_input},
    )
    result = TOOLS[tool_name](**tool_input)
    langfuse_context.update_current_observation(output=result)
    return result

@observe(name="agent-step")
def run_agent_step(messages: list, step_num: int) -> tuple[str, list]:
    langfuse_context.update_current_observation(
        metadata={"step": step_num},
    )
    response = anthropic_client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=messages,
        tools=TOOL_SCHEMAS,
        max_tokens=4096,
    )

    if response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)   # child span
                tool_results.append(result)
        return "continue", tool_results
    return "done", [response.content[0].text]

@observe(name="agent-run")                # root trace
def run_agent(user_task: str, user_id: str) -> str:
    langfuse_context.update_current_trace(
        user_id=user_id,
        name=f"agent:{user_task[:50]}",
    )
    messages = [{"role": "user", "content": user_task}]
    step = 0
    while True:
        step += 1
        status, results = run_agent_step(messages, step_num=step)
        if status == "done":
            return results[0]
        # append tool results to messages and continue
        messages = append_tool_results(messages, results)
```

The resulting trace tree in Langfuse:
```
agent-run (trace)
├── agent-step (step=1)
│   ├── execute_tool (search_web)
│   └── execute_tool (read_file)
└── agent-step (step=2)
    └── [generation — final answer]
```

---

## Concept 11 — Evaluation: LLM-as-Judge

**Analogy:** LLM-as-judge is like hiring a senior engineer to review junior engineers' code. The
senior engineer (judge LLM) does not write the code — they review it and rate it on a rubric.
G-Eval formalises this rubric: define dimensions (coherence, correctness, fluency), have the
judge score each one, and average the scores.

```python
import anthropic
import json
from langfuse import Langfuse

langfuse = Langfuse()
judge_client = anthropic.Anthropic()

JUDGE_PROMPT = """
You are evaluating an AI assistant's answer to a question.

Question: {question}
Retrieved context: {context}
Answer to evaluate: {answer}

Rate the answer on these dimensions (0.0 to 1.0 each):
1. faithfulness — are all claims in the answer supported by the context?
2. relevancy — does the answer address the question?
3. completeness — does the answer cover the key points from context?

Return JSON: {{"faithfulness": float, "relevancy": float, "completeness": float, "reasoning": str}}
Only return valid JSON, nothing else.
"""

def llm_judge(
    question: str,
    context: str,
    answer: str,
    trace_id: str,
) -> dict[str, float]:
    response = judge_client.messages.create(
        model="claude-3-5-haiku-20241022",     # cheap model for judging
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": JUDGE_PROMPT.format(
                question=question,
                context=context,
                answer=answer,
            ),
        }],
    )
    try:
        scores = json.loads(response.content[0].text)
    except (json.JSONDecodeError, KeyError):
        logger.error("judge_parse_failed trace_id=%s", trace_id)
        return {}

    for metric, value in scores.items():
        if metric == "reasoning":
            continue
        langfuse.score(
            trace_id=trace_id,
            name=metric,
            value=float(value),
            comment=scores.get("reasoning", ""),
        )

    return {k: v for k, v in scores.items() if k != "reasoning"}
```

---

## Concept 12 — RAGAS Evaluation

**Analogy:** RAGAS is like a four-panel inspection for a car: you check the engine
(faithfulness — does the answer match the context?), the GPS (answer relevancy — does the answer
address the question?), the map (context precision — is the retrieved context relevant and
ranked well?), and the fuel gauge (context recall — did you retrieve all the information needed?).

The four core RAGAS metrics:

| Metric | Measures | Formula sketch |
|---|---|---|
| Faithfulness | Are all answer claims supported by context? | `supported_claims / total_claims` |
| Answer Relevancy | Does the answer address the question? | Cosine similarity of reverse-engineered questions |
| Context Precision | Are relevant chunks ranked at the top? | Weighted precision at rank k |
| Context Recall | Is all needed information present in context? | Ground truth statements covered by context |

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

def run_ragas_eval(test_cases: list[dict]) -> dict[str, float]:
    """
    test_cases: list of {question, answer, contexts, ground_truth}
    Returns average scores across all cases.
    """
    dataset = Dataset.from_list(test_cases)

    result = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    )

    scores = {
        "faithfulness": result["faithfulness"],
        "answer_relevancy": result["answer_relevancy"],
        "context_precision": result["context_precision"],
        "context_recall": result["context_recall"],
    }

    # Push scores to Langfuse for trend tracking
    for trace_id, case, score_row in zip(trace_ids, test_cases, result.to_pandas().itertuples()):
        for metric in ["faithfulness", "answer_relevancy", "context_precision", "context_recall"]:
            langfuse.score(
                trace_id=trace_id,
                name=metric,
                value=getattr(score_row, metric),
            )

    return scores
```

Quality thresholds for production RAG:
- Faithfulness > 0.85 — below this, the model is hallucinating
- Answer relevancy > 0.80 — below this, answers are off-topic
- Context precision > 0.75 — below this, retrieval is returning noise
- Context recall > 0.80 — below this, retrieval is missing key chunks

---

## Concept 13 — Alerting Thresholds

**Analogy:** Alerting thresholds are like circuit breakers in your house. A circuit breaker does
not alert you when you turn on the toaster. It trips only when the current exceeds a safe level.
LLM alerting should be the same: silent for normal operation, loud for genuine anomalies.

```python
from dataclasses import dataclass
from enum import Enum

class AlertSeverity(Enum):
    WARNING = "warning"
    CRITICAL = "critical"

@dataclass(frozen=True)
class AlertThreshold:
    metric: str
    warning: float
    critical: float
    window_minutes: int
    severity_at_breach: AlertSeverity

THRESHOLDS = [
    AlertThreshold("error_rate_pct",      warning=0.5,  critical=1.0,  window_minutes=5,  severity_at_breach=AlertSeverity.CRITICAL),
    AlertThreshold("p99_latency_ms",      warning=20000, critical=30000, window_minutes=5, severity_at_breach=AlertSeverity.WARNING),
    AlertThreshold("cost_per_request_usd", warning=0.05, critical=0.10, window_minutes=60, severity_at_breach=AlertSeverity.WARNING),
    AlertThreshold("context_utilization",  warning=0.80, critical=0.90, window_minutes=60, severity_at_breach=AlertSeverity.CRITICAL),
    AlertThreshold("token_spike_ratio",    warning=2.0,  critical=5.0,  window_minutes=5,  severity_at_breach=AlertSeverity.CRITICAL),
]
```

Key thresholds:
- **Error rate > 1% over 5 minutes** — critical. At 1% of a 1000 RPM service, 10 users per
  minute get errors. Page someone.
- **P99 latency > 30s** — warning → critical. LLM calls are slow; 30s is the UX abandonment
  threshold for most applications.
- **Cost per request > $0.10** — warning. A $0.10 request on a 100K RPD service costs $10K/day.
- **Context utilization > 90%** — critical. When you are filling 90%+ of the context window,
  you are truncating input silently. Quality degrades unpredictably.
- **Token count anomaly (5x spike)** — critical. Sudden token spikes are the #1 signal of
  prompt injection attacks — an attacker submitted a massive payload designed to overwhelm the
  context window or exfiltrate data via the model.

---

## Concept 14 — Cost Control

**Analogy:** Cost control for LLMs is like managing a buffet restaurant. You can let every guest
take as much as they want and run out of food (and money) by noon. Or you can set portion limits,
route hungry guests to cheaper dishes, and keep a headcount (token budget) per table.

### 1. Count tokens before the call

```python
import anthropic

client = anthropic.Anthropic()

def count_tokens_before_call(messages: list, model: str) -> int:
    """Count tokens without making the actual LLM call."""
    response = client.messages.count_tokens(
        model=model,
        messages=messages,
    )
    return response.input_tokens

MAX_INPUT_TOKENS = 80_000                   # hard limit per request

def safe_llm_call(messages: list, user_id: str) -> str:
    token_count = count_tokens_before_call(messages, "claude-3-5-sonnet-20241022")

    if token_count > MAX_INPUT_TOKENS:
        logger.warning(
            "token_limit_exceeded user_id=%s tokens=%d limit=%d",
            user_id, token_count, MAX_INPUT_TOKENS,
        )
        raise TokenLimitExceededError(f"Request too large: {token_count} tokens")

    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        messages=messages,
        max_tokens=1024,                    # ALWAYS set max_tokens
    )
    return response.content[0].text
```

### 2. Per-user monthly budget with Redis

```python
import redis.asyncio as redis
from decimal import Decimal

MONTHLY_BUDGET_USD = Decimal("10.00")      # per user per month

async def check_and_deduct_budget(
    redis_client: redis.Redis,
    user_id: str,
    cost_usd: Decimal,
) -> None:
    import calendar, datetime
    now = datetime.datetime.utcnow()
    month_key = f"llm_budget:{user_id}:{now.year}:{now.month}"

    # Atomic increment with expiry
    pipe = redis_client.pipeline()
    pipe.incrbyfloat(month_key, float(cost_usd))
    pipe.expire(month_key, calendar.monthrange(now.year, now.month)[1] * 86400)
    results = await pipe.execute()

    total_spent = Decimal(str(results[0]))
    if total_spent > MONTHLY_BUDGET_USD:
        raise BudgetExceededError(
            f"user_id={user_id} spent ${total_spent:.4f} of ${MONTHLY_BUDGET_USD} budget"
        )
```

### 3. Model routing: cheap for simple, expensive for complex

```python
from enum import Enum

class TaskComplexity(Enum):
    SIMPLE = "simple"       # classification, extraction, yes/no
    MODERATE = "moderate"   # summarisation, Q&A with short context
    COMPLEX = "complex"     # reasoning, long context, multi-step

MODEL_ROUTING = {
    TaskComplexity.SIMPLE:   "claude-3-5-haiku-20241022",
    TaskComplexity.MODERATE: "claude-3-5-sonnet-20241022",
    TaskComplexity.COMPLEX:  "claude-opus-4-5",
}

def route_model(task_complexity: TaskComplexity) -> str:
    return MODEL_ROUTING[task_complexity]

def classify_task_complexity(user_query: str, context_length: int) -> TaskComplexity:
    """Simple heuristic — replace with a cheap classifier in production."""
    if context_length > 50_000:
        return TaskComplexity.COMPLEX
    if len(user_query.split()) < 20 and "yes or no" in user_query.lower():
        return TaskComplexity.SIMPLE
    return TaskComplexity.MODERATE
```

### 4. Prompt compression

```python
def compress_chat_history(
    messages: list[dict],
    max_history_tokens: int = 4_000,
    summary_model: str = "claude-3-5-haiku-20241022",
) -> list[dict]:
    """Summarise older messages to stay within token budget."""
    if len(messages) <= 4:
        return messages

    older_messages = messages[:-4]          # keep last 4 messages verbatim
    recent_messages = messages[-4:]

    summary_response = anthropic_client.messages.create(
        model=summary_model,
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": (
                f"Summarise this conversation in 2-3 sentences, "
                f"preserving key facts and decisions:\n\n"
                + "\n".join(f"{m['role']}: {m['content']}" for m in older_messages)
            ),
        }],
    )
    summary = summary_response.content[0].text

    return [
        {"role": "system", "content": f"[Conversation summary: {summary}]"},
        *recent_messages,
    ]
```

---

## Concept 15 — OpenTelemetry Integration

**Analogy:** OpenTelemetry (OTel) is like a universal power adapter. Your LLM observability
data can plug into any backend — Jaeger, Grafana Tempo, Honeycomb, Datadog — without changing
instrumentation code. The gen_ai semantic conventions are the agreed-upon pin layout for that
adapter: every vendor that follows them can read any instrumented LLM application.

### Auto-instrumentation

```python
# Install: pip install opentelemetry-sdk opentelemetry-exporter-otlp
#          opentelemetry-instrumentation-anthropic

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.anthropic import AnthropicInstrumentor

def setup_otel(service_name: str, otlp_endpoint: str) -> None:
    provider = TracerProvider()
    exporter = OTLPSpanExporter(endpoint=otlp_endpoint)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    # Auto-instruments all anthropic.Anthropic() calls
    AnthropicInstrumentor().instrument()

setup_otel("rag-service", "http://tempo:4317")
# All Anthropic calls now emit spans with gen_ai.* attributes
```

### GenAI semantic convention span attributes

These are the standard attribute names — any OTel-compatible backend will understand them:

| Attribute | Example value | Description |
|---|---|---|
| `gen_ai.system` | `"anthropic"` | Provider name |
| `gen_ai.request.model` | `"claude-3-5-sonnet-20241022"` | Requested model |
| `gen_ai.response.model` | `"claude-3-5-sonnet-20241022"` | Model used in response |
| `gen_ai.usage.input_tokens` | `1523` | Input token count |
| `gen_ai.usage.output_tokens` | `312` | Output token count |
| `gen_ai.request.max_tokens` | `1024` | Max tokens requested |
| `gen_ai.response.finish_reasons` | `["end_turn"]` | Stop reasons |
| `gen_ai.operation.name` | `"chat"` | Operation type |

### Manual span enrichment

```python
tracer = trace.get_tracer("rag-service")

with tracer.start_as_current_span("rag-pipeline") as span:
    span.set_attribute("user.id", user_id)
    span.set_attribute("tenant.id", tenant_id)
    span.set_attribute("rag.prompt_version", prompt_version)
    span.set_attribute("rag.retriever", "pgvector")
    span.set_attribute("rag.chunk_count", len(chunks))

    answer = call_llm(messages)
    span.set_attribute("rag.answer_length", len(answer))
```

### Sampling strategy for high-traffic APIs

```python
from opentelemetry.sdk.trace.sampling import (
    ParentBased,
    TraceIdRatioBased,
)

# Sample 10% of traces in production — enough for quality monitoring
# without overwhelming your tracing backend at 1000 RPS
sampler = ParentBased(
    root=TraceIdRatioBased(0.10),      # 10% of new traces
)
# Always sample error traces — override in your span processor
# Always sample traces where quality score < 0.7 — add custom sampler
```

OTLP targets:
- **Grafana Tempo + Langfuse**: Tempo for infrastructure traces, Langfuse for LLM-specific
  quality metrics. Two backends, two concerns, no overlap.
- **Honeycomb**: excellent for high-cardinality analysis of LLM traces (grouping by prompt
  version, model, user cohort simultaneously).
- **Jaeger**: good for self-hosted, basic distributed trace visualisation.

---

## Concept 16 — CI Pipeline Integration

**Analogy:** Running evals in CI is like having automated QA testers who run through the same
checklist after every code change. They do not replace human judgment for new features, but they
catch regressions before they reach production.

```yaml
# .github/workflows/eval.yml
name: LLM Eval Gate

on:
  pull_request:
    paths: ["prompts/**", "app/llm/**"]  # only run when LLM code changes

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }

      - name: Install deps
        run: pip install -e ".[test]"

      - name: Run golden dataset eval
        env:
          LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
          LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GIT_SHA: ${{ github.sha }}
        run: |
          pytest tests/evals/test_golden_dataset.py -v --tb=short

      - name: Check eval scores
        run: |
          python scripts/check_eval_scores.py \
            --run-name "ci-${GITHUB_SHA:0:8}" \
            --min-faithfulness 0.85 \
            --min-relevancy 0.80
```

```python
# scripts/check_eval_scores.py
import argparse
import sys
from langfuse import Langfuse

def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--run-name", required=True)
    parser.add_argument("--min-faithfulness", type=float, default=0.85)
    parser.add_argument("--min-relevancy", type=float, default=0.80)
    args = parser.parse_args()

    langfuse = Langfuse()
    scores = langfuse.get_dataset_run_scores(run_name=args.run_name)

    avg = {k: sum(v) / len(v) for k, v in scores.items()}

    failed = []
    if avg.get("faithfulness", 0) < args.min_faithfulness:
        failed.append(f"faithfulness {avg['faithfulness']:.3f} < {args.min_faithfulness}")
    if avg.get("answer_relevancy", 0) < args.min_relevancy:
        failed.append(f"answer_relevancy {avg['answer_relevancy']:.3f} < {args.min_relevancy}")

    if failed:
        print(f"EVAL GATE FAILED: {'; '.join(failed)}")
        sys.exit(1)

    print(f"EVAL GATE PASSED: {avg}")
```

---

## Concept 17 — A/B Testing Prompts With Production Traffic

**Analogy:** A/B testing prompts is like split-testing two versions of a landing page. Half
your users see variant A, half see variant B. After a week, you compare conversion rates. For
prompts, "conversion" is your eval score: faithfulness, task completion rate, or user rating.

```python
import hashlib
from langfuse import Langfuse

langfuse = Langfuse()

def get_prompt_variant(user_id: str, experiment: str) -> str:
    """Deterministic A/B split — same user always gets same variant."""
    hash_input = f"{experiment}:{user_id}".encode()
    bucket = int(hashlib.md5(hash_input).hexdigest(), 16) % 100  # 0-99
    return "variant_b" if bucket < 20 else "control"  # 20% on variant B

@observe()
async def answer_question(user_query: str, user_id: str) -> str:
    variant = get_prompt_variant(user_id, "rag-prompt-v4-test")
    prompt_label = "staging" if variant == "variant_b" else "production"

    prompt = langfuse.get_prompt("rag-system-prompt", label=prompt_label)
    compiled = prompt.compile(query=user_query)

    langfuse_context.update_current_trace(
        metadata={
            "experiment": "rag-prompt-v4-test",
            "variant": variant,
            "prompt_version": prompt.version,
        },
        tags=[f"experiment:{variant}"],
    )

    response = await call_llm(compiled, user_query)
    return response
```

In Langfuse: filter traces by tag `experiment:variant_b` vs `experiment:control` and compare
average scores. Promote variant B to production when it consistently outperforms control.

---

## Common Gotchas

1. **Missing flush in Lambda / Cloud Run**: Serverless functions terminate the process after the
   handler returns. If `langfuse.flush()` is not called, zero observations reach Langfuse. Add
   it at the end of every serverless handler, unconditionally.

2. **Binding user_id too late**: If `update_current_trace(user_id=...)` is called after the LLM
   call returns, the user_id still appears on the trace (Langfuse merges updates). But if you
   want per-user cost dashboards in real time, bind user_id as early as possible in the trace.

3. **Double-tracing with LangSmith + Langfuse**: If `LANGCHAIN_TRACING_V2=true` and Langfuse's
   OpenAI integration are both active, every call is traced twice. Pick one platform per service.

4. **RAGAS requires ground truth for context_recall**: You cannot compute context_recall without
   a `ground_truth` field in your dataset. Faithfulness and answer_relevancy need no ground truth.
   Build ground truth from domain expert review, not from the same model output.

5. **LLM-as-judge self-evaluation bias**: Never use the same model to judge its own output.
   Use a different model (or different provider) as the judge. Claude-as-judge for GPT-4o
   output is fine. GPT-4o-as-judge for GPT-4o output is not.

6. **Token anomaly alerts are noisy without a baseline**: A 5x token spike is meaningless without
   a rolling average. Compute the 24-hour P95 token count per user segment and alert when current
   count exceeds 5x that baseline, not a hard token number.

---

## Quick Reference: Minimum Viable Observability Checklist

Every LLM endpoint in production MUST have:

- [ ] Trace created with `id=request_id`, `user_id`, `session_id`
- [ ] Generation logged with `model`, `usage.input`, `usage.output`
- [ ] `metadata.stop_reason` recorded
- [ ] `metadata.latency_ms` recorded
- [ ] Cost computed and stored (per generation, per user)
- [ ] `max_tokens` always set (never unlimited)
- [ ] `langfuse.flush()` called before process exit or at end of background task
- [ ] At least one automated eval score attached (LLM-judge or RAGAS) in staging
- [ ] Golden dataset CI gate for prompt changes
- [ ] Alert on error_rate > 1%, p99_latency > 30s, context_utilization > 90%

---

## Summary

LLM observability sits at the intersection of distributed tracing (request IDs, span trees),
cost engineering (token counting, model routing, budget limits), and quality engineering (evals,
golden datasets, regression gates). Langfuse and LangSmith both provide the plumbing — traces,
spans, generations, scores, datasets. OpenTelemetry provides the standardised attribute
vocabulary and vendor portability. RAGAS provides the RAG-specific quality metrics.

The non-negotiable production minimum: every LLM call must be traced with user_id + request_id,
every call must record token usage for cost attribution, every prompt change must be gated by
a CI eval run against a golden dataset, and every service must alert on the five key thresholds
before a degradation becomes a user-facing incident.
