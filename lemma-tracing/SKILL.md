---
name: lemma-tracing
version: 1.0.0
sdk_version: ">=3.0.0"
last_updated: 2026-05-06
description: >-
  Integrate Lemma AI observability tracing into a codebase. Detects the user's
  language, framework, and existing OTel setup to pick the correct integration
  path and generate working instrumentation code. Use when the user asks to add
  Lemma tracing, fix missing traces, add tool call spans, track conversation
  threads, handle streaming agents, add Lemma alongside existing Datadog/Jaeger
  OTel, or add custom metadata to AI agent runs.
---

# Lemma Tracing

## Philosophy

You are an experienced Lemma integrator. Detect first, decide second, read the relevant doc page third, then write code. The docs are the reference; you are the judgment.

**Always:** detect → clarify if ambiguous → pick path → read the doc page → present a plan → generate code → check pitfalls.

Prefer native framework integration over provider-level instrumentation or Langfuse-style wrapping. If the app uses Vercel AI SDK, OpenAI Agents SDK, LangChain, or another framework with its own telemetry/OTel integration, use that framework path first. Only add OpenInference or manual wrappers when there is no framework layer and the app calls the provider SDK directly.

## How to access docs

Base URL: `https://docs.uselemma.ai`

Use your available fetch/search tools (WebFetch, mcp_fetch, WebSearch, etc.) in this order:

**1. Start with the index**

Fetch the full list of every published doc page with titles and URLs:
`https://docs.uselemma.ai/llms.txt`

Use this to discover which page covers the topic, then fetch that page directly. Always prefer this over guessing a URL — the published set of pages changes as docs are updated.

**2. Fetch individual pages as markdown**

Append `.md` to any page URL from the index:
`https://docs.uselemma.ai/integrations/vercel-ai-sdk.md`

Read the relevant page before writing code. The decision tree below tells you which page to look for.

**3. Search as a fallback**

If you can't identify the right page from the index, use your available web search tools:
`site:docs.uselemma.ai <query>`

## Detect

Inspect imports and file structure before deciding anything:

| Signal | How to detect |
|---|---|
| **Language** | `.ts`/`.js` = TypeScript · `.py` = Python |
| **Framework** | `from 'ai'` = Vercel AI SDK · `from '@openai/agents'` = OpenAI Agents SDK · `from 'langchain'` or `from langchain` = LangChain |
| **Provider** | `from 'openai'` = OpenAI · `from '@anthropic-ai/sdk'` = Anthropic · `import litellm` = LiteLLM |
| **Streaming** | `streamText`, `messages.stream()`, `messages.create({ stream: true })`, async generator patterns, SSE response |
| **Existing OTel** | `@opentelemetry/` imports, `TracerProvider`, `tracer.start_as_current_span` |
| **Runtime** | `next.config.*` or `instrumentation.ts` = Next.js · otherwise standalone Node.js/Python entry |

## Clarify

Before writing any code, verify you know exactly what to instrument. Ask the user for clarification when any of the following are true:

- **Multiple agent entry points found** — more than one file contains agent/run/pipeline definitions (e.g. `agentA.ts` and `agentB.ts`, or a `agents/` directory with several files).
- **Multiple frameworks coexist** — e.g. Vercel AI SDK and OpenAI Agents SDK imports both appear in the codebase.
- **Vague request** — the user said "add tracing" or "instrument my agents" without pointing to specific files or functions.
- **Multiple agent functions in a single file** — several `agent()` / `run()` / chain definitions are present and it's not obvious which should be wrapped.

When any of these apply, **do not guess** — ask once with a concrete, specific question. For example:

> "I found agent definitions in `src/agents/chat.ts` and `src/agents/search.ts`. Which one(s) should I instrument, or should I add tracing to all of them?"

> "I see both Vercel AI SDK (`generateText`) and OpenAI Agents SDK (`run`) used here. Should I instrument both, or just one?"

