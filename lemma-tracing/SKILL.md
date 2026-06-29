---
name: lemma-tracing
description: >-
  Integrate Lemma AI observability tracing into a codebase. Use when adding
  Lemma tracing, fixing missing or malformed traces, adding tool calls,
  generations, trace handles, thread/user context, Vercel AI SDK integration,
  or debugging Lemma trace delivery and trace shape.
---

# Lemma Tracing

## Operating Mode

Integrate Lemma with the direct tracing SDK by default. The SDK sends trace JSON to Lemma and emits the trace shape Lemma understands.

Work in this order:

1. Detect the app language, framework, runtime, agent entry points, model calls, tool calls, streaming/finalization path, and any existing tracing.
2. Choose the smallest integration path that produces the Lemma trace contract.
3. Read the relevant docs/reference files before editing.
4. Present a concise plan when the user asks for one or when the integration touches multiple files.
5. Implement, verify with tests or a smoke trace, and use debug mode for delivery or shape issues.

## Contract First

Every integration must satisfy this product contract:

- One agent execution becomes one `lemma.trace(...)` root trace.
- The root trace has a stable `name`, user input, final output or error, and `threadId` / `userId` when available.
- LLM calls are generation children: `recordGeneration(...)` / `record_generation(...)` or `startGeneration(...)` / `start_generation(...)`.
- Tool invocations are tool children: `recordTool(...)` / `record_tool(...)` or `startTool(...)` / `start_tool(...)`.
- Retrieval, ranking, planning, routing, and app logic are spans: `recordSpan(...)` / `record_span(...)` or `startSpan(...)` / `start_span(...)`.
- Nested work is recorded on the parent span handle when it should appear under that parent.

If the app cannot produce this shape at the exact call site, pass IDs and record from the coordinator that knows the full trace, or start one top-level trace per process and carry the same `threadId`.

## Choose the Integration Path

| Situation | Preferred path |
| --- | --- |
| New or manually instrumented TypeScript app | Use `@uselemma/tracing` directly. See [references/direct-sdk.md](references/direct-sdk.md). |
| New or manually instrumented Python app | Use `uselemma-tracing` directly. See [references/direct-sdk.md](references/direct-sdk.md). |
| Vercel AI SDK v7 or v6 | Wrap the run in `lemma.trace(...)` and use `vercelAI()`. See [references/vercel-ai-sdk.md](references/vercel-ai-sdk.md). |
| Streaming or callbacks where one function does not own the whole run | In TypeScript, use a trace handle and call/await `trace.end(...)` from the terminal callback or finalization path. In Python, prefer a callback trace around the owned run and record `start_*` handles inside that callback. |
| Existing Langfuse customers | Preserve their Langfuse setup for backwards compatibility when needed, but add Lemma SDK tracing for the Lemma contract. Do not tell new greenfield users to instrument with Langfuse first. |
| Existing OpenTelemetry only | Do not tear it out. Keep it if the user needs it, but use the Lemma SDK for the product trace contract unless the user explicitly asks for OTel export compatibility work. |

## Docs

Base URL: `https://docs.uselemma.ai`

Use docs in this order:

1. Fetch `https://docs.uselemma.ai/llms.txt` when live docs are needed.
2. Read the relevant current page before editing:
   - Setup: `https://docs.uselemma.ai/tracing/instrumentation/setup.md`
   - Step-by-step agent instrumentation: `https://docs.uselemma.ai/tracing/instrumentation/instrument-an-agent.md`
   - Traces and handles: `https://docs.uselemma.ai/tracing/instrumentation/traces.md`
   - Generations: `https://docs.uselemma.ai/tracing/instrumentation/generations.md`
   - Tool calls: `https://docs.uselemma.ai/tracing/instrumentation/tool-calls.md`
   - Spans: `https://docs.uselemma.ai/tracing/instrumentation/spans.md`
   - Context: `https://docs.uselemma.ai/tracing/instrumentation/context.md`
   - Vercel AI SDK: `https://docs.uselemma.ai/integrations/vercel-ai.md`
   - Trace contract: `https://docs.uselemma.ai/reference/trace-contract.md`
   - Troubleshooting: `https://docs.uselemma.ai/tracing/troubleshooting/common-issues.md`
   - Debug mode: `https://docs.uselemma.ai/tracing/troubleshooting/debug-mode.md`

