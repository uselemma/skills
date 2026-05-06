---
name: lemma-tracing
version: 1.0.0
last_updated: 2026-05-06
description: >-
  Integrate Lemma AI observability tracing into a codebase. Use when adding
  Lemma tracing, routing OpenTelemetry spans to Lemma, configuring Langfuse
  instrumentation with Lemma OpenTelemetry export, fixing missing traces, adding
  conversation thread metadata, or debugging Lemma span export.
---

# Lemma Tracing

## Philosophy

You are an experienced Lemma integrator. Detect first, decide second, read the relevant docs third, present a plan fourth, then edit only after confirmation.

**Always:** detect existing instrumentation -> choose path -> read docs -> present plan -> ask for confirmation -> implement -> verify.

## Primary path

Follow this sequence for every integration:

1. **Check if the app is already instrumented with OpenTelemetry-compatible spans.**
   Look for `@opentelemetry/*`, `TracerProvider`, `BatchSpanProcessor`, `OTLPTraceExporter`, OpenInference, Arize, Braintrust, Langfuse, AI SDK telemetry, or existing collector/exporter config.
2. **If already instrumented, proceed with that instrumentation.**
   Do not replace it. Add Lemma as an OTLP trace export destination on the existing provider, processor list, collector, or exporter pipeline.
3. **If not already instrumented, instrument with Langfuse.**
   Use Langfuse or a Langfuse-supported framework guide to create spans, then configure those spans to export to Lemma.
4. **Present a concise plan and ask for confirmation before editing.**
   Include the detected path, files to change, dependencies, required env vars, and verification steps. Do not edit code until the user approves, unless they explicitly asked you to proceed without confirmation.
5. **Proceed with the remainder.**
   Configure Lemma OpenTelemetry export, add Lemma metadata such as `lemma.thread_id` when relevant, and verify traces.

## Docs

Base URL: `https://docs.uselemma.ai`

Use docs in this order:

1. Fetch `https://docs.uselemma.ai/llms.txt` to discover current pages.
2. Read the most relevant page before editing:
   - Greenfield or Langfuse processing: `https://docs.uselemma.ai/integrations/langfuse.md`
   - Existing framework support: `https://docs.uselemma.ai/tracing/using-a-supported-framework.md`
   - Existing OTel pipeline or direct export: `https://docs.uselemma.ai/tracing/otlp-export.md`
   - Multiple destinations: `https://docs.uselemma.ai/tracing/patterns/dual-export.md`
   - OpenInference: `https://docs.uselemma.ai/integrations/openinference.md`
   - Troubleshooting: `https://docs.uselemma.ai/tracing/troubleshooting/common-problems.md`

## Detect

Inspect imports, startup files, and framework calls before deciding anything:

| Signal | How to detect |
|---|---|
| Existing OTel | `@opentelemetry/*`, `TracerProvider`, `BatchSpanProcessor`, `OTLPTraceExporter`, collector env vars |
| Existing Langfuse | `@langfuse/otel`, `@langfuse/tracing`, `LangfuseSpanProcessor`, `observe`, `startActiveObservation` |
| Existing OpenInference | `openinference`, `@arizeai/openinference-*`, Arize/Phoenix instrumentation |
| Vercel AI SDK | `generateText`, `streamText`, `generateObject`, `experimental_telemetry` from `ai` |
| Provider SDK | `openai`, `@anthropic-ai/sdk`, `anthropic`, `litellm` |
| Next.js runtime | root `instrumentation.ts` or `src/instrumentation.ts`, `NEXT_RUNTIME` guard |
| Streaming | `streamText`, SSE handlers, async iterables, `ReadableStream` |

Ask a focused clarification question when there are multiple agent entry points, multiple telemetry stacks, or unclear runtime ownership.

## Plan requirement

Before editing code, show the user a plan with:

- **Detected path:** already-instrumented OTel vs. needs Langfuse instrumentation.
- **Files:** startup instrumentation files and agent/model call sites to change.
- **Dependencies:** packages to add or keep.
- **Environment:** `LEMMA_BASE_URL` (`https://api.uselemma.ai/otel/v1/traces`), `LEMMA_API_KEY`, `LEMMA_PROJECT_ID`.
- **Verification:** type check, smoke trace, logs, and any manual dashboard check.

`LEMMA_BASE_URL` is the full Lemma OTLP traces endpoint, not just the API origin. Use `https://api.uselemma.ai/otel/v1/traces`.

Then ask for confirmation and wait.

