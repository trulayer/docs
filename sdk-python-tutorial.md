# TruLayer AI — Python SDK Tutorial

A task-oriented guide to instrumenting Python AI applications with TruLayer.

## Table of Contents

1. [Installation](#installation)
2. [Client initialization](#client-initialization)
3. [Tracing a function manually](#tracing-a-function-manually)
4. [Adding spans](#adding-spans)
5. [Auto-instrumentation](#auto-instrumentation)
   - [OpenAI](#openai)
   - [Anthropic](#anthropic)
   - [LangChain](#langchain)
   - [LlamaIndex](#llamaindex)
   - [PydanticAI](#pydanticai)
6. [Adding feedback](#adding-feedback)
7. [Async usage](#async-usage)
8. [Redacting sensitive data](#redacting-sensitive-data)
9. [Local / offline mode](#local--offline-mode)
10. [Testing](#testing)

---

## Installation

```bash
pip install trulayer
```

Python 3.11 or later is required.

---

## Client initialization

### Using the global client (`trulayer.init`)

Call `trulayer.init()` once at application startup. Every subsequent call to `trulayer.get_client()` returns the same instance.

```python
import trulayer

trulayer.init(
    api_key="tl_...",          # required; or set TRULAYER_API_KEY env var
    project_name="my-project", # required
)
```

`init()` accepts the full set of constructor parameters:

| Parameter | Default | Description |
|---|---|---|
| `api_key` | `""` | TruLayer API key |
| `project_name` | — | Project name shown in the dashboard |
| `endpoint` | `"https://api.trulayer.ai"` | Override the ingestion endpoint |
| `batch_size` | `50` | Max events per HTTP batch |
| `flush_interval` | `2.0` | Seconds between automatic flushes |
| `sample_rate` | `1.0` | Fraction of traces to send (0.0–1.0) |
| `scrub_fn` | `None` | `(str) -> str` applied to all string fields before sending |
| `metadata_validator` | `None` | `(dict) -> None` that raises if metadata is invalid |
| `redactor` | `None` | `Redactor` instance for structured PII removal |

`init()` returns the `TruLayerClient` instance if you need to hold a reference.

### Using an explicit client

If you prefer not to use global state, construct `TruLayerClient` directly:

```python
from trulayer import TruLayerClient

client = TruLayerClient(
    api_key="tl_...",
    project_name="my-project",
)
```

The constructor accepts the same parameters as `init()`.

---

## Tracing a function manually

A trace represents one logical operation (one request, one job, one conversation turn). Open a trace with `client.trace()` as a context manager. The trace is enqueued and sent automatically when the `with` block exits.

```python
from trulayer import TruLayerClient

client = TruLayerClient(api_key="tl_...", project_name="my-project")

with client.trace(
    name="basic-qa",
    external_id="req-42",          # your own request ID — useful for deduplication
    tags=["production", "rag"],
    metadata={"user_id": "u_123"},
) as t:
    t.set_input("What is the capital of France?")
    # ... do work ...
    t.set_output("Paris.")
    t.set_model("gpt-4o-mini")
    t.set_cost(0.00012)            # optional, in USD
```

You can read the auto-generated trace ID from `t._data.id` after the trace is open.

**Trace methods at a glance:**

| Method | Description |
|---|---|
| `t.set_input(text)` | Top-level input shown in the dashboard |
| `t.set_output(text)` | Top-level output |
| `t.set_model(model)` | Model name rolled up to the trace |
| `t.set_cost(usd)` | Cost in USD |
| `t.set_metadata(**kwargs)` | Merge key-value pairs into trace metadata |
| `t.add_tag(tag)` | Append a tag |

---

## Adding spans

Spans represent individual steps within a trace (retrieval, LLM call, tool use). Open a span with `trace.span()`:

```python
with client.trace(name="rag-pipeline") as t:
    t.set_input(question)

    # Retrieval step
    with t.span("retrieve-context", span_type="retrieval") as s:
        s.set_input(question)
        docs = vector_store.search(question)
        s.set_output(str(docs))
        s.set_metadata(source="pinecone", result_count=len(docs))

    # LLM step
    with t.span("openai.chat", span_type="llm") as s:
        s.set_model("gpt-4o-mini")
        s.set_input(prompt)
        resp = openai_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
        )
        answer = resp.choices[0].message.content
        s.set_output(answer)
        s.set_tokens(
            prompt=resp.usage.prompt_tokens,
            completion=resp.usage.completion_tokens,
        )

    t.set_output(answer)
```

The `span_type` field controls how the span appears in the dashboard. Accepted values:

| Value | Use for |
|---|---|
| `"llm"` | LLM completion calls |
| `"retrieval"` | Vector store / document retrieval |
| `"tool"` | Function / tool calls in an agent |
| `"chain"` | Multi-step pipelines |
| `"agent"` | Agent loop steps |
| `"default"` | Everything else |

Latency is measured automatically. If an exception propagates out of a span's `with` block, `error=True` and `error_message` are set automatically on the span data.

**Span methods at a glance:**

| Method | Description |
|---|---|
| `s.set_input(text)` | Input to this step |
| `s.set_output(text)` | Output from this step |
| `s.set_model(model)` | Model name for this span |
| `s.set_tokens(prompt, completion)` | Token counts (both are `int | None`) |
| `s.set_metadata(**kwargs)` | Merge key-value pairs into span metadata |

---

## Auto-instrumentation

Auto-instrumentation wraps a provider's SDK so every call inside an active trace automatically produces a span. You still define the trace boundary; TruLayer fills in the provider spans.

### OpenAI

```python
from trulayer import instrument_openai, TruLayerClient
from openai import OpenAI

client = TruLayerClient(api_key="tl_...", project_name="my-project")
instrument_openai(client)  # patches openai.chat.completions.create globally

openai_client = OpenAI()

with client.trace(name="travel-qa", tags=["openai-auto"]) as t:
    t.set_input("Name one landmark in Paris.")
    resp = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Name one landmark in Paris."}],
    )
    t.set_output(resp.choices[0].message.content)
```

The patch is idempotent — calling `instrument_openai()` multiple times is safe. To remove the patch:

```python
from trulayer import uninstrument_openai
uninstrument_openai()
```

Streaming responses are handled transparently: the span is opened when the request starts and closed after the stream is fully consumed.

### Anthropic

```python
from trulayer import instrument_anthropic, uninstrument_anthropic, TruLayerClient
import anthropic

client = TruLayerClient(api_key="tl_...", project_name="my-project")
instrument_anthropic(client)

ac = anthropic.Anthropic()

with client.trace(name="landmark-qa") as t:
    t.set_input(question)
    resp = ac.messages.create(
        model="claude-3-5-sonnet-latest",
        max_tokens=128,
        messages=[{"role": "user", "content": question}],
    )
    text = next((b.text for b in resp.content if b.type == "text"), "")
    t.set_output(text)
```

To remove the patch: `uninstrument_anthropic()`.

### LangChain

`instrument_langchain()` returns a `BaseCallbackHandler` that you attach to any LangChain runnable. Every LLM call in the chain emits a `langchain.llm` span into the current trace.

```python
from trulayer import instrument_langchain, TruLayerClient
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

client = TruLayerClient(api_key="tl_...", project_name="my-project")
handler = instrument_langchain(client)

llm = ChatOpenAI(model="gpt-4o-mini")
chain = ChatPromptTemplate.from_messages([("human", "{question}")]) | llm | StrOutputParser()

with client.trace(name="langchain-rag", tags=["langchain"]) as t:
    t.set_input(question)
    answer = chain.invoke(question, config={"callbacks": [handler]})
    t.set_output(str(answer).strip())
```

Requires `langchain-core` to be installed.

### LlamaIndex

```python
import trulayer
from llama_index.core import Settings

trulayer.init(api_key="tl_...", project_name="my-project")
handler = trulayer.instrument_llamaindex()
Settings.callback_manager.handlers.append(handler)
```

Requires `llama-index-core` to be installed.

### PydanticAI

`instrument_pydanticai()` wraps an existing PydanticAI `Agent` instance in place. Every `.run()`, `.run_sync()`, and `.run_stream()` call and each tool invocation are recorded as spans inside the provided `TraceContext`.

```python
from trulayer import instrument_pydanticai, TruLayerClient
from pydantic_ai import Agent

client = TruLayerClient(api_key="tl_...", project_name="my-project")
agent = Agent("openai:gpt-4o-mini", system_prompt="Be concise.")

with client.trace(name="pydantic-agent") as t:
    instrument_pydanticai(
        agent,
        t,
        capture_input=True,
        capture_output=True,
        run_name="my-agent-run",  # optional span name override
    )
    result = agent.run_sync("What is the capital of France?")
```

---

## Adding feedback

Feedback attaches a human or automated label to a trace after it has been ingested. Use `client.feedback()` with the trace ID you captured during the trace.

```python
with client.trace(name="feedback-demo") as t:
    t.set_input(question)
    # ... do work ...
    t.set_output(answer)
    trace_id = t._data.id

# Flush before submitting feedback so the trace exists server-side.
client.flush()

client.feedback(
    trace_id=trace_id,
    label="good",              # any string; conventions: "good", "bad", "neutral"
    score=1.0,                 # optional float
    comment="Accurate answer.",
    metadata={"source": "user-thumbsup", "reviewer": "alice"},
)
```

`feedback()` is fire-and-forget. It sends synchronously but never raises into your code — failures are emitted as `warnings.warn`.

---

## Async usage

`TraceContext` and `SpanContext` are both sync and async context managers. The same `client.trace()` and `trace.span()` calls work with `async with`:

```python
import asyncio
from trulayer import TruLayerClient

client = TruLayerClient(api_key="tl_...", project_name="my-project")

async def run():
    async with client.trace(name="concurrent-qa", tags=["async"]) as t:
        t.set_input("two questions")

        async def ask(question: str, span_name: str) -> str:
            async with t.span(span_name, span_type="llm") as s:
                s.set_model("gpt-4o-mini")
                s.set_input(question)
                resp = await async_openai_client.chat.completions.create(
                    model="gpt-4o-mini",
                    messages=[{"role": "user", "content": question}],
                )
                answer = resp.choices[0].message.content
                s.set_output(answer)
                return answer

        answers = await asyncio.gather(
            ask("Tallest building in New York?", "ask-nyc"),
            ask("Tallest building in London?", "ask-london"),
        )
        t.set_output(" | ".join(answers))

asyncio.run(run())
```

Both spans land in the same trace because the `TraceContext` is captured in the closure rather than via a context variable lookup. `asyncio.gather` tasks that share a `TraceContext` reference work correctly.

---

## Redacting sensitive data

### Global scrub function

Pass a `scrub_fn` to the client constructor. It is applied to every string field (`input`, `output`, `error_message`) across all traces and spans before they leave your process.

```python
import re
from trulayer import TruLayerClient

def _scrub(text: str) -> str:
    return re.sub(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}", "<EMAIL>", text)

client = TruLayerClient(api_key="tl_...", project_name="my-project", scrub_fn=_scrub)
```

### Structured redaction with `Redactor`

`Redactor` composes built-in entity packs and custom rules into a reusable scrubber. Pass it to the client or use it standalone.

```python
from trulayer.redact import Redactor, Rule

redactor = Redactor(
    packs=["standard", "secrets"],    # enable built-in packs
    rules=[
        Rule(name="employee_id", pattern=r"EMP-\d{6}"),
    ],
)

# Standalone use
clean = redactor.redact("Contact foo@bar.com, EMP-123456")
# -> "Contact <REDACTED:email>, <REDACTED:employee_id>"

# Attach to client
from trulayer import TruLayerClient
client = TruLayerClient(api_key="tl_...", project_name="proj", redactor=redactor)
```

Built-in packs:

| Pack | Entities covered |
|---|---|
| `"standard"` | email, SSN, JWT, bearer token, phone |
| `"strict"` | everything in `standard` + credit card (Luhn-validated), IBAN, IPv4 |
| `"phi"` | medical record number, ICD-10 code, date of birth |
| `"finance"` | SWIFT/BIC, routing number, account number, ticker/amount |
| `"secrets"` | AWS access key, GitHub PAT, PEM private key, GCP service account JSON |

To redact specific fields in a span dict:

```python
clean_span = redactor.redact_span(
    span_dict,
    fields=["input", "output", "metadata.user.email"],  # dot-path notation
)
```

Pseudonymization replaces matched values with a deterministic HMAC token instead of a static placeholder:

```python
redactor = Redactor(
    packs=["standard"],
    pseudonymize=True,
    pseudonymize_salt="my-secret-salt",
    pseudonym_length=8,   # hex chars of HMAC digest (4–64)
)
```

---

## Local / offline mode

Set the environment variable `TRULAYER_MODE=local` to run without an API key or network access. Traces are stored in memory and never sent.

```bash
TRULAYER_MODE=local python my_app.py
```

In local mode the SDK emits a warning and creates a `LocalBatchSender` internally. You can also create a `LocalBatchSender` directly in tests — see [Testing](#testing).

---

## Testing

Use `create_test_client()` to get an in-memory client with no network I/O. Then use `assert_sender()` for fluent assertions.

```python
from trulayer.testing import create_test_client, assert_sender

def test_my_pipeline():
    client, sender = create_test_client(project_name="test-project")

    with client.trace("my-pipeline") as t:
        with t.span("step", "llm") as s:
            s.set_input("hello")
            s.set_output("world")

    client.flush()

    assert_sender(sender).has_trace().span_count(1).has_span_named("step")

    # Direct access to captured data
    assert sender.traces[0]["name"] == "my-pipeline"
    assert sender.spans[0]["input"] == "hello"
```

`create_test_client()` accepts all `TruLayerClient` keyword arguments except `api_key` and `_sender`.

`SenderAssertions` methods are chainable:

| Method | Asserts |
|---|---|
| `.has_trace(trace_id=None)` | At least one trace, or the specific ID |
| `.span_count(n)` | Exactly `n` spans across all traces |
| `.has_span_named(name)` | At least one span with the given name |

Call `sender.clear()` between test cases to reset captured state.