Prefer local package types/tests over memory when the SDK API is available in the workspace.

## Detection Checklist

Inspect imports, startup files, agent handlers, and model/tool call sites:

| Signal | How to detect |
| --- | --- |
| Lemma SDK already present | `@uselemma/tracing`, `uselemma_tracing`, `Lemma`, `vercelAI`, `lemma.trace` |
| Agent boundary | request handler, job processor, CLI command, workflow step, streaming route |
| Vercel AI SDK | `generateText`, `streamText`, `generateObject`, `tool`, `telemetry`, `experimental_telemetry` from `ai` |
| Model call | `openai`, `anthropic`, provider adapters, AI SDK model usage |
| Tool call | functions passed as tools, MCP calls, retrieval/search/order/payment helpers |
| Trace finalization | callback return, `onEnd`, `onFinish`, SSE close, queue completion, background job completion |
| Existing tracing | Langfuse, OpenTelemetry, OpenInference, Arize/Phoenix, Braintrust |

Ask one focused clarification question only when the agent boundary or finalization path is genuinely ambiguous.

## Implementation Rules

- Create one shared `Lemma` client on the server side.
- Use `LEMMA_API_KEY` and `LEMMA_PROJECT_ID`. The default endpoint is `https://api.uselemma.ai`; pass `baseUrl` / `base_url` only for staging or self-hosted deployments.
- Never expose `LEMMA_API_KEY` in browser code or `NEXT_PUBLIC_*` variables.
- Use callback traces when one function owns the whole run.
- In TypeScript, use trace handles when the run is coordinated across callbacks, streaming, or helpers; do not set final duration until `trace.end(...)`.
- In Python, use callback traces as the root boundary and use `start_span`, `start_tool`, and `start_generation` on the active trace context for work in progress.
- Record completed work with `recordSpan`, `recordTool`, and `recordGeneration`.
- Use `startSpan`, `startTool`, and `startGeneration` when you need a handle before the work finishes.
- Use `traceId` and `parentSpanId` for detached recording by ID.
- Prefer native contract props such as `toolParameters`, `retrievalDocuments`, `llmInputMessages`, `llmInvocationParameters`, `model`, and `usage`; use raw `attributes` only as an escape hatch.
- Redact secrets and sensitive payloads before recording inputs or outputs.

## Debugging

Use debug mode whenever traces are missing, delayed, split into separate roots, missing tools/generations, or have blank input/output. For the full diagnostic workflow, read [references/debug-mode.md](references/debug-mode.md).

Enable it before creating the client:

```typescript
import { enableDebugMode, Lemma } from "@uselemma/tracing";

enableDebugMode();
const lemma = new Lemma();
```

or:

```bash
LEMMA_DEBUG=true
```

Expected successful sequence:

```text
[LEMMA:client] trace started
[LEMMA:client] sending trace
[LEMMA:client] trace sent
```

For trace handles, expect `trace handle created`, then `sending trace` when the handle flushes or ends. Use the `spanCount` (`span_count` in Python) on `sending trace` to confirm children are actually included.

Common reads:

- No Lemma logs: debug mode is not enabled in the running process, or the traced path is not executing.
- `trace started` but no `sending trace`: callback did not finish, or a handle was never flushed/ended.
- `sending trace` then `trace ingest failed`: check credentials, project ID, `baseUrl`, status code, and network policy.
- `trace sent` but dashboard shape is wrong: compare the trace against the contract and record missing children with typed helpers.

## Verification

Use the strongest available evidence:

- Type-check and run focused tests for changed code.
- Run a smoke trace from the same runtime that handles real traffic.
- With debug mode enabled, confirm `trace sent` and the expected child count.
- Verify the dashboard trace has one root, input/output, generation children, tool children, and thread/user context when expected.

## Skill Feedback

If this skill gives incorrect guidance, references a page that does not exist, or misses a scenario, offer to submit feedback. See [references/skill-feedback.md](references/skill-feedback.md).
