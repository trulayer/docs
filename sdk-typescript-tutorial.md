# TruLayer AI — TypeScript SDK Tutorial

A task-oriented guide to instrumenting TypeScript AI applications with TruLayer.

## Table of Contents

1. [Installation](#installation)
2. [Client initialization](#client-initialization)
3. [Creating a trace](#creating-a-trace)
4. [Adding spans](#adding-spans)
5. [Auto-instrumentation](#auto-instrumentation)
   - [OpenAI](#openai)
   - [Anthropic](#anthropic)
   - [Vercel AI SDK](#vercel-ai-sdk)
   - [LangChain.js](#langchainjs)
6. [Streaming spans](#streaming-spans)
7. [Adding feedback](#adding-feedback)
8. [Redacting sensitive data](#redacting-sensitive-data)
9. [Local / offline mode](#local--offline-mode)
10. [Browser usage](#browser-usage)
11. [Testing](#testing)
12. [Complete end-to-end example](#complete-end-to-end-example)

---

## Installation

```bash
npm install @trulayer/sdk
```

```bash
# or with pnpm / yarn
pnpm add @trulayer/sdk
yarn add @trulayer/sdk
```

Node.js 18 or later is required. The SDK also works in Edge runtimes (Vercel Edge Functions, Cloudflare Workers) and Bun without any additional configuration.

---

## Client initialization

### Using the global client (`init`)

Call `init()` once at application startup. Every subsequent call to `getClient()` returns the same instance.

```typescript
import { init } from '@trulayer/sdk'

init({
  apiKey: 'tl_...',          // required; or set TRULAYER_API_KEY env var
  projectName: 'my-project', // required
})
```

`init()` accepts the full set of constructor parameters:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | — | TruLayer API key |
| `projectName` | `string` | — | Project name shown in the dashboard |
| `endpoint` | `string` | `"https://api.trulayer.ai"` | Override the ingestion endpoint |
| `batchSize` | `number` | `50` | Max events per HTTP batch |
| `flushInterval` | `number` | `2000` | Milliseconds between automatic flushes |
| `sampleRate` | `number` | `1.0` | Fraction of traces to send (0.0–1.0) |
| `redact` | `(data: unknown) => unknown` | — | Applied to all string fields before sending |

`init()` returns the `TruLayer` client instance if you need a reference.

### Using an explicit client

If you prefer not to use global state, construct `TruLayer` directly:

```typescript
import { TruLayer } from '@trulayer/sdk'

const tl = new TruLayer({
  apiKey: 'tl_...',
  projectName: 'my-project',
})
```

The constructor accepts the same parameters as `init()`.

### Retrieving the global client

```typescript
import { getClient } from '@trulayer/sdk'

const tl = getClient() // throws if init() was not called first
```

---

## Creating a trace

A trace represents one logical operation — one request, one job, one conversation turn. Open a trace with `client.trace()`. The trace is automatically timed and enqueued when the callback resolves.

```typescript
import { TruLayer } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })

await tl.trace(
  'basic-qa',
  async (t) => {
    t.setInput('What is the capital of France?')
    // ... do work ...
    t.setOutput('Paris.')
    t.setModel('gpt-4o-mini')
    t.setCost(0.00012)  // optional, in USD
  },
  {
    externalId: 'req-42',            // your own request ID for deduplication
    tags: ['production', 'rag'],
    metadata: { userId: 'u_123' },
  },
)
```

You can read the auto-generated trace ID from `t.data.id` inside the callback:

```typescript
await tl.trace('my-trace', async (t) => {
  const traceId = t.data.id  // UUIDv7, available immediately
  // ...
})
```

**Trace callback methods at a glance:**

| Method | Description |
|---|---|
| `t.setInput(text)` | Top-level input shown in the dashboard |
| `t.setOutput(text)` | Top-level output |
| `t.setModel(model)` | Model name rolled up to the trace |
| `t.setCost(usd)` | Cost in USD |
| `t.setMetadata(obj)` | Merge key-value pairs into trace metadata |
| `t.addTag(tag)` | Append a tag |

If an exception propagates out of the trace callback, `error: true` is set on the trace automatically and the error is re-thrown.

---

## Adding spans

Spans represent individual steps within a trace (retrieval, LLM call, tool use). Open a span with `trace.span()`:

```typescript
await tl.trace('rag-pipeline', async (t) => {
  t.setInput(question)

  // Retrieval step
  const docs = await t.span('retrieve-context', 'retrieval', async (s) => {
    s.setInput(question)
    const results = await vectorStore.search(question)
    s.setOutput(JSON.stringify(results))
    s.setMetadata({ source: 'pinecone', resultCount: results.length })
    return results
  })

  // LLM step
  const answer = await t.span('openai.chat', 'llm', async (s) => {
    s.setModel('gpt-4o-mini')
    s.setInput(prompt)
    const resp = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: prompt }],
    })
    const text = resp.choices[0]?.message?.content ?? ''
    s.setOutput(text)
    if (resp.usage) {
      s.setTokens(resp.usage.prompt_tokens, resp.usage.completion_tokens)
    }
    return text
  })

  t.setOutput(answer)
})
```

The `spanType` parameter (second argument to `span()`) controls how the span appears in the dashboard:

| Value | Use for |
|---|---|
| `"llm"` | LLM completion calls |
| `"retrieval"` | Vector store / document retrieval |
| `"tool"` | Function / tool calls in an agent |
| `"chain"` | Multi-step pipelines |
| `"default"` | Everything else |

Latency is measured automatically. If an exception propagates out of a span's callback, `error: true` and `error_message` are set automatically on the span.

### Nested spans

Spans can be nested by calling `span()` on a `SpanContext`:

```typescript
await tl.trace('pipeline', async (t) => {
  await t.span('outer', 'chain', async (outer) => {
    await outer.span('inner', 'llm', async (inner) => {
      inner.setInput('nested call')
      // inner.parent_span_id === outer.data.id
    })
  })
})
```

**Span callback methods at a glance:**

| Method | Description |
|---|---|
| `s.setInput(text)` | Input to this step |
| `s.setOutput(text)` | Output from this step |
| `s.setModel(model)` | Model name for this span |
| `s.setTokens(prompt?, completion?)` | Token counts (both are `number \| undefined`) |
| `s.setMetadata(obj)` | Merge key-value pairs into span metadata |

---

## Auto-instrumentation

Auto-instrumentation wraps a provider's client so every call inside an active trace automatically produces a span. You still define the trace boundary; TruLayer fills in the provider spans.

### OpenAI

`instrumentOpenAI(client, trace)` returns a new instrumented proxy — the original client is never mutated. Every `chat.completions.create` call through the proxy emits an `openai.chat` span with `span_type="llm"`.

```typescript
import OpenAI from 'openai'
import { TruLayer, instrumentOpenAI } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })
const openai = new OpenAI()

await tl.trace('travel-qa', async (t) => {
  t.setInput('Name one landmark in Paris.')

  // Cast at the boundary to preserve full OpenAI types on the proxy
  const wrap = instrumentOpenAI as unknown as <T>(client: T, trace: unknown) => T
  const wrapped: OpenAI = wrap(openai, t)

  const resp = await wrapped.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [{ role: 'user', content: 'Name one landmark in Paris.' }],
  })
  t.setOutput(resp.choices[0]?.message?.content?.trim() ?? '')
}, { tags: ['openai-auto'] })
```

Streaming responses are handled transparently: the span is opened when the request starts and closed after the stream is fully consumed.

### Anthropic

`instrumentAnthropic(client, trace)` wraps an Anthropic client. Every `messages.create` call emits an `anthropic.messages` span.

```typescript
import Anthropic from '@anthropic-ai/sdk'
import { TruLayer, instrumentAnthropic } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })
const anthropic = new Anthropic()

await tl.trace('landmark-qa', async (t) => {
  t.setInput(question)

  const wrap = instrumentAnthropic as unknown as <T>(client: T, trace: unknown) => T
  const wrapped: Anthropic = wrap(anthropic, t)

  const resp = await wrapped.messages.create({
    model: 'claude-3-5-sonnet-latest',
    max_tokens: 128,
    messages: [{ role: 'user', content: question }],
  })
  const text = resp.content.find((b) => b.type === 'text')?.text ?? ''
  t.setOutput(text.trim())
}, { tags: ['anthropic-auto'] })
```

Streaming is supported transparently for both `instrumentOpenAI` and `instrumentAnthropic`.

### Vercel AI SDK

`instrumentVercelAI(fns, trace)` wraps Vercel AI SDK functions (`generateText`, `streamText`, `generateObject`). Pass only the functions you use; each returns an instrumented wrapper.

```typescript
import { generateText, streamText, generateObject } from 'ai'
import { openai } from '@ai-sdk/openai'
import { TruLayer, instrumentVercelAI } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })

await tl.trace('vercel-ai-demo', async (t) => {
  const { generateText: tracedGenerate } = instrumentVercelAI(
    { generateText },
    t as any,  // narrow TraceContext type — safe at runtime
  )

  const result = await tracedGenerate({
    model: openai('gpt-4o-mini'),
    prompt: 'What is the capital of France?',
  })

  t.setOutput(result.text)
}, { tags: ['vercel-ai'] })
```

For `streamText`, the span closes automatically when the text promise resolves:

```typescript
await tl.trace('stream-demo', async (t) => {
  const { streamText: tracedStream } = instrumentVercelAI({ streamText }, t as any)

  const result = tracedStream({
    model: openai('gpt-4o-mini'),
    prompt: 'List three landmarks in Paris, one per line.',
  })

  const chunks: string[] = []
  for await (const chunk of result.textStream) {
    chunks.push(chunk)
  }
  t.setOutput(chunks.join(''))
})
```

### LangChain.js

The SDK exports `TruLayerCallbackHandler` — a LangChain-compatible callback handler. Attach it to any chain, LLM, or chat model via the `callbacks` option.

```typescript
import { ChatOpenAI } from '@langchain/openai'
import { ChatPromptTemplate } from '@langchain/core/prompts'
import { StringOutputParser } from '@langchain/core/output_parsers'
import { TruLayer, TruLayerCallbackHandler } from '@trulayer/sdk'

const tl = new TruLayer({ apiKey: 'tl_...', projectName: 'my-project' })

const chain = ChatPromptTemplate.fromMessages([
  ['system', 'You are a concise travel guide. One sentence only.'],
  ['human', '{question}'],
])
  .pipe(new ChatOpenAI({ model: 'gpt-4o-mini' }))
  .pipe(new StringOutputParser())

await tl.trace('langchain-qa', async (t) => {
  t.setInput(question)
  const handler = new TruLayerCallbackHandler(t as any)
  const answer = await chain.invoke({ question }, { callbacks: [handler] })
  t.setOutput(String(answer).trim())
}, { tags: ['langchain'] })

await tl.shutdown()
```

The handler creates an LLM span for each chat model or LLM invocation inside the chain. Tool calls and chain steps each produce their own spans.

---

## Streaming spans

When you stream responses manually (without auto-instrumentation), accumulate the chunks inside the span callback and call `setOutput()` before the callback returns:

```typescript
await tl.trace('streaming-qa', async (t) => {
  t.setInput(question)

  const answer = await t.span('openai.chat', 'llm', async (s) => {
    s.setModel('gpt-4o-mini')
    s.setInput(question)

    const stream = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [{ role: 'user', content: question }],
      stream: true,
      stream_options: { include_usage: true },
    })

    const chunks: string[] = []
    let promptTokens: number | undefined
    let completionTokens: number | undefined

    for await (const event of stream) {
      const delta = event.choices[0]?.delta?.content
      if (delta) chunks.push(delta)
      if (event.usage) {
        promptTokens = event.usage.prompt_tokens
        completionTokens = event.usage.completion_tokens
      }
    }

    const text = chunks.join('').trim()
    s.setOutput(text)
    if (promptTokens !== undefined || completionTokens !== undefined) {
      s.setTokens(promptTokens, completionTokens)
    }
    return text
  })

  t.setOutput(answer)
}, { tags: ['streaming'] })
```

The span stays open for the entire duration of the stream, so latency reflects actual streaming time.

---

## Adding feedback

Feedback attaches a human or automated label to a trace after it has been ingested. Call `client.feedback()` with the trace ID captured during the trace:

```typescript
let traceId: string = ''

await tl.trace('feedback-demo', async (t) => {
  t.setInput(question)
  // ... do work ...
  t.setOutput(answer)
  traceId = t.data.id
})

// Ensure the trace is ingested before sending feedback.
await tl.shutdown()

tl.feedback(traceId, 'good', {
  score: 1.0,
  comment: 'Accurate and concise.',
  metadata: { source: 'user-thumbsup', reviewer: 'alice' },
})
```

`feedback()` is fire-and-forget — it never throws into your code. Failures are emitted as `console.warn`.

**Feedback labels** are arbitrary strings; conventional values are `"good"`, `"bad"`, and `"neutral"`.

---

## Redacting sensitive data

Pass a `redact` function to the client constructor. It is called on every string field (`input`, `output`) across all traces and spans before they are enqueued.

```typescript
import { TruLayer } from '@trulayer/sdk'

const tl = new TruLayer({
  apiKey: 'tl_...',
  projectName: 'my-project',
  redact: (value) => {
    if (typeof value !== 'string') return value
    return value.replace(/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/g, '<EMAIL>')
  },
})
```

If the `redact` callback throws, the field is stored as `null` and a warning is emitted.

### Structured redaction with `Redactor`

For more powerful redaction, use the built-in `Redactor` class and pass it through the `redact` option:

```typescript
import { TruLayer, Redactor } from '@trulayer/sdk'

const redactor = new Redactor({
  packs: ['standard', 'secrets'],   // enable built-in entity packs
  rules: [
    { name: 'employee_id', pattern: /EMP-\d{6}/g },
  ],
})

const tl = new TruLayer({
  apiKey: 'tl_...',
  projectName: 'my-project',
  redact: (value) => (typeof value === 'string' ? redactor.redact(value) : value),
})
```

You can also use `Redactor` standalone:

```typescript
import { Redactor } from '@trulayer/sdk'

const redactor = new Redactor({ packs: ['standard'] })

const clean = redactor.redact('Contact foo@bar.com, EMP-123456')
// -> 'Contact <REDACTED:email>, EMP-123456'

// Target specific fields in a span dict
const cleanSpan = redactor.redactSpan(spanDict, ['input', 'output', 'metadata.user.email'])
```

Built-in packs:

| Pack | Entities covered |
|---|---|
| `"standard"` | email, SSN, JWT, bearer token, phone |
| `"strict"` | everything in `standard` + credit card (Luhn-validated), IBAN, IPv4 |
| `"phi"` | medical record number, ICD-10 code, date of birth |
| `"finance"` | SWIFT/BIC, routing number, account number, ticker/amount |
| `"secrets"` | AWS access key, GitHub PAT, PEM private key, GCP service account JSON |

Pseudonymization replaces matched values with a deterministic HMAC-SHA256 token instead of a static placeholder:

```typescript
const redactor = new Redactor({
  packs: ['standard'],
  pseudonymize: true,
  pseudonymizeSalt: 'my-secret-salt',
  pseudonymLength: 8,  // hex chars of HMAC digest (4–64)
})

redactor.redact('user foo@bar.com')
// -> 'user <PSEUDO:a3f2b1c9>'
```

---

## Local / offline mode

Set the environment variable `TRULAYER_MODE=local` to run without an API key or network access. Traces are stored in memory and never sent.

```bash
TRULAYER_MODE=local node my_app.js
```

In local mode the SDK emits a warning and creates a `LocalBatchSender` internally. You can also pass a `LocalBatchSender` directly for testing — see [Testing](#testing).

---

## Browser usage

The SDK includes a browser entry point at `@trulayer/sdk/browser` that routes trace events through a server-side relay instead of calling the TruLayer API directly. This keeps the API key out of client-side bundles.

```typescript
// In your browser/React code
import { initBrowser } from '@trulayer/sdk/browser'

const tl = initBrowser({
  apiKey: '',            // not used in browser mode
  projectName: 'my-app',
  relayUrl: '/api/trulayer',  // server-side relay route
})

await tl.trace('user-action', async (t) => {
  t.setInput(userQuery)
  // ...
  t.setOutput(response)
})
```

The relay route (e.g. a Next.js API route) is responsible for attaching the `Authorization` header before proxying to TruLayer:

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

## Testing

Use `createTestClient()` to get an in-memory client with no network I/O. Then use `assertSender()` for fluent assertions.

```typescript
import { createTestClient, assertSender } from '@trulayer/sdk'

describe('my pipeline', () => {
  it('records spans', async () => {
    const { client, sender } = createTestClient({ projectName: 'test-project' })

    await client.trace('my-pipeline', async (t) => {
      await t.span('step', 'llm', async (s) => {
        s.setInput('hello')
        s.setOutput('world')
      })
    })

    await client.shutdown()

    assertSender(sender).hasTrace().spanCount(1).hasSpanNamed('step')

    // Direct access to captured data
    expect(sender.traces[0].name).toBe('my-pipeline')
    expect(sender.spans[0].input).toBe('hello')
  })
})
```

`createTestClient()` accepts all `TruLayerConfig` options except `apiKey` (not needed for testing).

`SenderAssertions` methods are chainable:

| Method | Asserts |
|---|---|
| `.hasTrace(traceId?)` | At least one trace, or a specific trace by ID |
| `.spanCount(n)` | Exactly `n` spans across all traces |
| `.hasSpanNamed(name)` | At least one span with the given name |

Call `sender.clear()` between test cases to reset captured state.

---

## Complete end-to-end example

This example implements a minimal RAG pipeline that traces all three stages — embedding, retrieval, and generation — into a single trace.

```typescript
import OpenAI from 'openai'
import { TruLayer } from '@trulayer/sdk'

const tl = new TruLayer({
  apiKey: process.env.TRULAYER_API_KEY!,
  projectName: 'rag-app',
})

const openai = new OpenAI()

const CORPUS: Record<string, string> = {
  paris: 'The Eiffel Tower is a wrought-iron lattice tower in Paris, France.',
  rome: 'The Colosseum is a large amphitheatre in the centre of Rome, Italy.',
  berlin: 'The Brandenburg Gate is an 18th-century neoclassical monument in Berlin, Germany.',
}

async function embed(text: string): Promise<number[]> {
  const resp = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  })
  return resp.data[0]?.embedding ?? []
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dot = 0, na = 0, nb = 0
  for (let i = 0; i < a.length; i++) {
    dot += a[i]! * b[i]!
    na += a[i]! * a[i]!
    nb += b[i]! * b[i]!
  }
  return na && nb ? dot / (Math.sqrt(na) * Math.sqrt(nb)) : 0
}

async function answerQuestion(question: string): Promise<string> {
  return tl.trace(
    'rag-query',
    async (t) => {
      t.setInput(question)

      // Stage 1: embed the query
      const queryVec = await t.span('embed-query', 'default', async (s) => {
        s.setModel('text-embedding-3-small')
        s.setInput(question)
        const vec = await embed(question)
        s.setOutput(`vector[${vec.length}]`)
        return vec
      })

      // Stage 2: retrieve top documents
      const docs = await t.span('retrieve-docs', 'retrieval', async (s) => {
        s.setInput(question)
        const scored: Array<[string, number]> = []
        for (const [id, doc] of Object.entries(CORPUS)) {
          const docVec = await embed(doc)
          scored.push([id, cosineSimilarity(queryVec, docVec)])
        }
        scored.sort((a, b) => b[1] - a[1])
        const top = scored.slice(0, 2).map(([id]) => CORPUS[id]!)
        s.setOutput(top.join('\n---\n'))
        s.setMetadata({ topK: scored.slice(0, 2).map(([id, score]) => ({ id, score })) })
        return top
      })

      // Stage 3: generate the answer
      const context = docs.map((d) => `- ${d}`).join('\n')
      const prompt = `Answer using only the context.\n\nContext:\n${context}\n\nQuestion: ${question}`

      const answer = await t.span('generate-answer', 'llm', async (s) => {
        s.setModel('gpt-4o-mini')
        s.setInput(prompt)
        const resp = await openai.chat.completions.create({
          model: 'gpt-4o-mini',
          messages: [
            { role: 'system', content: 'Answer strictly from the given context.' },
            { role: 'user', content: prompt },
          ],
          temperature: 0,
        })
        const text = resp.choices[0]?.message?.content?.trim() ?? ''
        s.setOutput(text)
        if (resp.usage) {
          s.setTokens(resp.usage.prompt_tokens, resp.usage.completion_tokens)
        }
        return text
      })

      t.setOutput(answer)
      return answer
    },
    { tags: ['rag', 'production'] },
  )
}

// Run the pipeline
const result = await answerQuestion('Which city is the Eiffel Tower in?')
console.log('Answer:', result)

await tl.shutdown()
```

This trace will appear in the TruLayer dashboard with:

- A top-level trace named `rag-query` with `input`, `output`, latency, and tags
- Three child spans: `embed-query` (default), `retrieve-docs` (retrieval), and `generate-answer` (llm)
- Token counts and model name on the LLM span
- Cosine similarity scores in the retrieval span metadata
