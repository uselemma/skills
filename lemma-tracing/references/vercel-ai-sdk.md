# Vercel AI SDK

Use this reference when the app uses Vercel AI SDK v7 or v6.

## Decision Path

1. Detect AI SDK version from package files and call shape.
2. Wrap one agent execution in `lemma.trace(...)`.
3. Use `vercelAI()` so model calls become Lemma generations and tool executions become Lemma tool calls.
4. For streaming or externally finalized work, pass a trace handle to `vercelAI({ trace })`.
5. Verify with debug mode and a real AI SDK call.

Do not use Langfuse as the integration layer for Lemma work. If the app already has Langfuse, keep it only if the customer still needs Langfuse data, and add Lemma SDK tracing alongside it. Langfuse instrumentation alone is not sufficient for Lemma because it usually does not produce the Lemma trace contract.

Docs:

- `https://docs.uselemma.ai/integrations/vercel-ai.md`
- `https://docs.uselemma.ai/tracing/instrumentation/traces.md`
- `https://docs.uselemma.ai/tracing/troubleshooting/debug-mode.md`
- `https://docs.uselemma.ai/reference/trace-contract.md`

## Install

```bash
npm install @uselemma/tracing ai zod
```

Use server-side environment variables:

```bash
LEMMA_API_KEY=...
LEMMA_PROJECT_ID=...
```

Never expose `LEMMA_API_KEY` to browser code or `NEXT_PUBLIC_*`.

## AI SDK v7

Use `telemetry.integrations`.

```typescript
import { generateText, tool } from "ai";
import { z } from "zod";
import { Lemma, vercelAI } from "@uselemma/tracing";

const lemma = new Lemma();

const answer = await lemma.trace(
  {
    name: "support-agent",
    input: userMessage,
    threadId: conversationId,
    userId,
  },
  async () => {
    const result = await generateText({
      model,
      prompt: userMessage,
      tools: {
        searchDocs: tool({
          inputSchema: z.object({ query: z.string() }),
          execute: async ({ query }) => searchDocs(query),
        }),
      },
      telemetry: {
        integrations: [vercelAI()],
      },
    });

    return result.text;
  },
);
```

## AI SDK v6

Use `experimental_telemetry.integrations`.

```typescript
import { generateText, tool } from "ai";
import { z } from "zod";
import { Lemma, vercelAI } from "@uselemma/tracing";

const lemma = new Lemma();

const answer = await lemma.trace(
  {
    name: "support-agent",
    input: userMessage,
    threadId: conversationId,
    userId,
  },
  async () => {
    const result = await generateText({
      model,
      prompt: userMessage,
      tools: {
        searchDocs: tool({
          inputSchema: z.object({ query: z.string() }),
          execute: async ({ query }) => searchDocs(query),
        }),
      },
      experimental_telemetry: {
        integrations: [vercelAI()],
      },
    });

    return result.text;
  },
);
```

## Streaming

For streaming, create a trace handle and pass it to `vercelAI({ trace })`. The integration closes the handle from the AI SDK terminal callback:

- AI SDK v7: `onEnd`
- AI SDK v6: `onFinish`

```typescript
import { streamText } from "ai";
import { Lemma, vercelAI } from "@uselemma/tracing";

const lemma = new Lemma();

const trace = lemma.trace({
  name: "support-agent",
  input: userMessage,
  threadId: conversationId,
  userId,
});

const result = streamText({
  model,
  prompt: userMessage,
  telemetry: {
    integrations: [vercelAI({ trace })],
  },
});

for await (const part of result.fullStream) {
  // stream to the response
}
```

If the app uses AI SDK v6 streaming syntax, keep its existing stream handling but put `vercelAI({ trace })` under `experimental_telemetry.integrations`.

## Recording Controls

Disable captured prompts, tool inputs, tool outputs, or model output text when needed:

```typescript
telemetry: {
  integrations: [
    vercelAI({
      recordInputs: false,
      recordOutputs: false,
    }),
  ],
}
```

## What Lemma Records

| AI SDK event | Lemma record |
| --- | --- |
| Model call | Generation with model, provider, messages, output text, token usage, and duration when available |
| Tool execution | Tool call with name, input, output or error, and duration when available |

AI SDK v7 provides model and tool durations directly. AI SDK v6 provides tool execution durations; model-call durations may be inferred by Lemma from the surrounding trace when not provided.

## Debugging

Use [debug-mode.md](debug-mode.md) when Vercel AI traces are missing or incomplete.

For AI SDK-specific debugging:

- Expect `trace started` for callback traces or `trace handle created` for streaming handle traces.
- Expect `sending trace` followed by `trace sent`.
- Confirm `spanCount` includes the model generation and any tool executions.
- If streaming never sends, confirm the terminal callback (`onEnd` in v7, `onFinish` in v6) runs and the trace handle is ended.
