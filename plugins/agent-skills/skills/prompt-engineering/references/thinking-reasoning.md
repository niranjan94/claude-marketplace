# Thinking and Reasoning

Guidance on configuring and steering Claude's thinking capabilities, including adaptive thinking, extended thinking, effort parameter, and overthinking control.

## Table of contents

- [Adaptive thinking (Claude 4.6)](#adaptive-thinking-claude-46)
- [Extended thinking (Sonnet 4.6, older models)](#extended-thinking-sonnet-46-older-models)
- [Effort parameter](#effort-parameter)
- [Controlling overthinking](#controlling-overthinking)
- [Prompting thinking behavior](#prompting-thinking-behavior)
- [Manual chain-of-thought](#manual-chain-of-thought)

---

## Adaptive thinking (Claude 4.6)

Claude Opus 4.6 uses adaptive thinking by default: `thinking: {type: "adaptive"}`. Claude dynamically decides when and how much to think based on two factors:

- **Effort parameter** -- higher effort elicits more thinking
- **Query complexity** -- harder queries trigger deeper reasoning

On easier queries that don't require thinking, the model responds directly. In internal evaluations, adaptive thinking reliably drives better performance than manual extended thinking.

```python
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # or max, medium, low
    messages=[{"role": "user", "content": "..."}],
)
```

The triggering behavior is promptable. If Claude thinks more often than you'd like:

```
Extended thinking adds latency and should only be used when it will meaningfully improve
answer quality -- typically for problems that require multi-step reasoning. When in doubt,
respond directly.
```

## Extended thinking (Sonnet 4.6, older models)

Claude Sonnet 4.6 supports both adaptive thinking and manual extended thinking with interleaved mode. Older models use manual thinking with `budget_tokens`.

For Sonnet 4.6, consider adaptive thinking for:
- Autonomous multi-step agents
- Computer use agents
- Bimodal workloads (mix of easy and hard tasks)

If adaptive thinking doesn't fit, manual extended thinking with interleaved mode remains supported.

### Migration from budget_tokens to adaptive

Before (extended thinking, older models):
```python
client.messages.create(
    model="claude-sonnet-4-5-20250929",
    max_tokens=64000,
    thinking={"type": "enabled", "budget_tokens": 32000},
    messages=[{"role": "user", "content": "..."}],
)
```

After (adaptive thinking):
```python
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "..."}],
)
```

If not using extended thinking, no changes needed. Thinking is off by default when `thinking` parameter is omitted.

## Effort parameter

Controls thinking depth for adaptive thinking:

| Level | Use case |
|-------|----------|
| `max` | Hardest problems, maximum quality |
| `high` | Complex tasks requiring deep reasoning (Sonnet 4.6 default) |
| `medium` | Most applications, good balance |
| `low` | High-volume or latency-sensitive workloads |

For Sonnet 4.6 specifically, switching from adaptive to extended thinking with a `budget_tokens` cap provides a hard ceiling on thinking costs while preserving quality.

## Controlling overthinking

Claude Opus 4.6 does significantly more upfront exploration than previous models, especially at higher effort settings. If this is undesirable:

- **Replace blanket defaults with targeted instructions.** Instead of "Default to using [tool]," use "Use [tool] when it would enhance your understanding of the problem."
- **Remove over-prompting.** Tools that undertriggered previously are likely to trigger appropriately now. "If in doubt, use [tool]" will cause overtriggering.
- **Use lower effort** as a fallback.
- **Add explicit commitment guidance:**

```
When deciding how to approach a problem, choose an approach and commit to it. Avoid
revisiting decisions unless you encounter new information that directly contradicts your
reasoning. If you're weighing two approaches, pick one and see it through.
```

## Prompting thinking behavior

- **Prefer general instructions over prescriptive steps.** "Think thoroughly" often produces better reasoning than a hand-written step-by-step plan.
- **Multishot examples work with thinking.** Use `<thinking>` tags inside few-shot examples to show the reasoning pattern.
- **Ask Claude to self-check.** "Before you finish, verify your answer against [test criteria]." Catches errors reliably for coding and math.
- **Guide interleaved thinking:**

```
After receiving tool results, carefully reflect on their quality and determine optimal next
steps before proceeding. Use your thinking to plan and iterate based on this new information,
and then take the best next action.
```

## Manual chain-of-thought

When thinking is disabled, you can still encourage step-by-step reasoning:

```
Think through this problem step by step. Use <thinking> tags for your reasoning and
<answer> tags for your final output.
```

Note: When extended thinking is disabled, Claude Opus 4.5 is particularly sensitive to the word "think" and its variants. Consider using alternatives like "consider," "evaluate," or "reason through."