If none of the above apply — there is exactly one agent, one framework, and the scope is unambiguous — skip this step and proceed directly to the decision tree.

## Present a plan before proceeding

Before writing or editing code, present a concise plan to the user. Include:

- The files or entry points you will instrument.
- The detected framework/provider and why that integration path was selected.
- Any docs or references you will follow.
- Any user-visible tradeoffs, such as adding a new dependency or changing runtime initialization.

Proceed after the user approves the plan, unless the user explicitly asked you to make the change without another confirmation.

## Decision tree

Check in this order — sequence matters.

**1. Existing OTel provider?**

→ **YES:** Do not call `registerOTel()` — it replaces the global provider and discards all existing processors. Use `createLemmaSpanProcessor()` instead and add it to the existing provider.
→ Read: `https://docs.uselemma.ai/tracing/patterns/dual-export.md`
→ Recipe: `https://docs.uselemma.ai/recipes/dual-export.md`

**2. Framework or provider SDK detected?**

→ Fetch `https://docs.uselemma.ai/llms.txt` and find the matching integration page (look for the detected framework or provider name in the titles).
→ If a framework integration exists, use it before any provider-level or Langfuse-style instrumentation. Native framework spans avoid duplicate traces and preserve the app's existing AI SDK conventions.
→ Read the integration page before writing code — it specifies the exact pattern (e.g. `registerOTel()` alone vs. `registerOTel()` + framework telemetry).
→ Integration pages are split by language (TypeScript / Python) — check the right section for the detected language.
→ If Vercel AI SDK detected: also read [references/vercel-ai-sdk.md](references/vercel-ai-sdk.md) before writing code.
→ If OpenInference is needed (direct provider SDK, no framework): also read [references/openinference.md](references/openinference.md).

**3. No matching integration page?**

→ Manual instrumentation with `tool()`, `llm()`, `retrieval()`, `trace()` helpers.
→ Read: `https://docs.uselemma.ai/tracing/manual-instrumentation.md`
→ Recipe: `https://docs.uselemma.ai/recipes/manual-instrumentation.md`
→ Also read [references/manual-instrumentation.md](references/manual-instrumentation.md) — non-obvious helper constraints (single input arg in TS, nesting rules, when not to use raw spans).

**Then, on every path:**

| Need | Read |
|---|---|
| Streaming | `/tracing/patterns/streaming.md` · Recipe: `/recipes/streaming-sse.md` |
| Thread/conversation tracking | `/tracing/patterns/threads.md` |
| Custom metadata | `/tracing/patterns/custom-attributes.md` |
| Adding Lemma alongside existing OTel | `/tracing/patterns/dual-export.md` · Recipe: `/recipes/dual-export.md` |

## Recipes

If the user's pattern matches one of these, start there before writing from scratch:

| Recipe | Pattern |
|---|---|
| `https://docs.uselemma.ai/recipes/sync-agent.md` | Basic synchronous agent |
| `https://docs.uselemma.ai/recipes/async-agent.md` | Basic async agent |
| `https://docs.uselemma.ai/recipes/streaming-sse.md` | Streaming with SSE response |
| `https://docs.uselemma.ai/recipes/tool-calling-agent.md` | Agent with tool calls |
| `https://docs.uselemma.ai/recipes/multi-step-agent.md` | Multi-step / loop agent |
| `https://docs.uselemma.ai/recipes/context-manager.md` | Python context manager form |
| `https://docs.uselemma.ai/recipes/dual-export.md` | Existing OTel + Lemma side-by-side |
| `https://docs.uselemma.ai/recipes/manual-instrumentation.md` | Full manual span control |

## Critical constraints

Rules the docs mention that agents routinely miss:

- **Registration order:** `registerOTel()` / `register_otel()` must execute before any `agent()` call or provider client instantiation. In Next.js, place it inside `instrumentation.ts` → `register()`, guarded by `process.env.NEXT_RUNTIME === "nodejs"`.
- **Streaming always pairs:** `{ streaming: true }` must always be paired with `ctx.complete(assembledText)` in the finish callback — and in the error path too. Missing either one leaves the span open indefinitely. → `/tracing/patterns/streaming.md`
- **Vercel AI SDK — every call:** `experimental_telemetry: { isEnabled: true }` must appear on every `generateText` / `streamText` call. Missing it on even one call means that call produces no child spans. The root span appears but looks empty.
- **OpenInference registration order:** Register all OpenInference instrumentors against the provider returned by `registerOTel()`, before instantiating any provider client.
- **Thread IDs:** Pass at the call site as the second argument (`{ threadId: "..." }` in TS, `{"thread_id": "..."}` in Python). In context manager mode, set `run.span.set_attribute("lemma.thread_id", ...)` directly. → `/tracing/patterns/threads.md`
- **`lemma.*` prefix:** Custom attributes intended to be filterable in the dashboard must use the `lemma.*` prefix (e.g. `lemma.user_id`, `lemma.environment`). → `/tracing/patterns/custom-attributes.md`
- **Explicit return:** Always return a value from the wrapped function. Returning `undefined` / `None` = blank output field in the dashboard.

## Known pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Missing `experimental_telemetry` on Vercel AI SDK calls | Root span appears, no child LLM spans | Add `experimental_telemetry: { isEnabled: true }` to every `generateText` / `streamText` call |
| `messages.stream()` with AnthropicInstrumentation | `messages.create(...).withResponse is not a function` | Use `messages.create({ stream: true })` and consume the async iterable directly → `/tracing/patterns/streaming.md` |
| `from litellm import acompletion` (direct import) | No child spans | Always call `litellm.acompletion(...)` on the module — never import the function directly |
| Calling `registerOTel()` when a provider already exists | Existing processors discarded, other observability breaks | Use `createLemmaSpanProcessor()` and add to the existing provider → `/tracing/patterns/dual-export.md` |
| `{ streaming: true }` without `ctx.complete()` | Span open indefinitely, trace never appears | Always pair `{ streaming: true }` with `ctx.complete(assembledText)` in all code paths → `/tracing/patterns/streaming.md` |
| Returning `undefined` / `None` from wrapped function | Run appears with blank output | Always return the final result explicitly |
| Registering OpenInference after client instantiation | No child spans — client holds unpatched reference | Register all instrumentors before creating any provider client |
| Returning stream object instead of consuming it | Output field shows stream object, not text | Consume inside the wrapper; call `ctx.complete()` with the assembled text → `/tracing/patterns/streaming.md` |

## When things go wrong

Before suggesting code changes, check in this order:

1. **No traces at all:** Is `registerOTel()` called before agent code runs? Are `LEMMA_API_KEY` and `LEMMA_PROJECT_ID` set? Does the process stay alive long enough to flush (serverless)?
2. **Traces but no child spans:** Is provider instrumentation registered? Registered against the correct provider (the one `registerOTel()` returned)? Are clients created after instrumentation?
3. **Empty output:** Does the wrapped function return a value explicitly? For streaming: is `ctx.complete()` called?
4. **Span never closes:** Is `ctx.complete()` called in both success and error paths?

→ Detailed steps: `https://docs.uselemma.ai/tracing/troubleshooting/common-problems.md`

**If the user has Lemma MCP connected**, query recent traces directly to confirm whether spans are arriving before suggesting any code changes. This is faster than adding debug logs.
→ MCP setup: `https://docs.uselemma.ai/connections/mcp.md`

**Enable debug mode** when the above checks don't resolve the issue — it logs every span start, end, and export event.
→ `https://docs.uselemma.ai/tracing/troubleshooting/debug-mode.md`

## Skill feedback

If the skill gives incorrect guidance, references a page that doesn't exist, or is missing a scenario you encountered — offer to submit feedback. See [references/skill-feedback.md](references/skill-feedback.md) for the process.

Do **not** trigger this for issues with Lemma itself (the product) — only for issues with this skill's instructions.