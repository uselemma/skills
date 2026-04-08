# Manual Instrumentation — Integration Notes

Targets `@uselemma/tracing` >= 3.0.0.

Docs: `https://docs.uselemma.ai/tracing/manual-instrumentation.md`

---

## How helpers actually work

`tool()`, `llm()`, `retrieval()`, `trace()` are **function wrappers**, not decorators (in TypeScript). They take a name and a function, and return a new callable. The returned callable accepts one input argument.

```typescript
// tool() returns a new function
const lookupOrder = tool("lookup-order", async (orderId: string) => {
  return db.orders.findById(orderId);
});

// Call the returned function
const order = await lookupOrder("123"); // span: tool.lookup-order
```

In Python, the same helpers work as decorators (`@tool`) **or** as callables (`tool("name", fn)`). Both forms produce identical spans.

---

## One input argument in TypeScript

The TS helpers pass a single value to the wrapped function. If you need multiple arguments, wrap them in an object:

```typescript
// Wrong — fn receives only the first argument
const search = retrieval("search", async (query: string, topK: number) => { ... });

// Correct — use an options object
const search = retrieval("search", async ({ query, topK }: { query: string; topK: number }) => {
  return vectorDB.search(query, { topK });
});

const results = await search({ query: "...", topK: 5 });
```

Python decorators don't have this limitation — decorated functions keep their full signature.

---

## Nesting is automatic — don't pass `ctx`

Helpers nest under the currently active OTel context. When called from inside an `agent()` wrapper, they become children of the agent span automatically. You don't pass `ctx` or any parent reference.

```typescript
const myAgent = agent("my-agent", async (input: string) => {
  // ctx is available here but you never pass it to helpers
  const result = await lookupOrder(input.orderId); // auto-nests under agent span
  return result;
});
```

If a helper is called **outside** an `agent()` wrapper, it still creates a span — but it has no parent and appears as a top-level trace, not a child. This is usually unintentional.

---

## `agent()` always starts a new trace root

From the source: `agent()` starts its span from `ROOT_CONTEXT`, not from the currently active context. This is intentional — each agent run is its own trace.

Consequence: **nested `agent()` calls create sibling traces, not parent-child spans.** If `agentA` calls `agentB` (which is wrapped with `agent()`), you get two separate traces in Lemma, not one trace with a nested agent span. Use `tool()` or `trace()` for sub-operations you want to appear nested.

---

## Error handling is automatic

All helpers call `span.recordException()` and set `SpanStatusCode.ERROR` automatically on throw. You don't need to wrap helper calls in try/catch just to record errors on the span — the helper handles it.

```typescript
// No try/catch needed just for span error recording
const order = await lookupOrder(orderId);
// If lookupOrder throws, the span is ended with ERROR status automatically
```

You still need try/catch if you want to handle the error in your own code.

---

## `ctx.complete()` is optional in non-streaming mode

The wrapper auto-captures the return value as `ai.agent.output` and closes the span when the function returns. Only call `ctx.complete()` explicitly if:

- You need to record a **different** value than the return value
- You need to close the span **before** the function returns

Calling it unnecessarily (once before return, once on return) is harmless — `complete()` is idempotent; subsequent calls are no-ops.

---

## Span name prefixes

| Helper | Span name |
|---|---|
| `trace("my-step")` | `my-step` |
| `tool("lookup-order")` | `tool.lookup-order` |
| `llm("gpt-4o")` | `llm.gpt-4o` |
| `retrieval("vector-search")` | `retrieval.vector-search` |

Python bare decorator uses the function name:
```python
@tool  # span: tool.lookup_order (underscore from function name)
async def lookup_order(order_id: str) -> dict: ...

@tool("lookup-order")  # span: tool.lookup-order (explicit hyphen)
async def lookup_order(order_id: str) -> dict: ...
```

---

## Raw `startActiveSpan` — when to use it

Use raw OTel spans only when you need explicit control over attributes the helpers don't expose:

- `llm.model.requested`, `llm.tokens.prompt_uncached`, `llm.finish_reason`
- `tool.args`, `tool.result`

For everything else, use the typed helpers — they handle span lifecycle, error recording, and context propagation automatically. Raw spans require manual `span.end()` which is easy to forget in error paths.

```typescript
// Raw span — must end manually in all code paths
tracer.startActiveSpan("llm.step", async (span) => {
  try {
    const response = await llmCall(input);
    span.setAttribute("llm.model.requested", "gpt-4o");
    span.end(); // must call in success path
    return response;
  } catch (err) {
    span.recordException(err as Error);
    span.setStatus({ code: 2 });
    span.end(); // must call in error path too
    throw err;
  }
});
```
