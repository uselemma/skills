# Vercel AI SDK — Integration Notes

Targets `@uselemma/tracing` >= 3.0.0.

Docs: `https://docs.uselemma.ai/integrations/vercel-ai-sdk.md`

---

## Prefer the native AI SDK path

When the app already uses Vercel AI SDK, keep the AI SDK as the integration layer. Do not replace `generateText`, `streamText`, `generateObject`, or AI SDK-managed tools with Langfuse-style wrappers or provider-level instrumentation.

Use AI SDK telemetry first:

- Add `experimental_telemetry: { isEnabled: true }` to every AI SDK call.
- Wrap the application-level agent/run boundary with `agent()` when a Lemma root span is needed.
- Avoid OpenInference for the underlying OpenAI/Anthropic provider unless the app bypasses AI SDK and calls the provider SDK directly.

---

## `experimental_telemetry` must be on every call

This is the #1 missed step. Every single `generateText`, `streamText`, or `generateObject` call needs it. Miss it on one call inside a multi-step agent and that call produces no child spans — the root span appears but looks empty.

```typescript
// every AI SDK call, every time
experimental_telemetry: { isEnabled: true }
```

AI SDK tool calls are also captured automatically when this flag is set — you do **not** need the `tool()` helper for AI SDK-managed tool calls.

---

## `agent()` span is always the trace root

Looking at the source: `agent()` starts its span from `ROOT_CONTEXT`, not the current active context. This means:

- AI SDK spans nest **inside** the `agent()` span — the agent run is the root
- Calling `agent()` **inside** another `agent()` creates two parallel trace roots, not a parent-child span. Don't nest `agent()` calls.

---

## `onFinish` is where `ctx.complete()` belongs

For streaming with Vercel AI SDK, `streamText`'s `onFinish` callback fires once the full output is assembled — this is the canonical place to call `ctx.complete()`. The snippet below is the complete base pattern:

```typescript
import { agent } from "@uselemma/tracing";
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

const myAgent = agent(
  "my-agent",
  async (input: string, ctx) => {
    return await streamText({
      model: openai("gpt-4o"),
      messages: [{ role: "user", content: input }],
      experimental_telemetry: { isEnabled: true },
      onFinish({ text }) {
        ctx.complete(text); // records output, closes the run span
      },
    });
  },
  { streaming: true }
);
```

Avoid calling `ctx.complete()` inside a `for await` loop over the stream — `onFinish` fires after the loop ends, so calling it mid-loop closes the span before the output is fully assembled.

---

## `instrumentation.ts` location

Must be at the **project root** — the same level as `package.json`. If using a `src/` directory layout, place it at `src/instrumentation.ts`.

The `register()` function is called **twice** in development (once for `edge`, once for `nodejs`). Without the `NEXT_RUNTIME === "nodejs"` guard, `registerOTel()` runs in the edge runtime where the Node.js OTel SDK is not supported and will throw.

---

## `ctx.complete()` is optional in non-streaming mode

In non-streaming mode, the wrapper calls `complete(result)` automatically after your function returns — it uses whatever value the function returned as `ai.agent.output` and then closes the span. You never need to call it yourself unless you want to override one of those two behaviors:

**Override the recorded output** — call `ctx.complete(differentValue)` before returning when the return value isn't what you want to appear as the run output. For example, if your function returns a complex object but you only want to record the text field:

```typescript
import { agent } from "@uselemma/tracing";
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

const myAgent = agent("my-agent", async (input: string, ctx) => {
  const response = await generateText({
    model: openai("gpt-4o"),
    messages: [{ role: "user", content: input }],
    experimental_telemetry: { isEnabled: true },
  });
  // ctx.complete(response.text);
  return response;             // caller still gets the full response object
});
```

**Close the span early** — call `ctx.complete()` mid-function when you want the trace to end before the function exits. Useful when the "real" work finishes before cleanup code runs and you don't want the cleanup time attributed to the run.

`complete()` is idempotent — the first call wins, all subsequent calls are no-ops. So calling it explicitly and then returning normally is safe; the auto-close at the end just becomes a no-op.

---

## Env vars must not be `NEXT_PUBLIC_`

`LEMMA_API_KEY` and `LEMMA_PROJECT_ID` are server-only secrets. Never prefix them with `NEXT_PUBLIC_` — that would embed them in the client bundle.
