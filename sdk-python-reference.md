# TruLayer AI — Python SDK API Reference

Full reference for the `trulayer` Python package (Python 3.11+).

## Table of Contents

- [Module-level functions](#module-level-functions)
  - [init](#init)
  - [get_client](#get_client)
  - [instrument_openai](#instrument_openai)
  - [uninstrument_openai](#uninstrument_openai)
  - [instrument_anthropic](#instrument_anthropic)
  - [uninstrument_anthropic](#uninstrument_anthropic)
  - [instrument_langchain](#instrument_langchain)
  - [instrument_llamaindex](#instrument_llamaindex)
  - [instrument_pydanticai](#instrument_pydanticai)
  - [instrument_crewai](#instrument_crewai)
  - [instrument_dspy / uninstrument_dspy](#instrument_dspy--uninstrument_dspy)
  - [instrument_haystack](#instrument_haystack)
  - [instrument_autogen](#instrument_autogen)
  - [current_trace](#current_trace)
- [TruLayerClient](#trulayerclient)
  - [Constructor](#constructor)
  - [trace](#trace)
  - [feedback](#feedback)
  - [flush](#flush)
  - [shutdown](#shutdown)
- [TraceContext](#tracecontext)
  - [span](#span)
  - [set_input](#traceset_input)
  - [set_output](#traceset_output)
  - [set_model](#traceset_model)
  - [set_cost](#set_cost)
  - [set_metadata](#traceset_metadata)
  - [add_tag](#add_tag)
- [SpanContext](#spancontext)
  - [set_input](#spanset_input)
  - [set_output](#spanset_output)
  - [set_model](#spanset_model)
  - [set_tokens](#set_tokens)
  - [set_metadata](#spanset_metadata)
- [Data models](#data-models)
  - [TraceData](#tracedata)
  - [SpanData](#spandata)
  - [EventData](#eventdata)
  - [FeedbackData](#feedbackdata)
- [Redaction](#redaction)
  - [Rule](#rule)
  - [Redactor](#redactor)
  - [redact (function)](#redact-function)
  - [BUILTIN_PACKS](#builtin_packs)
- [Testing helpers](#testing-helpers)
  - [create_test_client](#create_test_client)
  - [LocalBatchSender](#localbatchsender)
  - [assert_sender](#assert_sender)
  - [SenderAssertions](#senderassertions)
- [Environment variables](#environment-variables)
- [Exceptions](#exceptions)

---

## Module-level functions

All public names are importable directly from `trulayer`:

```python
import trulayer
from trulayer import TruLayerClient, TraceContext, instrument_openai, ...
```

---

### `init`

```python
trulayer.init(
    api_key: str,
    project_name: str | None = None,
    endpoint: str = "https://api.trulayer.ai",
    batch_size: int = 50,
    flush_interval: float = 2.0,
    sample_rate: float = 1.0,
    scrub_fn: Callable[[str], str] | None = None,
    metadata_validator: Callable[[dict[str, Any]], None] | None = None,
    redactor: Redactor | None = None,
    project_id: str | None = None,  # deprecated
) -> TruLayerClient
```

Initialize the process-global `TruLayerClient`. Call once at application startup.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `api_key` | `str` | — | TruLayer API key |
| `project_name` | `str \| None` | `None` | Project name shown in the dashboard. Required unless `TRULAYER_MODE=local` |
| `endpoint` | `str` | `"https://api.trulayer.ai"` | Override the ingestion API base URL |
| `batch_size` | `int` | `50` | Maximum events per outbound HTTP batch |
| `flush_interval` | `float` | `2.0` | Seconds between automatic background flushes |
| `sample_rate` | `float` | `1.0` | Fraction of traces to send (0.0 = drop all, 1.0 = send all). Must be in [0.0, 1.0] |
| `scrub_fn` | `Callable[[str], str] \| None` | `None` | Applied to all string-typed trace and span fields (`input`, `output`, `error_message`) before sending |
| `metadata_validator` | `Callable[[dict], None] \| None` | `None` | Called for each metadata dict; if it raises, the metadata is cleared and a warning is emitted |
| `redactor` | `Redactor \| None` | `None` | Structured PII redactor (see [`Redactor`](#redactor)) |
| `project_id` | `str \| None` | `None` | Deprecated alias for `project_name`. Removed in 0.3.x |

**Returns** `TruLayerClient`

**Raises** `ValueError` if `sample_rate` is outside [0.0, 1.0].

---

### `get_client`

```python
trulayer.get_client() -> TruLayerClient
```

Return the client initialized by `trulayer.init()`.

**Raises** `RuntimeError` if `trulayer.init()` has not been called.

---

### `instrument_openai`

```python
trulayer.instrument_openai(client: TruLayerClient) -> None
```

Monkey-patch `openai.resources.chat.completions.Completions.create` (and the async variant) to emit a `"openai.chat"` span with `span_type="llm"` for every call made inside an active `TraceContext`. Idempotent — safe to call multiple times. Streaming responses are handled transparently.

**Requires** `openai` package. Emits `UserWarning` and returns without patching if the package is not installed.

---

### `uninstrument_openai`

```python
trulayer.uninstrument_openai() -> None
```

Restore the original OpenAI methods. Idempotent.

---

### `instrument_anthropic`

```python
trulayer.instrument_anthropic(client: TruLayerClient) -> None
```

Monkey-patch `anthropic.resources.messages.Messages.create` (and the async variant) to emit an `"anthropic.messages"` span with `span_type="llm"` for every call inside an active `TraceContext`. Idempotent. Streaming is handled transparently.

**Requires** `anthropic` package.

---

### `uninstrument_anthropic`

```python
trulayer.uninstrument_anthropic() -> None
```

Restore the original Anthropic methods. Idempotent.

---

### `instrument_langchain`

```python
trulayer.instrument_langchain(client: TruLayerClient) -> TruLayerCallbackHandler
```

Return a LangChain `BaseCallbackHandler` that records each LLM call as a `"langchain.llm"` span with `span_type="llm"` inside the current `TraceContext`. Pass the returned handler to any LangChain LLM, chat model, or chain via `callbacks=[handler]`.

**Requires** `langchain-core` (or `langchain`). Raises `ImportError` if neither is installed.

**Returns** `TruLayerCallbackHandler` (a dynamically-constructed `BaseCallbackHandler` subclass)

**Example**

```python
handler = trulayer.instrument_langchain(client)
answer = chain.invoke(question, config={"callbacks": [handler]})
```

---

### `instrument_llamaindex`

```python
trulayer.instrument_llamaindex() -> TruLayerCallbackHandler
```

Return a LlamaIndex callback handler that records query and LLM events as TruLayer spans. Attach it to a LlamaIndex `CallbackManager`.

**Requires** `llama-index-core`. Uses the global client set by `trulayer.init()`.

**Raises** `RuntimeError` if `trulayer.init()` has not been called.

**Example**

```python
from llama_index.core import Settings
handler = trulayer.instrument_llamaindex()
Settings.callback_manager.handlers.append(handler)
```

---

### `instrument_pydanticai`

```python
trulayer.instrument_pydanticai(
    agent: Any,
    trace_ctx: TraceContext,
    *,
    capture_input: bool = True,
    capture_output: bool = True,
    run_name: str | None = None,
) -> Any
```

Wrap a PydanticAI `Agent` in place. Every `.run()`, `.run_sync()`, and `.run_stream()` call is recorded as a span with `span_type="agent"`. Each tool in `agent._function_tools` is also wrapped as a `span_type="tool"` child span.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `agent` | `Any` | — | PydanticAI `Agent` instance to instrument |
| `trace_ctx` | `TraceContext` | — | The active trace to attach spans to |
| `capture_input` | `bool` | `True` | Record the user prompt as span input |
| `capture_output` | `bool` | `True` | Record the agent result as span output |
| `run_name` | `str \| None` | `None` | Override the span name. Defaults to `"pydanticai:{agent.name}"` |

**Returns** The same `agent` instance (mutated in place). Never raises into user code.

---

### `instrument_crewai`

```python
trulayer.instrument_crewai(client: TruLayerClient) -> None
```

Auto-instrumentation for CrewAI. Emits spans for crew and agent execution events inside active traces.

---

### `instrument_dspy` / `uninstrument_dspy`

```python
trulayer.instrument_dspy(client: TruLayerClient) -> None
trulayer.uninstrument_dspy() -> None
```

Auto-instrumentation for DSPy. Patches DSPy's LM calls to emit spans inside active traces. `uninstrument_dspy()` restores original methods.

---

### `instrument_haystack`

```python
trulayer.instrument_haystack(client: TruLayerClient) -> None
```

Auto-instrumentation for Haystack pipelines. Emits spans for component runs inside active traces.

---

### `instrument_autogen`

```python
trulayer.instrument_autogen(client: TruLayerClient) -> None
```

Auto-instrumentation for AutoGen. Emits spans for agent conversations inside active traces.

---

### `current_trace`

```python
trulayer.current_trace() -> TraceContext | None
```

Return the `TraceContext` active in the current execution context (thread or async task), or `None` if no trace is open. Used internally by auto-instrumentation hooks.

---

## TruLayerClient

```python
from trulayer import TruLayerClient
```

Main SDK client. One instance per process is typical.

### Constructor

```python
TruLayerClient(
    api_key: str = "",
    project_name: str | None = None,
    endpoint: str = "https://api.trulayer.ai",
    batch_size: int = 50,
    flush_interval: float = 2.0,
    sample_rate: float = 1.0,
    scrub_fn: Callable[[str], str] | None = None,
    metadata_validator: Callable[[dict[str, Any]], None] | None = None,
    redactor: Redactor | None = None,
    project_id: str | None = None,  # deprecated
)
```

Parameters are identical to [`trulayer.init()`](#init). An `atexit` handler is registered automatically to flush pending events on process exit.

**Raises** `ValueError` if `sample_rate` is outside [0.0, 1.0].  
**Raises** `TypeError` if `project_name` is not provided and `TRULAYER_MODE` is not `"local"`.

---

### `trace`

```python
client.trace(
    name: str | None = None,
    session_id: str | None = None,
    external_id: str | None = None,
    tags: list[str] | None = None,
    metadata: dict[str, Any] | None = None,
) -> TraceContext
```

Create and return a `TraceContext`. Use as a sync or async context manager.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `name` | `str \| None` | Human-readable operation name |
| `session_id` | `str \| None` | Group multiple traces into a session (e.g. a conversation) |
| `external_id` | `str \| None` | Your own identifier for idempotent ingest |
| `tags` | `list[str] \| None` | Filterable labels |
| `metadata` | `dict[str, Any] \| None` | Arbitrary key-value pairs |

**Returns** `TraceContext`

---

### `feedback`

```python
client.feedback(
    trace_id: str,
    label: str,
    score: float | None = None,
    comment: str | None = None,
    metadata: dict[str, Any] | None = None,
) -> None
```

Submit feedback for a trace. Sends synchronously via `httpx.post`. Never raises — HTTP failures are emitted as `UserWarning`.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `trace_id` | `str` | ID of the target trace (`t._data.id`) |
| `label` | `str` | Short label; conventional values: `"good"`, `"bad"`, `"neutral"` |
| `score` | `float \| None` | Numeric score |
| `comment` | `str \| None` | Freeform text note |
| `metadata` | `dict[str, Any] \| None` | Additional key-value pairs |

**Returns** `None`

---

### `flush`

```python
client.flush(timeout: float = 5.0) -> None
```

Block until all buffered events have been sent, or until `timeout` seconds elapse. The background batch sender is restarted after flush so subsequent traces continue to work.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `timeout` | `float` | `5.0` | Max seconds to wait |

---

### `shutdown`

```python
client.shutdown(timeout: float = 5.0) -> None
```

Flush all buffered events and stop the background sender. Called automatically by the `atexit` handler registered at construction time. After `shutdown()`, the client will not send further events unless `flush()` restarts the sender.

---

## TraceContext

Returned by `client.trace()`. Works as both a sync context manager (`with`) and an async context manager (`async with`). Latency is measured from `__enter__` to `__exit__`. If an exception propagates out of the block, `error=True` is set on the trace.

The trace payload is enqueued to the batch sender in `__exit__`. Sampling is applied at entry: if `sample_rate < 1.0`, some traces are silently dropped.

The active `TraceContext` is stored in a `contextvars.ContextVar` and can be retrieved with `trulayer.current_trace()`.

---

### `span`

```python
trace.span(name: str, span_type: str = "other") -> SpanContext
```

Create a child `SpanContext`. Use as a sync or async context manager.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | — | Span name |
| `span_type` | `str` | `"other"` | One of `"llm"`, `"retrieval"`, `"tool"`, `"other"` |

**Returns** `SpanContext`

---

### `trace.set_input`

```python
trace.set_input(value: str) -> None
```

Set the top-level input for the trace.

---

### `trace.set_output`

```python
trace.set_output(value: str) -> None
```

Set the top-level output for the trace.

---

### `trace.set_model`

```python
trace.set_model(model: str) -> None
```

Set the model name rolled up to the trace level.

---

### `set_cost`

```python
trace.set_cost(cost: float) -> None
```

Set the estimated cost in USD for the entire trace.

---

### `trace.set_metadata`

```python
trace.set_metadata(**kwargs: Any) -> None
```

Merge one or more key-value pairs into the trace's `metadata` dict.

---

### `add_tag`

```python
trace.add_tag(tag: str) -> None
```

Append a tag string to the trace's `tags` list.

---

### `set_tag`

```python
trace.set_tag(key: str, value: str) -> None
```

Attach a structured key → value tag to the trace. Unlike `add_tag`, structured tags are indexed server-side and filterable via the `tag_key` / `tag_value` query parameters on list endpoints.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `key` | `str` | Tag key. Max 64 characters. Max 20 keys per trace |
| `value` | `str` | Tag value. Max 64 characters |

**Example**

```python
trace.set_tag("environment", "production")
trace.set_tag("region", "us-west-2")
```

---

## SpanContext

Returned by `trace.span()`. Works as both a sync and async context manager. Latency is measured automatically. If an exception propagates out of the block, `error=True` and `error_message` (formatted traceback) are set on the span.

---

### `span.set_input`

```python
span.set_input(value: str) -> None
```

Set the input recorded for this span.

---

### `span.set_output`

```python
span.set_output(value: str) -> None
```

Set the output recorded for this span.

---

### `span.set_model`

```python
span.set_model(model: str) -> None
```

Set the model name for this span.

---

### `set_tokens`

```python
span.set_tokens(prompt: int | None = None, completion: int | None = None) -> None
```

Set token counts for the span. Both parameters are optional.

| Parameter | Type | Description |
|---|---|---|
| `prompt` | `int \| None` | Prompt / input token count |
| `completion` | `int \| None` | Completion / output token count |

---

### `span.set_metadata`

```python
span.set_metadata(**kwargs: Any) -> None
```

Merge one or more key-value pairs into the span's `metadata` dict.

---

## Data models

Pydantic v2 models used to serialize trace data before sending to the API. All IDs are UUIDv7 strings generated at construction time.

### `TraceData`

```python
from trulayer import TraceData
```

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | `str` | auto (UUIDv7) | Trace identifier |
| `project_id` | `str` | — | Project name/identifier |
| `session_id` | `str \| None` | `None` | Session grouping |
| `external_id` | `str \| None` | `None` | Caller-provided identifier |
| `name` | `str \| None` | `None` | Operation name |
| `input` | `str \| None` | `None` | Top-level input |
| `output` | `str \| None` | `None` | Top-level output |
| `model` | `str \| None` | `None` | Model name |
| `latency_ms` | `int \| None` | `None` | Duration in milliseconds (set automatically) |
| `cost` | `float \| None` | `None` | Cost in USD |
| `error` | `str \| None` | `None` | Exception traceback string, or `None` if the trace succeeded |
| `tags` | `list[str]` | `[]` | Tag list |
| `tag_map` | `dict[str, str] \| None` | `None` | Structured key→value tags (server-indexed, filterable via `tag_key`/`tag_value` query params) |
| `metadata` | `dict[str, Any]` | `{}` | Arbitrary metadata |
| `spans` | `list[SpanData]` | `[]` | Child spans |
| `started_at` | `datetime` | now (UTC) | Wall-clock start time |
| `ended_at` | `datetime \| None` | `None` | Wall-clock end time (set on exit) |

---

### `SpanData`

```python
from trulayer import SpanData
```

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | `str` | auto (UUIDv7) | Span identifier |
| `trace_id` | `str` | `""` | Parent trace ID |
| `name` | `str` | — | Span name |
| `span_type` | `str` | `"other"` | `"llm"`, `"retrieval"`, `"tool"`, `"other"` |
| `input` | `str \| None` | `None` | Input to this step |
| `output` | `str \| None` | `None` | Output from this step |
| `error` | `str \| None` | `None` | Exception traceback string, or `None` if the span succeeded |
| `latency_ms` | `int \| None` | `None` | Duration in milliseconds (set automatically) |
| `model` | `str \| None` | `None` | Model name |
| `prompt_tokens` | `int \| None` | `None` | Prompt token count |
| `completion_tokens` | `int \| None` | `None` | Completion token count |
| `metadata` | `dict[str, Any]` | `{}` | Arbitrary metadata |
| `started_at` | `datetime` | now (UTC) | Wall-clock start time |
| `ended_at` | `datetime \| None` | `None` | Wall-clock end time (set on exit) |

---

### `EventData`

```python
from trulayer import EventData
```

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | `str` | auto (UUIDv7) | Event identifier |
| `trace_id` | `str` | `""` | Parent trace ID |
| `span_id` | `str \| None` | `None` | Parent span ID |
| `level` | `str` | `"info"` | `"debug"`, `"info"`, `"warning"`, `"error"` |
| `message` | `str` | — | Event message |
| `metadata` | `dict[str, Any]` | `{}` | Arbitrary metadata |
| `timestamp` | `datetime` | now (UTC) | Wall-clock timestamp |

---

### `FeedbackData`

```python
from trulayer import FeedbackData
```

| Field | Type | Description |
|---|---|---|
| `trace_id` | `str` | Target trace ID |
| `label` | `str` | Short label (`"good"`, `"bad"`, `"neutral"`, or any string) |
| `score` | `float \| None` | Numeric score |
| `comment` | `str \| None` | Freeform text |
| `metadata` | `dict[str, Any]` | Additional key-value pairs |

---

## Redaction

```python
from trulayer.redact import Redactor, Rule, redact, BUILTIN_PACKS
# or
from trulayer import Redactor, Rule, redact, BUILTIN_PACKS
```

### `Rule`

```python
@dataclass
class Rule:
    name: str
    pattern: str | re.Pattern[str]
    replacement: str | None = None
    pseudonymize: bool | None = None
    validator: Any = None
```

A single redaction rule.

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Short identifier used in `<REDACTED:{name}>` tokens |
| `pattern` | `str \| re.Pattern` | Regular expression to match sensitive values |
| `replacement` | `str \| None` | Custom replacement string. When `None`, emits `<REDACTED:{name}>` |
| `pseudonymize` | `bool \| None` | Override the redactor-level pseudonymize default for this rule |
| `validator` | `Any` | Optional `(match_str) -> bool` — return `False` to skip redaction |

---

### `Redactor`

```python
class Redactor:
    def __init__(
        self,
        packs: Sequence[str] | None = None,
        rules: Sequence[Rule] | None = None,
        pseudonymize: bool = False,
        pseudonymize_salt: str | bytes | None = None,
        pseudonym_length: int = 8,
    ) -> None
```

Compose built-in packs and custom rules into a reusable scrubber.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `packs` | `Sequence[str] \| None` | `None` | Names of built-in packs to enable (see [`BUILTIN_PACKS`](#builtin_packs)) |
| `rules` | `Sequence[Rule] \| None` | `None` | Additional custom rules applied after the packs |
| `pseudonymize` | `bool` | `False` | When `True`, all rules emit HMAC pseudonyms instead of static tokens |
| `pseudonymize_salt` | `str \| bytes \| None` | `None` | Secret for HMAC-SHA256; required when any rule pseudonymizes |
| `pseudonym_length` | `int` | `8` | Number of hex characters of the HMAC digest to retain (4–64) |

**Raises** `ValueError` for unknown pack names, invalid `pseudonym_length`, or pseudonymization requested without a salt.

#### `Redactor.redact`

```python
redactor.redact(text: str) -> str
```

Return `text` with every configured rule applied in order.

#### `Redactor.redact_span`

```python
redactor.redact_span(
    span: Mapping[str, Any],
    fields: Iterable[str] = ("input", "output", "metadata"),
) -> dict[str, Any]
```

Return a shallow copy of `span` with the listed fields redacted. Field paths support dot notation (`"metadata.user.email"`) to target nested values. Missing paths are silently skipped.

---

### `redact` (function)

```python
trulayer.redact(
    text: str,
    packs: Sequence[str] = ("standard",),
    rules: Sequence[Rule] | None = None,
    pseudonymize: bool = False,
    pseudonymize_salt: str | bytes | None = None,
) -> str
```

One-shot convenience wrapper. Constructs a `Redactor` and applies it to `text` once. For repeated use prefer a persistent `Redactor` instance to avoid recompiling regexes.

---

### `BUILTIN_PACKS`

```python
BUILTIN_PACKS: dict[str, list[tuple[str, str, Any]]]
```

Registry of built-in entity packs. Keys are pack names; values are lists of `(entity_name, pattern, validator)` tuples.

| Pack | Entity names |
|---|---|
| `"standard"` | `email`, `ssn`, `jwt`, `bearer_token`, `phone` |
| `"strict"` | everything in `standard` + `credit_card` (Luhn-validated), `iban`, `ipv4` |
| `"phi"` | `mrn`, `icd10`, `dob` |
| `"finance"` | `swift_bic`, `routing_number`, `account_number`, `ticker_amount` |
| `"secrets"` | `aws_access_key`, `github_pat`, `pem_private_key`, `gcp_service_account` |

---

## Testing helpers

```python
from trulayer.testing import create_test_client, assert_sender, SenderAssertions
# or
from trulayer import create_test_client, assert_sender, SenderAssertions
```

### `create_test_client`

```python
create_test_client(**kwargs: Any) -> tuple[TruLayerClient, LocalBatchSender]
```

Return a `(client, sender)` pair for unit tests. No API key is required. No network I/O occurs. Any `TruLayerClient` keyword argument except `api_key` and `_sender` can be passed via `**kwargs`.

```python
client, sender = create_test_client(project_name="test")
```

---

### `LocalBatchSender`

```python
from trulayer import LocalBatchSender
```

In-memory trace collector. Drop-in replacement for the real batch sender. Activated automatically when `TRULAYER_MODE=local`.

| Member | Type | Description |
|---|---|---|
| `traces` | `list[dict[str, Any]]` | Property — all captured traces (flat) |
| `spans` | `list[dict[str, Any]]` | Property — all captured spans (flat) |
| `clear()` | `-> None` | Reset all captured state |

Set `TRULAYER_LOCAL_VERBOSE=1` to print a line to stdout for each captured trace.

---

### `assert_sender`

```python
assert_sender(sender: LocalBatchSender) -> SenderAssertions
```

Return a `SenderAssertions` wrapper for fluent assertions on captured data.

---

### `SenderAssertions`

All methods return `self` for chaining.

| Method | Signature | Asserts |
|---|---|---|
| `has_trace` | `(trace_id: str \| None = None) -> SenderAssertions` | At least one trace exists, or the specific `trace_id` is present |
| `span_count` | `(n: int) -> SenderAssertions` | Exactly `n` spans exist across all captured traces |
| `has_span_named` | `(name: str) -> SenderAssertions` | At least one span with `name` exists |

All methods raise `AssertionError` on failure.

```python
assert_sender(sender).has_trace().span_count(2).has_span_named("retrieve-docs")
```

---

## Environment variables

| Variable | Description |
|---|---|
| `TRULAYER_MODE=local` | Run in local mode — no API key required, traces are stored in memory via `LocalBatchSender`. A `UserWarning` is emitted at startup |
| `TRULAYER_LOCAL_VERBOSE=1` | When running in local mode, print a summary line to stdout for each captured trace |

---

## Exceptions

The SDK is designed to never raise exceptions into user code during normal operation (trace/span creation, batch sending, feedback submission, auto-instrumentation). All failures are surfaced via `warnings.warn`. The following situations do raise directly:

| Exception | When |
|---|---|
| `RuntimeError` | `trulayer.get_client()` called before `trulayer.init()` |
| `RuntimeError` | `trulayer.instrument_llamaindex()` called before `trulayer.init()` |
| `ValueError` | `sample_rate` outside [0.0, 1.0] in `TruLayerClient` constructor or `trulayer.init()` |
| `TypeError` | `project_name` not provided and `TRULAYER_MODE` is not `"local"` |
| `ValueError` | Unknown pack name passed to `Redactor` |
| `ValueError` | `pseudonym_length` outside [4, 64] in `Redactor` |
| `ValueError` | `pseudonymize=True` without `pseudonymize_salt` in `Redactor` |
| `ImportError` | `instrument_langchain()` called without `langchain-core` or `langchain` installed |