## Implementation patterns

### Existing OpenTelemetry-compatible instrumentation

Keep the current instrumentation and add a Lemma OTLP trace exporter.

TypeScript exporters should use protobuf:

```typescript
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";

const lemmaExporter = new OTLPTraceExporter({
  url: process.env.LEMMA_BASE_URL,
  headers: {
    Authorization: `Bearer ${process.env.LEMMA_API_KEY}`,
    "X-Lemma-Project-ID": process.env.LEMMA_PROJECT_ID,
  },
});
```

Python exporters should use `opentelemetry.exporter.otlp.proto.http.trace_exporter.OTLPSpanExporter` with the same endpoint and headers.

### Not yet instrumented

Use Langfuse as the instrumentation layer, then export the resulting spans to Lemma. For framework-specific apps, follow the matching Langfuse framework guide and then apply Lemma OpenTelemetry export:

- Vercel AI SDK: Langfuse Vercel AI SDK guide plus `experimental_telemetry`.
- OpenAI Agents SDK: Langfuse OpenAI Agents guide.
- Claude Agent SDK JS/TS: Langfuse Claude Agent SDK guide.
- LangChain/LangGraph: Langfuse framework guide.
- Custom frameworks: Langfuse OpenTelemetry or native OpenTelemetry spans.

For Vercel AI SDK in Next.js, also read [references/vercel-ai-sdk.md](references/vercel-ai-sdk.md) for single-export and dual-export startup patterns.

For TypeScript with `LangfuseSpanProcessor`, send to Lemma by passing a custom protobuf OpenTelemetry exporter:

```typescript
import { LangfuseSpanProcessor } from "@langfuse/otel";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";

new LangfuseSpanProcessor({
  exporter: new OTLPTraceExporter({
    url: process.env.LEMMA_BASE_URL,
    headers: {
      Authorization: `Bearer ${process.env.LEMMA_API_KEY}`,
      "X-Lemma-Project-ID": process.env.LEMMA_PROJECT_ID,
    },
  }),
});
```

Use `LANGFUSE_*` variables only if Langfuse is also a storage destination. Lemma-only export requires only Lemma's endpoint, API key, and project ID.

## Metadata

Use `lemma.thread_id` as Lemma's canonical conversation grouping key. Add it where the instrumentation accepts OpenTelemetry-style metadata.

By semantic convention, `gen_ai.agent.name` values should use `snake_case`, `CamelCase`, or `kebab-case`, such as `support_agent`, `SupportAgent`, or `support-agent`.

For Vercel AI SDK:

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

Use semantic convention keys for general metadata when possible: `user.id`, `session.id`, `deployment.environment.name`, `service.version`.

## Critical constraints

- Register startup instrumentation before app code emits spans or provider clients are instantiated.
- In Next.js, put startup registration in root `instrumentation.ts` or `src/instrumentation.ts` and guard Node-only SDK setup with `process.env.NEXT_RUNTIME === "nodejs"`.
- Use `@opentelemetry/exporter-trace-otlp-proto` for TypeScript HTTP/protobuf export. A JSON OpenTelemetry exporter can produce "invalid protobuf payload" errors.
- Never expose `LEMMA_API_KEY` in client code or `NEXT_PUBLIC_*` env vars.
- If another tracer provider already exists, add Lemma to it instead of replacing it.
- For short-lived/serverless handlers, flush processors/exporters at request end when the runtime may freeze before batches export.

## Troubleshooting

Before suggesting fixes, check in this order:

1. **No traces:** startup instrumentation ran before traffic; Lemma env vars are set; egress to the OTLP endpoint works.
2. **Traces in source backend but not Lemma:** Lemma exporter is attached; headers include `Authorization` and `X-Lemma-Project-ID`; runtime logs show no exporter errors.
3. **Missing child spans:** framework/provider instrumentation is enabled and registered before clients are created.
4. **Invalid payload:** TypeScript uses `@opentelemetry/exporter-trace-otlp-proto`, and `LEMMA_BASE_URL` is the full Lemma OTLP traces endpoint: `https://api.uselemma.ai/otel/v1/traces`.
5. **Delayed data:** batch processor may need `forceFlush()` in short-lived runtimes.

If Lemma MCP is connected, query recent traces directly before adding debug logs.

## Skill feedback

If the skill gives incorrect guidance, references a page that does not exist, or misses a scenario, offer to submit feedback. See [references/skill-feedback.md](references/skill-feedback.md).
