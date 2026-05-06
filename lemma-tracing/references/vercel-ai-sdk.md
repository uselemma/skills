# Vercel AI SDK in Next.js

Use this reference only when the app uses Vercel AI SDK in Next.js.

## Decision path

1. Check whether AI SDK calls already use `experimental_telemetry`.
2. If yes, keep that instrumentation and add Lemma export.
3. If no, enable AI SDK telemetry and use Langfuse as the instrumentation layer.
4. Present a plan and ask for confirmation before editing.

Docs:
- `https://docs.uselemma.ai/tracing/using-a-supported-framework.md`
- `https://docs.uselemma.ai/tracing/patterns/otlp-export.md`
- `https://docs.uselemma.ai/tracing/patterns/dual-export.md`
- `https://langfuse.com/integrations/frameworks/vercel-ai-sdk`

## AI SDK telemetry

Every `generateText`, `streamText`, or `generateObject` call that should appear in traces needs telemetry enabled:

```typescript
experimental_telemetry: {
  isEnabled: true,
  functionId: "support-agent",
  metadata: {
    "gen_ai.agent.name": "support-agent",
    "lemma.thread_id": threadId,
    "user.id": userId,
    "session.id": sessionId,
  },
}
```

By semantic convention, `gen_ai.agent.name` values should use `snake_case`, `CamelCase`, or `kebab-case`, such as `support_agent`, `SupportAgent`, or `support-agent`.

## Single export: Lemma only

Use this when Langfuse processes AI SDK spans but Lemma is the only trace destination.

Install:

```bash
npm install @langfuse/otel @opentelemetry/exporter-trace-otlp-proto @opentelemetry/sdk-trace-node
```

Set env vars:

```bash
LEMMA_OTLP_TRACES_URL=...
LEMMA_API_KEY=...
LEMMA_PROJECT_ID=...
```

In root `instrumentation.ts` or `src/instrumentation.ts`:

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { LangfuseSpanProcessor } = await import("@langfuse/otel");
    const { OTLPTraceExporter } = await import(
      "@opentelemetry/exporter-trace-otlp-proto"
    );
    const { NodeTracerProvider } = await import(
      "@opentelemetry/sdk-trace-node"
    );

    const provider = new NodeTracerProvider({
      spanProcessors: [
        new LangfuseSpanProcessor({
          exporter: new OTLPTraceExporter({
            url: process.env.LEMMA_OTLP_TRACES_URL,
            headers: {
              Authorization: `Bearer ${process.env.LEMMA_API_KEY}`,
              "X-Lemma-Project-ID": process.env.LEMMA_PROJECT_ID,
            },
          }),
        }),
      ],
    });

    provider.register();
  }
}
```

`LANGFUSE_*` variables are not required for this mode.

## Dual export: Langfuse and Lemma

Use this when traces should be stored in both Langfuse and Lemma.

Set Lemma env vars plus:

```bash
LANGFUSE_PUBLIC_KEY=...
LANGFUSE_SECRET_KEY=...
LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

In root `instrumentation.ts` or `src/instrumentation.ts`:

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    const { LangfuseSpanProcessor } = await import("@langfuse/otel");
    const { OTLPTraceExporter } = await import(
      "@opentelemetry/exporter-trace-otlp-proto"
    );
    const { NodeTracerProvider } = await import(
      "@opentelemetry/sdk-trace-node"
    );

    const lemmaExporter = new OTLPTraceExporter({
      url: process.env.LEMMA_OTLP_TRACES_URL,
      headers: {
        Authorization: `Bearer ${process.env.LEMMA_API_KEY}`,
        "X-Lemma-Project-ID": process.env.LEMMA_PROJECT_ID,
      },
    });

    const provider = new NodeTracerProvider({
      spanProcessors: [
        new LangfuseSpanProcessor(),
        new LangfuseSpanProcessor({ exporter: lemmaExporter }),
      ],
    });

    provider.register();
  }
}
```

## Constraints

- Guard Node OpenTelemetry setup with `process.env.NEXT_RUNTIME === "nodejs"`.
- Never expose `LEMMA_API_KEY` through `NEXT_PUBLIC_*`.
- Use `@opentelemetry/exporter-trace-otlp-proto`; Lemma expects protobuf payloads.
- If the app already has a tracer provider, add processors/exporters to that provider instead of replacing it.
