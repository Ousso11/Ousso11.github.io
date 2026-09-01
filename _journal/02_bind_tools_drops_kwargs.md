---
title: "max_tokens=20 Produced 200 Tokens — Only When Tools Were Attached"
collection: journal
permalink: /journal/bind-tools-drops-kwargs/
excerpt: "A per-call max_tokens was honoured on tool-free calls and silently ignored the moment a tool was attached. LangChain's bind_tools() strips the kwargs from a prior chat.bind(), so the value never reached the wire."
---

## Issue

Customers set a cap and did not get one:

```python
client.messages.create(model=..., max_tokens=20, tools=[search])
# → ~200 output tokens
```

Remove `tools=[...]` and the same call capped at 20 correctly. The same asymmetry applied to `temperature`, `top_p` — every per-call knob. Only tool-free calls honoured them.

That asymmetry is the whole clue, and it is easy to misread. The instinct is to suspect the tool-calling loop, or the model ignoring instructions, or a retry re-issuing the request. It was none of those. The parameters were never sent.

## Root Cause

Inside the SDK's agent engine, LLM kwargs were bound **post-hoc**:

```python
chat = init_chat_model(f"{provider}:{model}")
chat = chat.bind(max_tokens=20)      # ← bound here
agent = create_agent(chat, tools=tools)
```

When `tools` is non-empty, `create_agent` internally calls **`bind_tools(...)`** — and `bind_tools` **replaces** the kwargs from a prior `chat.bind(...)` rather than merging with them. The binding was silently discarded on its way through, so the request that reached the provider had no `max_tokens` field at all.

With no tools, `bind_tools` is never called, the binding survives, and the cap works. Hence the asymmetry.

## Solution

**Stop binding after construction. Bake the kwargs into the constructor**, which `bind_tools` does not touch:

```python
chat = init_chat_model(f"{provider}:{model}", **extra_kwargs)
```

That change forces a cache redesign, because the chat model was memoised by model name alone — and now two calls with the same model but different `max_tokens` are genuinely different objects:

- **Python:** `_get_or_build_chat(model_name, extra_kwargs=None)`, cache keyed by `(model_name, sorted_kwargs_tuple)`. Identical kwargs hit the cache; different kwargs build a fresh chat model.
- **TypeScript:** the same shape, keyed by `` `${modelName}::${stableStringify(extraKwargs)}` `` — with a `stableStringify` helper, because JSON key order is not guaranteed and a cache key that reorders is a cache that never hits.

`_Engine.run` no longer calls `chat.bind()` at all.

## Verification

A regression test that would have caught it, asserting the *combination* rather than either half:

```
test_anthropic_max_tokens_caps_output_WITH_TOOLS
→ max_tokens=20 with Tavily attached → output_tokens == 20   ✅ (live)
```

251 Python unit tests + 5 live integration tests, 199 TypeScript unit tests, ESM + CJS builds green. The pre-existing no-tools test stayed green throughout — which is exactly why the bug survived as long as it did.

## 💡 Takeaway

- **A builder API that replaces instead of merges is a trap.** `bind_tools` discarding a prior `bind` is defensible in isolation and lethal in composition.
- **Test the interaction, not the features.** There was full coverage of `max_tokens`, and full coverage of tools. The bug lived in the cell of the matrix nobody wrote a test for.
- **Constructor arguments survive things that post-hoc configuration does not.** When a value must not be lost, put it where it cannot be rebound.
