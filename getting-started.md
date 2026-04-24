# Getting Started

Get from zero to your first trace in about 10 minutes.

## Prerequisites

- Python 3.9+ **or** Node.js 18+
- An OpenAI API key (`OPENAI_API_KEY`)
- A TruLayer account — [sign up free](https://app.trulayer.ai/signup)

## Create an API key

1. Open [app.trulayer.ai](https://app.trulayer.ai) and go to **Settings → API Keys**.
2. Click **New key**, give it a name, and copy the value immediately — it won't be shown again.
3. Store it as an environment variable:

```bash
export TRULAYER_API_KEY="tl-..."
```

## Install the SDK

```python
# Python
pip install trulayer
```

```typescript
// TypeScript
npm install @trulayer/sdk
```

## Send your first trace

The snippet below opens a trace, creates two child spans (a retrieval step and an LLM call), then closes everything out. The SDK ships the data to `https://api.trulayer.ai` in the background.

```python
# Python
import time
from trulayer import TruLayerClient
import openai

client = TruLayerClient(api_key="tl-...")   # or reads TRULAYER_API_KEY
openai_client = openai.OpenAI()

question = "What is the capital of France? Answer in one short sentence."

with client.trace(
    name="basic-qa",
    external_id="basic-qa-demo",
    tags=["demo", "basic-trace"],
    metadata={"example": "basic_trace.py"},
) as t:
    t.set_input(question)
    t.set_model("gpt-4o-mini")

    # Step 1 — retrieval span
    with t.span("retrieve-context", span_type="retrieval") as s:
        s.set_input(question)
        context = "France is a country in Western Europe; its capital is Paris."
        s.set_output(context)
        s.set_metadata(source="static-fixture")

    # Step 2 — LLM span
    prompt = f"Use this context:\n{context}\n\nQuestion: {question}"
    with t.span("openai.chat", span_type="llm") as s:
        s.set_model("gpt-4o-mini")
        s.set_input(prompt)

        start = time.monotonic()
        resp = openai_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a concise assistant."},
                {"role": "user", "content": prompt},
            ],
            temperature=0,
        )
        elapsed_ms = int((time.monotonic() - start) * 1000)

        answer = (resp.choices[0].message.content or "").strip()
        s.set_output(answer)
        if resp.usage is not None:
            s.set_tokens(
                prompt=resp.usage.prompt_tokens,
                completion=resp.usage.completion_tokens,
            )
        s.set_metadata(provider_latency_ms=elapsed_ms)

    t.set_output(answer)

client.shutdown(timeout=2.0)
print(f"Trace sent: {t._data.id}")
```

```typescript
// TypeScript
import TruLayerClient from '@trulayer/sdk'
import OpenAI from 'openai'

const client = new TruLayerClient({ apiKey: 'tl-...' })  // or reads TRULAYER_API_KEY
const openai = new OpenAI()

const question = 'What is the capital of France? Answer in one short sentence.'

const { traceId } = await client.trace(
  'basic-qa',
  async (t) => {
    t.setInput(question)
    t.setModel('gpt-4o-mini')

    // Step 1 — retrieval span
    const context = await t.span('retrieve-context', 'retrieval', async (s) => {
      s.setInput(question)
      const text = 'France is a country in Western Europe; its capital is Paris.'
      s.setOutput(text)
      s.setMetadata({ source: 'static-fixture' })
      return text
    })

    // Step 2 — LLM span
    const prompt = `Use this context:\n${context}\n\nQuestion: ${question}`
    const answer = await t.span('openai.chat', 'llm', async (s) => {
      s.setModel('gpt-4o-mini')
      s.setInput(prompt)

      const started = Date.now()
      const resp = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
          { role: 'system', content: 'You are a concise assistant.' },
          { role: 'user', content: prompt },
        ],
        temperature: 0,
      })
      const providerLatencyMs = Date.now() - started

      const text = resp.choices[0]?.message?.content?.trim() ?? ''
      s.setOutput(text)
      if (resp.usage) {
        s.setTokens(resp.usage.prompt_tokens, resp.usage.completion_tokens)
      }
      s.setMetadata({ providerLatencyMs })
      return text
    })

    t.setOutput(answer)
    return { traceId: t.data.id }
  },
  {
    externalId: 'basic-qa-demo',
    tags: ['demo', 'basic-trace'],
    metadata: { example: 'basic-trace.ts' },
  },
)

await client.shutdown()
console.log(`Trace sent: ${traceId}`)
```

## View your trace

Open [app.trulayer.ai](https://app.trulayer.ai) and go to **Traces**. Your trace appears within a few seconds. Click it to see the timeline of spans, per-step latency, token counts, and the input/output at each level.

## Next steps

- [Core Concepts](/concepts) — traces, spans, evaluations, and feedback loops explained
- [Python SDK reference](/sdks/python/reference) — full API reference for `trulayer`
- [TypeScript SDK reference](/sdks/typescript/reference) — full API reference for `@trulayer/sdk`
