# Thinking and Reasoning

Guidance on configuring and steering Claude's thinking capabilities, including adaptive thinking on Opus 4.7, the effort parameter (with the new `xhigh` level), thinking display modes, and overthinking control.

## Table of contents

- [Adaptive thinking on Opus 4.7](#adaptive-thinking-on-opus-47)
- [Adaptive thinking on Opus 4.6 / Sonnet 4.6](#adaptive-thinking-on-opus-46--sonnet-46)
- [Extended thinking with `budget_tokens` (deprecated / removed)](#extended-thinking-with-budget_tokens-deprecated--removed)
- [Effort parameter](#effort-parameter)
- [Thinking display (Opus 4.7)](#thinking-display-opus-47)
- [Controlling overthinking](#controlling-overthinking)
- [Prompting thinking behavior](#prompting-thinking-behavior)
- [Manual chain-of-thought](#manual-chain-of-thought)

---

## Adaptive thinking on Opus 4.7

On Opus 4.7, adaptive thinking is the **only supported thinking-on mode**. Manual extended thinking with `budget_tokens` returns a 400 error.

Two important defaults differ from Opus 4.6:

- **Adaptive thinking is OFF by default.** Requests with no `thinking` field run without thinking. You must explicitly set `thinking: {type: "adaptive"}` to enable it. (On 4.6, adaptive was the default when configured.)
- **Thinking content is omitted by default.** The thinking block still appears in the response stream but its `thinking` field is empty. Opt back in with `display: "summarized"` if you need to surface reasoning to users.

```python
client.messages.create(
    model="claude-opus-4-7",
    max_tokens=64000,
    thinking={"type": "adaptive", "display": "summarized"},
    output_config={"effort": "xhigh"},
    messages=[{"role": "user", "content": "..."}],
)
```

Adaptive thinking calibrates depth based on:
- The `effort` parameter (higher elicits more thinking)
- Query complexity (simpler queries skip thinking entirely)

In Anthropic's internal evaluations, adaptive thinking reliably outperforms manual extended thinking.

## Adaptive thinking on Opus 4.6 / Sonnet 4.6

On Opus 4.6 and Sonnet 4.6, adaptive thinking is also the recommended path:

```python
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "..."}],
)
```

On these models, manual extended thinking with `budget_tokens` is still functional but deprecated.

## Extended thinking with `budget_tokens` (deprecated / removed)

| Model | `budget_tokens` status |
|-------|------------------------|
| Opus 4.7 | **Removed.** Returns 400 error. |
| Opus 4.6 | Functional but deprecated. |
| Sonnet 4.6 | Functional but deprecated. |
| Sonnet 4.5 and earlier | The only thinking mode. |

If you need a hard ceiling on thinking costs while running adaptive thinking, prefer:
- Lowering the `effort` setting, or
- Using `max_tokens` as a hard per-request cap, or
- Setting an advisory `task_budget` (Opus 4.7 beta -- see `agentic-systems.md`)

## Effort parameter

Controls thinking depth and overall capability vs. cost/latency tradeoff for adaptive thinking.

| Level | Use case |
|-------|----------|
| `max` | Hardest intelligence-demanding tasks. Some risk of overthinking and diminishing returns vs. tokens spent. |
| `xhigh` | **New on Opus 4.7.** Recommended default for coding and agentic use cases. Sits between `high` and `max`. |
| `high` | Minimum recommended for most intelligence-sensitive use cases. Sonnet 4.6 default. |
| `medium` | Good balance for cost-sensitive applications that can trade off some intelligence. |
| `low` | Short, scoped, latency-sensitive tasks that aren't intelligence-sensitive. |

Effort matters more on Opus 4.7 than on any prior Opus. At `low` and `medium`, Opus 4.7 scopes work tightly to what was asked. On moderately complex tasks at `low`, this can manifest as under-thinking -- if you see shallow reasoning, raise effort rather than prompting around the literalism.

If you must keep effort at `low` for latency reasons, add targeted guidance:

```
This task involves multi-step reasoning. Think carefully through the problem before responding.
```

When running at `xhigh` or `max`, set a generous `max_tokens` (start at 64k) so the model has room to think and act across subagents and tool calls.

## Thinking display (Opus 4.7)

On Opus 4.7, the `display` field on the `thinking` config controls whether reasoning appears in the response:

```python
thinking = {
    "type": "adaptive",
    "display": "summarized",  # or "omitted" (default)
}
```

| `display` | Behavior |
|-----------|----------|
| `"omitted"` (default on 4.7) | Thinking blocks appear in stream but `thinking` field is empty. Slightly lower latency. |
| `"summarized"` | Summarized thinking text returned (matches the previous Opus 4.6 default). |

If your product streams reasoning to users, the new `"omitted"` default appears as a long pause before output begins. Set `display: "summarized"` to restore visible progress during thinking.

## Controlling overthinking

Opus 4.6 and 4.7 do significantly more upfront exploration than older models, especially at higher effort. If this is undesirable:

- **Replace blanket defaults with targeted instructions.** Instead of "Default to using [tool]," use "Use [tool] when it would enhance your understanding of the problem."
- **Remove over-prompting.** Tools that undertriggered on older models are likely to trigger appropriately now. "If in doubt, use [tool]" causes overtriggering.
- **Lower the effort setting** as a fallback.
- **Add explicit commitment guidance:**

```
When deciding how to approach a problem, choose an approach and commit to it. Avoid
revisiting decisions unless you encounter new information that directly contradicts your
reasoning. If you're weighing two approaches, pick one and see it through.
```

## Prompting thinking behavior

- **Prefer general instructions over prescriptive steps.** "Think thoroughly" often produces better reasoning than a hand-written step-by-step plan, since the model's reasoning frequently exceeds what a human would prescribe.
- **Multishot examples work with thinking.** Use `<thinking>` tags inside few-shot examples to show the reasoning pattern you want.
- **Ask Claude to self-check.** "Before you finish, verify your answer against [test criteria]." Catches errors reliably for coding and math.
- **Guide interleaved thinking after tool calls:**

```
After receiving tool results, carefully reflect on their quality and determine optimal next
steps before proceeding. Use your thinking to plan and iterate based on this new information,
and then take the best next action.
```

- **Raise effort instead of adding "think step by step".** On Opus 4.7, scaffolding instructions like "think step by step", "plan before acting", or "reason carefully before responding" were compensating for a reasoning gap that higher effort closes natively. Prefer raising effort to `high`/`xhigh` over adding such phrases.

## Manual chain-of-thought

When thinking is disabled (or running on a model without it), you can still encourage step-by-step reasoning:

```
Think through this problem step by step. Use <thinking> tags for your reasoning and
<answer> tags for your final output.
```

Note: When extended thinking is disabled, Opus 4.5 and 4.6 are particularly sensitive to the word "think" and its variants. Consider alternatives like "consider," "evaluate," or "reason through."
