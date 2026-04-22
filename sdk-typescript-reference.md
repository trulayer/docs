# TruLayer AI — TypeScript SDK API Reference

Full reference for the `@trulayer/sdk` npm package (Node.js 18+, Edge runtimes, Bun).

## Table of Contents

- [Module-level functions](#module-level-functions)
  - [init](#init)
  - [getClient](#getclient)
- [TruLayer](#trulayer)
  - [Constructor](#constructor)
  - [trace](#trace)
  - [feedback](#feedback)
  - [flush](#flush)
  - [shutdown](#shutdown)
- [TraceContext](#tracecontext)
  - [span](#tracecontextspan)
  - [setInput](#tracecontextsetinput)
  - [setOutput](#tracecontextsetoutput)
  - [setModel](#tracecontextsetmodel)
  - [setCost](#tracecontextsetcost)
  - [setMetadata](#tracecontextsetmetadata)
  - [addTag](#tracecontextaddtag)
  - [data](#tracecontextdata)
- [SpanContext](#spancontext)
  - [span](#spancontextspan)
  - [setInput](#spancontextsetinput)
  - [setOutput](#spancontextsetoutput)
  - [setModel](#spancontextsetmodel)
  - [setTokens](#spancontextsettokens)
  - [setMetadata](#spancontextsetmetadata)
  - [data](#spancontextdata)
- [NoopTraceContext and NoopSpanContext](#nooptracecontext-and-noopspancontext)
- [Auto-instrumentation](#auto-instrumentation)
  - [instrumentOpenAI](#instrumentopenai)
  - [instrumentAnthropic](#instrumentanthropic)
  - [instrumentVercelAI](#instrumentvercelai)
  - [TruLayerCallbackHandler](#trulayercallbackhandler)
- [Redaction](#redaction)
  - [Redactor](#redactor)
  - [redact (function)](#redact-function)
  - [BUILTIN_PACKS](#builtin_packs)
  - [Rule](#rule)
  - [RedactorOptions](#redactoroptions)
- [Data models](#data-models)
  - [TruLayerConfig](#trulayerconfig)
  - [TraceData](#tracedata)
  - [SpanData](#spandata)
  - [FeedbackData](#feedbackdata)
  - [SpanType](#spantype)
- [Testing helpers](#testing-helpers)
  - [createTestClient](#createtestclient)
  - [assertSender](#assertsender)
  - [SenderAssertions](#senderassertions)
  - [LocalBatchSender](#localbatchsender)
  - [CapturedBatch](#capturedbatch)
- [Browser entry point](#browser-entry-point)
  - [initBrowser](#initbrowser)
  - [BrowserConfig](#browserconfig)
- [Environment variables](#environment-variables)

---

## Module-level functions

All public names are importable directly from `@trulayer/sdk`:

```typescript
import { TruLayer, init, getClient, instrumentOpenAI, Redactor, createTestClient } from '@trulayer/sdk'
```

---

### `init`

```typescript
function init(config: TruLayerConfig): TruLayer
```

Initialize the process-global `TruLayer` client. Call once at application startup. Subsequent calls replace the global instance.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `config.apiKey` | `string` | — | TruLayer API key. Required unless `TRULAYER_MODE=local` |
| `config.projectName` | `string` | — | Project name shown in the dashboard. Required unless `TRULAYER_MODE=local` |
| `config.endpoint` | `string` | `"https://api.trulayer.ai"` | Override the ingestion API base URL |
| `config.batchSize` | `number` | `50` | Maximum events per outbound HTTP batch |
| `config.flushInterval` | `number` | `2000` | Milliseconds between automatic background flushes |
| `config.sampleRate` | `number` | `1.0` | Fraction of traces to send (0.0 = drop all, 1.0 = send all) |
| `config.redact` | `(data: unknown) => unknown` | — | Applied to every string field before enqueuing. Return value replaces the original. Throw to store `null` |
| `config.relayUrl` | `string` | — | Server-side relay URL for browser mode. See [Browser entry point](#browser-entry-point) |

**Returns** the `TruLayer` client instance.

**Throws** `Error` if `apiKey` or `projectName` is missing and `TRULAYER_MODE` is not `"local"`.

**Example**

```typescript
import { init } from '@trulayer/sdk'

const tl = init({
  apiKey: process.env.TRULAYER_API_KEY!,
  projectName: 'my-project',
  sampleRate: 0.5,  // sample 50% of traces
})
```

---

### `getClient`

```typescript
function getClient(): TruLayer
```

Return the global client created by `init()`.

**Returns** the `TruLayer` instance.

**Throws** `Error` with message `[trulayer] call init() before getClient()` if `init()` has not been called.

**Example**

```typescript
import { init, getClient } from '@trulayer/sdk'

// At startup:
init({ apiKey: 'tl_...', projectName: 'my-project' })

// Anywhere else in the codebase:
const tl = getClient()
```

---

## TruLayer

```typescript
import { TruLayer } from '@trulayer/sdk'
```

The main client class. Manages configuration, batching, and trace/span lifecycle.

### Constructor

```typescript
new TruLayer(config: TruLayerConfig, _batchOverride?: BatchSenderLike)
```

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `config` | `TruLayerConfig` | Client configuration. See [TruLayerConfig](#trulayerconfig) |
| `_batchOverride` | `BatchSenderLike` | Optional — inject a custom sender (used internally for browser and test modes) |

**Throws** `Error` if `apiKey` or `projectName` is missing and `TRULAYER_MODE` is not `"local"`.

**Example**

```typescript
import { TruLayer } from '@trulayer/sdk'

const tl = new TruLayer({
  apiKey: 'tl_...',
  projectName: 'my-project',
  batchSize: 100,
  flushInterval: 5000,
})
```

---

### `trace`

```typescript
async trace<T>(
  name: string,
  callback: (trace: TraceContext | NoopTraceContext) => Promise<T>,
  options?: {
    sessionId?: string
    externalId?: string
    tags?: string[]
    metadata?: Record<string, unknown>
  },
): Promise<T>
```

Open a trace, run the callback, and enqueue the trace for upload when the callback resolves.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Human-readable trace name shown in the dashboard |
| `callback` | `(trace: TraceContext \| NoopTraceContext) => Promise<T>` | Async function that runs the traced operation |
| `options.sessionId` | `string` | Group related traces into a conversation session |
| `options.externalId` | `string` | Your own identifier (request ID, job ID) for deduplication |
| `options.tags` | `string[]` | Labels for filtering in the dashboard |
| `options.metadata` | `Record<string, unknown>` | Arbitrary JSON attached to the trace |

**Returns** `Promise<T>` — resolves with the callback's return value.

**Throws** Re-throws any error that propagates from the callback. The trace is marked `error: true` before rethrowing.

When `sampleRate < 1.0` and the trace is sampled out, the callback still executes but receives a `NoopTraceContext` — no data is sent.

**Example**

```typescript
const answer = await tl.trace(
  'qa-pipeline',
  async (t) => {
    t.setInput(question)
    const result = await runPipeline(question)
    t.setOutput(result)
    return result
  },
  {
    sessionId: 'session-abc',
    externalId: 'req-123',
    tags: ['production'],
    metadata: { userId: 'u_456' },
  },
)
```

---

### `feedback`

```typescript
feedback(
  traceId: string,
  label: string,
  options?: {
    score?: number
    comment?: string
    metadata?: Record<string, unknown>
  },
): void
```

Attach a human or automated label to an already-ingested trace. Fire-and-forget — never throws.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `traceId` | `string` | ID of the trace to label (from `t.data.id`) |
| `label` | `string` | Arbitrary label string. Conventional values: `"good"`, `"bad"`, `"neutral"` |
| `options.score` | `number` | Optional numeric score (e.g. 0.0–1.0) |
| `options.comment` | `string` | Free-text comment |
| `options.metadata` | `Record<string, unknown>` | Arbitrary JSON attached to the feedback event |

**Returns** `void`. HTTP failures are emitted as `console.warn`.

**Example**

```typescript
// Capture the trace ID during the trace
let traceId: string = ''
await tl.trace('my-trace', async (t) => {
  traceId = t.data.id
  // ...
})

// Flush so the trace is ingested before feedback references it
await tl.shutdown()

tl.feedback(traceId, 'good', {
  score: 1.0,
  comment: 'Correct and concise.',
  metadata: { source: 'thumbs-up', reviewer: 'alice' },
})
```

---

### `flush`

```typescript
flush(): void
```

Trigger an immediate flush of the event buffer. Does not wait for the HTTP request to complete.

**Example**

```typescript
tl.flush()
```

---

### `shutdown`

```typescript
shutdown(): Promise<void>
```

Flush all pending events and shut down the background flush loop. Call this before your process exits to ensure no data is lost.

**Returns** `Promise<void>` that resolves when the final flush attempt completes.

**Example**

```typescript
await tl.shutdown()
process.exit(0)
```

---

## TraceContext

```typescript
import { TraceContext } from '@trulayer/sdk'
```

The context object passed to the `trace()` callback. Provides methods to annotate the trace and open child spans. Latency is measured automatically from when the `TraceContext` is created.

---

### `TraceContext.span`

```typescript
async span<T>(
  name: string,
  spanType: SpanType,
  callback: (span: SpanContext) => Promise<T>,
  explicitParentId?: string,
): Promise<T>
```

Open a child span, run the callback, and record the span when it resolves.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Span name (e.g. `"openai.chat"`, `"retrieve-docs"`) |
| `spanType` | `SpanType` | Category. See [SpanType](#spantype) |
| `callback` | `(span: SpanContext) => Promise<T>` | Async function that runs the step |
| `explicitParentId` | `string` | Optional — set a specific parent span ID (overrides AsyncLocalStorage) |

**Returns** `Promise<T>` — the callback's return value.

**Throws** Re-throws any error from the callback. `span.data.error` and `span.data.error_message` are set before rethrowing.

Span nesting is tracked automatically via `AsyncLocalStorage` on Node.js, Bun, and runtimes that expose `globalThis.AsyncLocalStorage`. On Edge runtimes without `AsyncLocalStorage`, parent span IDs are omitted.

**Example**

```typescript
const result = await t.span('embed-query', 'default', async (s) => {
  s.setModel('text-embedding-3-small')
  s.setInput(question)
  const vec = await embed(question)
  s.setOutput(`vector[${vec.length}]`)
  return vec
})
```

---

### `TraceContext.setInput`

```typescript
setInput(value: string): this
```

Set the top-level input displayed in the dashboard. If a `redact` function is configured, it is applied before storing.

**Returns** `this` for chaining.

---

### `TraceContext.setOutput`

```typescript
setOutput(value: string): this
```

Set the top-level output displayed in the dashboard. If a `redact` function is configured, it is applied before storing.

**Returns** `this` for chaining.

---

### `TraceContext.setModel`

```typescript
setModel(model: string): this
```

Set the model name rolled up to the trace level.

**Returns** `this` for chaining.

---

### `TraceContext.setCost`

```typescript
setCost(cost: number): this
```

Set the cost of this trace in USD.

**Returns** `this` for chaining.

---

### `TraceContext.setMetadata`

```typescript
setMetadata(meta: Record<string, unknown>): this
```

Merge additional key-value pairs into the trace's `metadata` object. Subsequent calls merge — they do not replace.

**Returns** `this` for chaining.

**Example**

```typescript
t.setMetadata({ userId: 'u_123', region: 'us-west-2' })
```

---

### `TraceContext.addTag`

```typescript
addTag(tag: string): this
```

Append a tag to the trace. Tags can also be set at trace open time via `options.tags`.

**Returns** `this` for chaining.

---

### `TraceContext.data`

```typescript
readonly data: TraceData
```

The raw trace data object. Use `data.id` to capture the trace ID for feedback or correlation. All fields are live — they reflect the current state and are finalized when the trace callback exits.

**Example**

```typescript
await tl.trace('my-trace', async (t) => {
  const traceId = t.data.id  // UUIDv7, available immediately
  // ...
})
```

---

## SpanContext

```typescript
import { SpanContext } from '@trulayer/sdk'
```

The context object passed to each `span()` callback. Provides methods to annotate the span. Latency is measured automatically.

---

### `SpanContext.span`

```typescript
async span<T>(
  name: string,
  spanType: SpanType,
  callback: (span: SpanContext) => Promise<T>,
): Promise<T>
```

Open a nested child span under this span. The child's `parent_span_id` is set to `this.data.id`.

**Parameters** Same as `TraceContext.span`.

**Example**

```typescript
await t.span('outer', 'chain', async (outer) => {
  await outer.span('inner-llm', 'llm', async (inner) => {
    inner.setInput(prompt)
    // inner.data.parent_span_id === outer.data.id
  })
})
```

---

### `SpanContext.setInput`

```typescript
setInput(value: string): this
```

Set the input for this span. If a `redact` function is configured, it is applied before storing.

**Returns** `this` for chaining.

---

### `SpanContext.setOutput`

```typescript
setOutput(value: string): this
```

Set the output for this span. If a `redact` function is configured, it is applied before storing.

**Returns** `this` for chaining.

---

### `SpanContext.setModel`

```typescript
setModel(model: string): this
```

Set the model name for this span (shown on LLM spans in the dashboard).

**Returns** `this` for chaining.

---

### `SpanContext.setTokens`

```typescript
setTokens(prompt?: number, completion?: number): this
```

Record token counts for this span. Pass `undefined` to leave either count unset.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `prompt` | `number \| undefined` | Prompt / input token count |
| `completion` | `number \| undefined` | Completion / output token count |

**Returns** `this` for chaining.

**Example**

```typescript
if (resp.usage) {
  s.setTokens(resp.usage.prompt_tokens, resp.usage.completion_tokens)
}
```

---

### `SpanContext.setMetadata`

```typescript
setMetadata(meta: Record<string, unknown>): this
```

Merge additional key-value pairs into the span's `metadata`. Subsequent calls merge — they do not replace.

**Returns** `this` for chaining.

---

### `SpanContext.data`

```typescript
readonly data: SpanData
```

The raw span data object. `data.id` is a UUIDv7 assigned at span creation.

---

## NoopTraceContext and NoopSpanContext

```typescript
import { NoopTraceContext, NoopSpanContext } from '@trulayer/sdk'
```

Silent no-op implementations returned when a trace is sampled out (`sampleRate < 1.0` and the trace was not selected). All methods are no-ops; nothing is enqueued or sent. The user callback still executes normally.

Both classes expose a `data` property with the same shape as `TraceData` / `SpanData` (but with empty/null values) so that code that reads `t.data.id` does not crash.

You generally do not need to use these classes directly — they are the type of the argument when `client.trace()` samples out, but the callback interface is identical.

```typescript
await tl.trace('my-trace', async (t) => {
  // t is TraceContext | NoopTraceContext
  // All method calls are safe regardless of which it is
  t.setInput(question)
  await t.span('step', 'llm', async (s) => {
    s.setOutput(answer)
  })
})
```

---

## Auto-instrumentation

```typescript
import {
  instrumentOpenAI,
  instrumentAnthropic,
  instrumentVercelAI,
  TruLayerCallbackHandler,
} from '@trulayer/sdk'
```

---

### `instrumentOpenAI`

```typescript
function instrumentOpenAI<T extends OpenAIClient>(client: T, trace: TraceContext): T
```

Return a new instrumented proxy of the OpenAI client. Every `chat.completions.create` call through the proxy emits an `openai.chat` span with `span_type="llm"` into `trace`. The original client is never mutated.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `client` | `T extends OpenAIClient` | An OpenAI client instance (or any structurally compatible object) |
| `trace` | `TraceContext` | The active trace context to emit spans into |

**Returns** A new proxy of the same type `T`.

The proxy intercepts `client.chat.completions.create`. For each call it:
- Sets `span.input` to the content of the last message
- Sets `span.model` from `params.model`
- Sets `span.output` and `span.tokens` from the response (or accumulated stream)

Streaming responses (`stream: true`) are handled transparently. The span stays open until the async iterator is exhausted.

**Example**

```typescript
import OpenAI from 'openai'
import { TruLayer, instrumentOpenAI } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })
const openai = new OpenAI()

await tl.trace('my-trace', async (t) => {
  // Cast to preserve full OpenAI types on the returned proxy
  const wrap = instrumentOpenAI as unknown as <T>(client: T, trace: unknown) => T
  const wrapped: OpenAI = wrap(openai, t)

  const resp = await wrapped.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: 'Hello' }],
  })
  t.setOutput(resp.choices[0]?.message?.content ?? '')
})
```

---

### `instrumentAnthropic`

```typescript
function instrumentAnthropic<T extends AnthropicClient>(client: T, trace: TraceContext): T
```

Return a new instrumented proxy of an Anthropic client. Every `messages.create` call emits an `anthropic.messages` span with `span_type="llm"`. The original client is never mutated.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `client` | `T extends AnthropicClient` | An Anthropic client instance |
| `trace` | `TraceContext` | The active trace context |

**Returns** A new proxy of the same type `T`.

The proxy intercepts `client.messages.create`. For each call it:
- Sets `span.input` to the content of the last message
- Sets `span.model` from `params.model`
- Sets `span.output` from the first `text` content block
- Sets `span.tokens` from `usage.input_tokens` and `usage.output_tokens`

Streaming responses (`stream: true`) are handled transparently. Content is accumulated from `content_block_delta` events; tokens from `message_start` and `message_delta` events.

**Example**

```typescript
import Anthropic from '@anthropic-ai/sdk'
import { TruLayer, instrumentAnthropic } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })
const anthropic = new Anthropic()

await tl.trace('my-trace', async (t) => {
  const wrap = instrumentAnthropic as unknown as <T>(client: T, trace: unknown) => T
  const wrapped: Anthropic = wrap(anthropic, t)

  const resp = await wrapped.messages.create({
    model: 'claude-3-5-sonnet-latest',
    max_tokens: 128,
    messages: [{ role: 'user', content: 'What is the Eiffel Tower?' }],
  })
  const text = resp.content.find((b) => b.type === 'text')?.text ?? ''
  t.setOutput(text)
})
```

---

### `instrumentVercelAI`

```typescript
function instrumentVercelAI<G, S, O>(
  fns: { generateText?: G; streamText?: S; generateObject?: O },
  trace: TraceContext,
): { generateText: G; streamText: S; generateObject: O }
```

Return instrumented wrappers for Vercel AI SDK's `generateText`, `streamText`, and `generateObject`. Pass only the functions you use; unwrapped functions are returned as-is.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `fns.generateText` | `AIFunction` | Vercel AI SDK `generateText` function |
| `fns.streamText` | `AIFunction` | Vercel AI SDK `streamText` function |
| `fns.generateObject` | `AIFunction` | Vercel AI SDK `generateObject` function |
| `trace` | `TraceContext` | The active trace context |

**Returns** An object `{ generateText, streamText, generateObject }` with the same types as the inputs.

Each wrapped function emits an LLM span named after the function (`vercel-ai.generateText`, `vercel-ai.streamText`, `vercel-ai.generateObject`). Input is extracted from `params.prompt` or the last message in `params.messages`. Model is extracted from `params.model.modelId`.

For `streamText`, the span closes when `result.text` resolves (after the stream is consumed).

**Example**

```typescript
import { generateText, streamText } from 'ai'
import { openai } from '@ai-sdk/openai'
import { TruLayer, instrumentVercelAI } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })

await tl.trace('vercel-ai-trace', async (t) => {
  const { generateText: tracedGenerate, streamText: tracedStream } = instrumentVercelAI(
    { generateText, streamText },
    t as any,
  )

  // generateText — span closes when the promise resolves
  const result = await tracedGenerate({
    model: openai('gpt-4o-mini'),
    prompt: 'Name one landmark in Paris.',
  })
  console.log(result.text)

  // streamText — span closes when result.text resolves
  const streamResult = tracedStream({
    model: openai('gpt-4o-mini'),
    prompt: 'List three cities in France.',
  })
  for await (const chunk of streamResult.textStream) {
    process.stdout.write(chunk)
  }
  t.setOutput(await streamResult.text)
})
```

---

### `TruLayerCallbackHandler`

```typescript
import { TruLayerCallbackHandler } from '@trulayer/sdk'

class TruLayerCallbackHandler {
  constructor(trace: TraceContext)
  readonly name: 'TruLayerCallbackHandler'
}
```

A LangChain.js-compatible callback handler that creates TruLayer spans for chain steps. Works via structural typing — no runtime dependency on `@langchain/core`. LangChain accepts any object with the right method signatures as a callback handler.

**Constructor**

| Parameter | Type | Description |
|---|---|---|
| `trace` | `TraceContext` | The active trace context to emit spans into |

The handler responds to these LangChain callbacks:

| Callback | Span name | Span type |
|---|---|---|
| `handleLLMStart` | LLM name (or `"llm"`) | `"llm"` |
| `handleChainStart` | Chain name (or `"chain"`) | `"default"` |
| `handleToolStart` | Tool name (or `"tool"`) | `"tool"` |

Each span closes when the corresponding `End` or `Error` callback fires.

**Example**

```typescript
import { ChatOpenAI } from '@langchain/openai'
import { ChatPromptTemplate } from '@langchain/core/prompts'
import { StringOutputParser } from '@langchain/core/output_parsers'
import { TruLayer, TruLayerCallbackHandler } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })

const chain = ChatPromptTemplate.fromMessages([
  ['system', 'You are a concise assistant.'],
  ['human', '{question}'],
])
  .pipe(new ChatOpenAI({ model: 'gpt-4o-mini' }))
  .pipe(new StringOutputParser())

await tl.trace('langchain-qa', async (t) => {
  t.setInput(question)
  const handler = new TruLayerCallbackHandler(t as any)
  const answer = await chain.invoke({ question }, { callbacks: [handler] })
  t.setOutput(String(answer).trim())
})

await tl.shutdown()
```

---

## Redaction

```typescript
import { Redactor, redact, BUILTIN_PACKS } from '@trulayer/sdk'
import type { Rule, RedactorOptions, PackName } from '@trulayer/sdk'
```

---

### `Redactor`

```typescript
class Redactor {
  constructor(opts?: RedactorOptions)
  redact(text: string): string
  redactSpan<T extends Record<string, unknown>>(
    span: T,
    fields?: readonly string[],
  ): T
}
```

Compose built-in entity packs and custom `Rule`s into a reusable scrubber. Regex patterns are compiled once at construction time.

**Constructor — `RedactorOptions`**

| Option | Type | Default | Description |
|---|---|---|---|
| `packs` | `PackName[]` | `[]` | Built-in entity packs to enable |
| `rules` | `Rule[]` | `[]` | Custom redaction rules |
| `pseudonymize` | `boolean` | `false` | Replace matches with deterministic HMAC tokens instead of static placeholders |
| `pseudonymizeSalt` | `string` | — | Required when any rule pseudonymizes. Used as HMAC key |
| `pseudonymLength` | `number` | `8` | Hex characters of the HMAC digest to include in the token (4–64) |

**Throws** `Error` if `pseudonymize: true` but `pseudonymizeSalt` is not provided, or if `pseudonymLength` is outside [4, 64].

#### `redact(text)`

```typescript
redact(text: string): string
```

Apply all compiled rules to `text` and return the scrubbed string. Operates synchronously; safe in all runtimes.

**Example**

```typescript
const redactor = new Redactor({ packs: ['standard'] })
redactor.redact('Contact admin@example.com for help')
// -> 'Contact <REDACTED:email> for help'
```

#### `redactSpan(span, fields?)`

```typescript
redactSpan<T extends Record<string, unknown>>(
  span: T,
  fields?: readonly string[],
): T
```

Return a shallow copy of `span` with the specified fields scrubbed. Uses dot-path notation to target nested fields.

**Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `span` | `T` | — | The span-like object to scrub |
| `fields` | `readonly string[]` | `['input', 'output', 'metadata']` | Dot-path field names to scrub |

**Example**

```typescript
const clean = redactor.redactSpan(spanData, ['input', 'output', 'metadata.user.email'])
```

---

### `redact` (function)

```typescript
function redact(text: string, opts?: RedactorOptions): string
```

One-shot convenience wrapper. Constructs a `Redactor` with the `standard` pack (plus any `opts`) and applies it to `text`. For repeated use, construct a `Redactor` directly to amortize the regex compilation cost.

**Example**

```typescript
import { redact } from '@trulayer/sdk'

const clean = redact('My SSN is 123-45-6789')
// -> 'My SSN is <REDACTED:ssn>'
```

---

### `BUILTIN_PACKS`

```typescript
const BUILTIN_PACKS: Record<PackName, PackEntry[]>
```

The full registry of built-in redaction patterns, exported for inspection or extension.

| Pack | Entities |
|---|---|
| `"standard"` | `email`, `ssn`, `jwt`, `bearer_token`, `phone` |
| `"strict"` | All of `standard` + `credit_card` (Luhn-validated), `iban`, `ipv4` |
| `"phi"` | `mrn` (medical record number), `icd10`, `dob` (date of birth) |
| `"finance"` | `swift_bic`, `routing_number`, `account_number`, `ticker_amount` |
| `"secrets"` | `aws_access_key`, `github_pat`, `pem_private_key`, `gcp_service_account` |

---

### `Rule`

```typescript
interface Rule {
  name: string
  pattern: string | RegExp
  replacement?: string
  pseudonymize?: boolean
  validator?: (match: string) => boolean
}
```

Defines a custom redaction rule.

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Identifier used in the default replacement token `<REDACTED:{name}>` |
| `pattern` | `string \| RegExp` | Regex string or `RegExp`. String patterns are compiled with the `g` flag. RegExp patterns are forced global |
| `replacement` | `string` | Custom replacement string. Overrides the default `<REDACTED:{name}>` |
| `pseudonymize` | `boolean` | Override the `Redactor`-level `pseudonymize` setting for this rule |
| `validator` | `(match: string) => boolean` | Called for each regex match; the match is only redacted if this returns `true` |

**Example**

```typescript
import { Redactor } from '@trulayer/sdk'

const redactor = new Redactor({
  rules: [
    {
      name: 'internal_id',
      pattern: /INT-\d{8}/g,
      replacement: '<INTERNAL_ID>',
    },
    {
      name: 'credit_card',
      pattern: /\b(?:\d[ -]?){13,19}\b/g,
      validator: (match) => luhnCheck(match),  // only redact valid card numbers
    },
  ],
})
```

---

### `RedactorOptions`

```typescript
interface RedactorOptions {
  packs?: PackName[]
  rules?: Rule[]
  pseudonymize?: boolean
  pseudonymizeSalt?: string
  pseudonymLength?: number
}
```

See [Redactor constructor](#redactor) for field descriptions.

---

## Data models

```typescript
import type { TruLayerConfig, TraceData, SpanData, FeedbackData, SpanType } from '@trulayer/sdk'
```

---

### `TruLayerConfig`

```typescript
interface TruLayerConfig {
  apiKey: string
  projectName?: string
  /** @deprecated Use projectName. Removed in 0.3.x */
  projectId?: string
  endpoint?: string
  batchSize?: number
  flushInterval?: number    // ms
  sampleRate?: number       // 0.0–1.0
  redact?: (data: unknown) => unknown
  relayUrl?: string
}
```

---

### `TraceData`

```typescript
interface TraceData {
  id: string                       // UUIDv7
  project_id: string
  session_id: string | null
  external_id: string | null
  name: string | null
  input: string | null
  output: string | null
  model: string | null
  latency_ms: number | null
  cost: number | null
  error: boolean
  tags: string[]
  metadata: Record<string, unknown>
  spans: SpanData[]
  started_at: string               // ISO 8601
  ended_at: string | null          // ISO 8601
}
```

---

### `SpanData`

```typescript
interface SpanData {
  id: string                       // UUIDv7
  trace_id: string
  parent_span_id?: string          // set when span is nested
  name: string
  span_type: SpanType
  input: string | null
  output: string | null
  error: boolean
  error_message: string | null
  latency_ms: number | null
  model: string | null
  prompt_tokens: number | null
  completion_tokens: number | null
  metadata: Record<string, unknown>
  started_at: string               // ISO 8601
  ended_at: string | null          // ISO 8601
}
```

---

### `FeedbackData`

```typescript
interface FeedbackData {
  trace_id: string
  label: string
  score?: number
  comment?: string
  metadata?: Record<string, unknown>
}
```

---

### `SpanType`

```typescript
type SpanType = 'llm' | 'tool' | 'retrieval' | 'chain' | 'default'
```

Controls the icon and grouping in the TruLayer dashboard.

| Value | Use for |
|---|---|
| `"llm"` | LLM completion calls |
| `"tool"` | Function / tool calls in an agent |
| `"retrieval"` | Vector store / document retrieval |
| `"chain"` | Multi-step pipelines or sequential steps |
| `"default"` | Everything else |

---

## Testing helpers

```typescript
import { createTestClient, assertSender, SenderAssertions, LocalBatchSender } from '@trulayer/sdk'
```

---

### `createTestClient`

```typescript
function createTestClient(config?: Partial<TruLayerConfig>): {
  client: TruLayer
  sender: LocalBatchSender
}
```

Create a `TruLayer` client wired to an in-memory `LocalBatchSender`. No API key is required. No network calls are made.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `config` | `Partial<TruLayerConfig>` | Optional overrides. Defaults: `apiKey: 'test-key'`, `projectName: 'test'` |

**Returns** `{ client, sender }` where `sender` is the `LocalBatchSender` instance you can use for assertions.

**Example**

```typescript
import { createTestClient, assertSender } from '@trulayer/sdk'
import { describe, it, expect } from 'vitest'

describe('my pipeline', () => {
  it('records a trace with spans', async () => {
    const { client, sender } = createTestClient()

    await client.trace('pipeline', async (t) => {
      t.setInput('hello')
      await t.span('step-1', 'llm', async (s) => {
        s.setInput('prompt')
        s.setOutput('response')
        s.setModel('gpt-4o-mini')
        s.setTokens(10, 20)
      })
      t.setOutput('done')
    })

    await client.shutdown()

    assertSender(sender)
      .hasTrace()
      .spanCount(1)
      .hasSpanNamed('step-1')

    expect(sender.traces[0].name).toBe('pipeline')
    expect(sender.spans[0].model).toBe('gpt-4o-mini')
    expect(sender.spans[0].prompt_tokens).toBe(10)
  })
})
```

---

### `assertSender`

```typescript
function assertSender(sender: LocalBatchSender): SenderAssertions
```

Create a fluent `SenderAssertions` instance for the given sender.

**Example**

```typescript
assertSender(sender).hasTrace().spanCount(2).hasSpanNamed('retrieve-docs')
```

---

### `SenderAssertions`

```typescript
class SenderAssertions {
  constructor(sender: LocalBatchSender)
  hasTrace(traceId?: string): this
  spanCount(n: number): this
  hasSpanNamed(name: string): this
}
```

Chainable assertion helper. All methods throw `Error` if the assertion fails.

| Method | Asserts |
|---|---|
| `hasTrace(traceId?)` | At least one trace exists, or a specific trace by ID |
| `spanCount(n)` | Exactly `n` spans across all traces |
| `hasSpanNamed(name)` | At least one span has the given name |

---

### `LocalBatchSender`

```typescript
class LocalBatchSender {
  enqueue(trace: TraceData): void
  flush(): void
  shutdown(): Promise<void>

  readonly traces: TraceData[]
  readonly spans: SpanData[]
  readonly batches: readonly CapturedBatch[]

  clear(): void
}
```

In-memory batch sender for local/offline mode and testing. Never makes network calls. All enqueued traces are stored synchronously.

| Property / Method | Description |
|---|---|
| `traces` | All captured `TraceData` objects across all batches (flat) |
| `spans` | All captured `SpanData` objects across all batches (flat) |
| `batches` | All `CapturedBatch` records in insertion order |
| `clear()` | Reset all captured state. Call between test cases |

Set `TRULAYER_LOCAL_VERBOSE=1` to emit a log line for each enqueued trace.

---

### `CapturedBatch`

```typescript
interface CapturedBatch {
  traces: TraceData[]
  sentAt: Date
}
```

A single batch as captured by `LocalBatchSender`.

---

## Browser entry point

```typescript
import { initBrowser } from '@trulayer/sdk/browser'
```

The browser entry point routes trace events through a server-side relay instead of calling the TruLayer API directly. This avoids CORS issues and keeps API keys out of client-side bundles.

---

### `initBrowser`

```typescript
function initBrowser(config: BrowserConfig): TruLayer
```

Initialize the SDK for browser environments. Requires `relayUrl`.

**Parameters** Same as `TruLayerConfig` plus the required `relayUrl` field.

**Throws** `Error` if `relayUrl` is missing.

**Example**

```typescript
// Browser / React code
import { initBrowser } from '@trulayer/sdk/browser'

const tl = initBrowser({
  apiKey: '',                   // not used in browser mode
  projectName: 'my-app',
  relayUrl: '/api/trulayer',    // path to your server-side relay
})

await tl.trace('user-search', async (t) => {
  t.setInput(searchQuery)
  // ...
  t.setOutput(results)
})
```

The browser sender POSTs batches to `relayUrl` without an `Authorization` header. Your relay route is responsible for adding the header before proxying to TruLayer:

```typescript
// app/api/trulayer/route.ts (Next.js App Router)
export async function POST(req: Request) {
  const body = await req.json()
  const res = await fetch('https://api.trulayer.ai/v1/batch', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${process.env.TRULAYER_API_KEY}`,
    },
    body: JSON.stringify(body),
  })
  return new Response(await res.text(), { status: res.status })
}
```

---

### `BrowserConfig`

```typescript
type BrowserConfig = TruLayerConfig & { relayUrl: string }
```

Same as `TruLayerConfig` but `relayUrl` is required.

---

## Environment variables

| Variable | Description |
|---|---|
| `TRULAYER_MODE=local` | Run in local/offline mode. No API key required. Traces are stored in memory and never sent |
| `TRULAYER_LOCAL_VERBOSE=1` | When in local mode, log a line for each enqueued trace |
