# Migration Guide

Model-specific guidance for migrating prompts between Claude versions, with a focus on Sonnet 4.5 to Sonnet 4.6 and general Claude 4.6 migration.

## Table of contents

- [General migration to Claude 4.6](#general-migration-to-claude-46)
- [Sonnet 4.5 to Sonnet 4.6](#sonnet-45-to-sonnet-46)
- [Vision capabilities](#vision-capabilities)
- [Frontend design](#frontend-design)

---

## General migration to Claude 4.6

Key changes when migrating from earlier generations:

1. **Be specific about desired behavior** -- describe exactly what you'd like in the output.

2. **Frame instructions with modifiers** -- instead of "Create an analytics dashboard", use "Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to create a fully-featured implementation."

3. **Request features explicitly** -- animations and interactive elements should be explicitly requested.

4. **Update thinking configuration** -- Claude 4.6 uses adaptive thinking (`thinking: {type: "adaptive"}`) instead of manual `budget_tokens`. Use the effort parameter to control depth.

5. **Migrate away from prefilled responses** -- deprecated in Claude 4.6. See `output-formatting.md` for alternatives.

6. **Tune anti-laziness prompting** -- previous prompts encouraging thoroughness or aggressive tool use may cause overtriggering. Dial back forceful language.

## Sonnet 4.5 to Sonnet 4.6

Sonnet 4.6 defaults to effort level `high` (Sonnet 4.5 had no effort parameter). You may experience higher latency with the default.

### Recommended effort settings

| Setting | Use case |
|---------|----------|
| `medium` | Most applications |
| `low` | High-volume or latency-sensitive workloads |

Set a large max output token budget (64k recommended) at medium or high effort to give room for thinking.

**When to use Opus 4.6 instead:** For the hardest, longest-horizon problems (large-scale migrations, deep research, extended autonomous work).

### If not using extended thinking

Continue without it. Explicitly set effort to the appropriate level. At `low` effort with thinking disabled, expect similar or better performance relative to Sonnet 4.5 with no extended thinking.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "disabled"},
    output_config={"effort": "low"},
    messages=[{"role": "user", "content": "..."}],
)
```

### If using extended thinking

Extended thinking continues to be supported with no changes needed. Keep thinking budget around 16k tokens -- most tasks don't use that much, but it provides headroom.

**For coding use cases** (agentic coding, tool-heavy workflows):

Start with `medium` effort. Reduce to `low` if latency is too high. Increase to `high` or switch to Opus 4.6 for higher intelligence.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "medium"},
    messages=[{"role": "user", "content": "..."}],
)
```

**For chat and non-coding use cases:**

Start with `low` effort. Increase to `medium` if more depth is needed.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "low"},
    messages=[{"role": "user", "content": "..."}],
)
```

### When to try adaptive thinking

Consider adaptive thinking instead of `budget_tokens` for:

- **Autonomous multi-step agents** -- coding agents, data analysis pipelines, bug finding. Start at `high` effort, scale down to `medium` if needed.
- **Computer use agents** -- best-in-class accuracy on computer use evaluations.
- **Bimodal workloads** -- mix of easy and hard tasks where adaptive skips thinking on simple queries.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "..."}],
)
```

## Vision capabilities

Claude Opus 4.5 and Opus 4.6 have improved vision, especially with multiple images. They can analyze videos by breaking them into frames.

Effective technique: give Claude a crop tool or skill. Testing shows consistent uplift when Claude can "zoom" into relevant regions of an image.

## Frontend design

Claude Opus 4.5 and 4.6 excel at building complex web applications with strong frontend design. Without guidance, they may default to generic patterns ("AI slop" aesthetic).

```xml
<frontend_aesthetics>
Avoid generic "AI slop" aesthetics. Make creative, distinctive frontends that surprise
and delight.

Focus on:
- Typography: Choose beautiful, unique fonts. Avoid Arial, Inter, Roboto, system fonts.
- Color & Theme: Commit to a cohesive aesthetic. CSS variables for consistency. Dominant
  colors with sharp accents. Draw from IDE themes and cultural aesthetics.
- Motion: CSS animations, micro-interactions. Staggered reveals on page load.
- Backgrounds: Atmosphere and depth. Layer gradients, geometric patterns, contextual effects.

Avoid: overused font families, purple gradients on white, predictable layouts, cookie-cutter
design. Vary between light/dark themes, different fonts, different aesthetics. Avoid
converging on common choices (Space Grotesk, etc.) across generations.
</frontend_aesthetics>
```
