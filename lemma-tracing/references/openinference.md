# OpenInference — Integration Notes

Targets `@uselemma/tracing` >= 3.0.0.

Docs: `https://docs.uselemma.ai/tracing/adding-provider-instrumentation.md`

---

## Native framework integrations come first

Before adding OpenInference, check whether the app is using an AI framework with native telemetry or OTel support. Prefer that framework integration over provider-level instrumentation or Langfuse-style wrapping. OpenInference is only the right choice when the app calls the provider SDK directly with no framework layer in between.

---

## Package lookup

OpenInference provides instrumentors for many providers. The table below covers the ones referenced in Lemma's docs — for the full list and setup guides, see the [OpenInference docs](https://arize-ai.github.io/openinference).

| Provider | npm package | pip package |
|---|---|---|
| OpenAI | `@arizeai/openinference-instrumentation-openai` | `openinference-instrumentation-openai` |
| Anthropic | `@arizeai/openinference-instrumentation-anthropic` | `openinference-instrumentation-anthropic` |
| LiteLLM | — (Python only) | `openinference-instrumentation-litellm` |

Note: npm packages have the `@arizeai/` scope; pip packages don't.

---

## Registration patterns

### TypeScript — two valid forms

**`instrumentProvider` (preferred when you have the provider object):**

```typescript
import { OpenAIInstrumentation } from "@arizeai/openinference-instrumentation-openai";

const provider = registerOTel(); // returns the NodeTracerProvider
new OpenAIInstrumentation().instrumentProvider(provider);

// Create client AFTER registration
const openai = new OpenAI();
```

**`registerInstrumentations` (when working with an existing provider):**

```typescript
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { OpenAIInstrumentation } from "@arizeai/openinference-instrumentation-openai";

registerInstrumentations({
  instrumentations: [new OpenAIInstrumentation()],
  tracerProvider: provider, // must be the same provider Lemma uses
});
```

### Python — call before importing the client

```python
from openinference.instrumentation.openai import OpenAIInstrumentor
from uselemma_tracing import register_otel

provider = register_otel()
OpenAIInstrumentor().instrument(tracer_provider=provider)

# Import client AFTER instrumentation
import openai
```

Python Anthropic:
```python
from openinference.instrumentation.anthropic import AnthropicInstrumentor
AnthropicInstrumentor().instrument(tracer_provider=provider)
```

Python LiteLLM:
```python
from openinference.instrumentation.litellm import LiteLLMInstrumentor
LiteLLMInstrumentor().instrument(tracer_provider=provider)
```

---

## The critical constraint: client after instrumentation

OpenInference patches the provider SDK at registration time. Any client object created before instrumentation holds an unpatched reference and will emit no child spans. The import itself can also bind references — in Python, `from openai import AsyncOpenAI` at the top of a module runs at import time, which may precede your instrumentation call.

Safe pattern: call `register_otel()` + `Instrumentor().instrument()` in a module that is imported first, before any module that imports the provider SDK.

---

## LiteLLM: always call on the module

OpenInference patches `litellm.acompletion`, `litellm.completion`, etc. on the module object. Importing the function directly (`from litellm import acompletion`) captures the unpatched reference before the patch runs.

```python
# Wrong — captures unpatched reference
from litellm import acompletion
result = await acompletion(...)

# Correct — reads patched attribute at call time
import litellm
result = await litellm.acompletion(...)
```

---

## When NOT to use OpenInference

These frameworks emit their own spans automatically — adding OpenInference on top produces **duplicate child spans**:

- **Vercel AI SDK** — uses `experimental_telemetry` to emit `ai.generateText` / `ai.streamText` spans natively. No OpenInference needed.
- **OpenAI Agents SDK** — has its own built-in OTel instrumentation. No OpenInference needed.
- **LangChain** — uses `LangChainInstrumentor` (a different Arize package), not the OpenAI/Anthropic ones.

Only add OpenInference when calling the provider SDK **directly**, with no framework layer in between.

---

## What OpenInference emits

Each instrumented provider call produces a `gen_ai.chat` child span with:

- `gen_ai.system` (e.g. `openai`)
- `gen_ai.request.model` — model name requested
- `gen_ai.response.model` — model name returned
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens`
- Full prompt messages and completion content

These nest automatically under the active `agent()` span — no manual wiring needed.
