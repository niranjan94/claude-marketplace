# Migration Guide

Model-specific guidance for migrating prompts and code between Claude versions. The 4.6 → 4.7 path is the most consequential -- it includes breaking API changes.

## Table of contents

- [Opus 4.6 to Opus 4.7](#opus-46-to-opus-47)
  - [Breaking API changes](#breaking-api-changes)
  - [Behavior changes](#behavior-changes)
  - [Choosing an effort level on 4.7](#choosing-an-effort-level-on-47)
  - [Recommended changes (not required)](#recommended-changes-not-required)
  - [Migration checklist](#migration-checklist)
- [Opus 4.5 (or earlier) directly to Opus 4.7](#opus-45-or-earlier-directly-to-opus-47)
- [Sonnet 4.5 to Sonnet 4.6](#sonnet-45-to-sonnet-46)
- [Vision capabilities](#vision-capabilities)
- [Frontend design](#frontend-design)

---

## Opus 4.6 to Opus 4.7

Opus 4.7 should perform well on existing 4.6 prompts, at the same `$5 / $25` per MTok pricing, but several breaking changes and behavior shifts mean a tuning pass usually pays off. Same feature set as 4.6: 1M context window at standard pricing, 128k max output, adaptive thinking, prompt caching, batch processing, Files API, PDF support, vision, and the full server-side / client-side tool set.

```python
# Before
model = "claude-opus-4-6"
# After
model = "claude-opus-4-7"
```

### Breaking API changes

These apply to the Messages API. Claude Managed Agents has no breaking changes beyond the model ID.

**1. Extended thinking with `budget_tokens` removed.** Adaptive thinking is the only thinking-on mode on 4.7. `thinking: {type: "enabled", budget_tokens: N}` returns a 400 error.

```python
# Before (Opus 4.6)
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "enabled", "budget_tokens": 32000},
    messages=[...],
)

# After (Opus 4.7)
client.messages.create(
    model="claude-opus-4-7",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "xhigh"},
    messages=[...],
)
```

Adaptive thinking is **off by default** on Opus 4.7 -- requests with no `thinking` field run without thinking. Set `thinking: {type: "adaptive"}` explicitly to enable it.

**2. Sampling parameters removed.** `temperature`, `top_p`, and `top_k` return a 400 error on 4.7 if set to non-default values. Omit them and use prompting + `effort` to shape behavior. (Note: `temperature = 0` never guaranteed identical outputs anyway.)

**3. Thinking content omitted by default.** Thinking blocks still appear in the response stream but their `thinking` field is empty unless the caller opts in. Restore visible reasoning with:

```python
thinking = {"type": "adaptive", "display": "summarized"}
```

If your UI streams reasoning to users, the new default looks like a long pause before output begins.

**4. New tokenizer.** The same text uses ~1.0x to 1.35x more tokens on 4.7 than 4.6 (up to ~35% more, varying by content). `/v1/messages/count_tokens` returns different numbers. Bump `max_tokens` to give headroom (especially for compaction triggers), and re-test any client-side token estimates.

**5. Prefill removed (carried over from 4.6).** Prefilling assistant messages returns 400. Use structured outputs, system prompt instructions, or `output_config.format`.

### Behavior changes

These don't break the API but usually require prompt tuning:

1. **More literal instruction following.** 4.7 will not silently generalize an instruction from one item to another, and will not infer requests you didn't make. Especially pronounced at `low`/`medium` effort. Vague first-turn prompts → conservative, incomplete work. Be specific about intent, constraints, acceptance criteria, and file paths.

2. **Verbosity calibrates to task complexity.** Shorter answers on simple lookups, much longer on open-ended analysis. To reduce verbosity:
   ```
   Provide concise, focused responses. Skip non-essential context, and keep examples minimal.
   ```
   Positive examples of the desired voice work better than "don't do X" instructions.

3. **More direct, opinionated tone.** Less validation-forward phrasing, fewer emoji than 4.6's warmer style. Re-evaluate style prompts; add explicit tone directives if you need warmth.

4. **Built-in progress updates in agentic traces.** 4.7 emits more regular status updates without scaffolding. Try removing prompts like "After every 3 tool calls, summarize progress." If 4.7's update style doesn't match your use case, describe what you want explicitly with examples.

5. **Fewer subagents by default.** 4.7 is more judicious about delegation. If your workflow benefits from parallel subagents (fanning across files, independent items), spell that out:
   ```
   When you have N independent items to process, fan them out across N subagents in parallel.
   ```

6. **Stricter effort calibration.** 4.7 respects `effort` levels strictly, especially at the low end. At `low`/`medium`, work scopes tightly to what was asked. On moderately complex tasks at `low`, watch for under-thinking -- raise effort to `high`/`xhigh` rather than prompting around it.

7. **Fewer tool calls by default.** 4.7 uses reasoning more, tools less. Raise effort to increase tool usage; or add explicit instructions about when to call which tools.

8. **Real-time cybersecurity safeguards.** Newly added in 4.7. Requests touching prohibited or high-risk topics may refuse. Legitimate security work (pentesting, vulnerability research, red-teaming) can apply to Anthropic's Cyber Verification Program.

9. **High-resolution image support.** First Claude model to support up to 2576px / 3.75MP (up from 1568px / 1.15MP). Coordinate output is 1:1 with image pixels (no scale factor). Full-resolution images can use ~3x more image tokens (up to 4,784 per image vs. ~1,600). Re-budget `max_tokens` for image-heavy workloads, or downsample if you don't need the fidelity.

### Choosing an effort level on 4.7

Effort matters more on Opus 4.7 than on any prior Opus. Recommended starting points:

- **`xhigh`** (new on 4.7) -- best default for coding and agentic use cases.
- **`high`** -- minimum for most intelligence-sensitive applications.
- **`medium`** -- cost-sensitive applications willing to trade off some intelligence.
- **`low`** -- short, scoped, latency-sensitive tasks that aren't intelligence-sensitive.
- **`max`** -- hardest intelligence-demanding tasks. Watch for diminishing returns and overthinking.

When running at `xhigh` or `max`, set `max_tokens` to at least 64k so the model has room across thinking, tool calls, and subagents.

### Recommended changes (not required)

- **Re-evaluate `max_tokens`.** Same text → more tokens on 4.7. Bump headroom, including compaction triggers.
- **Audit token-count assumptions.** Any code path estimating tokens client-side should re-test with `count_tokens()` on 4.7.
- **Adopt task budgets (beta).** Advisory token cap across the full agentic loop; the model sees a running countdown and paces itself. Set the beta header `task-budgets-2026-03-13` and add to `output_config`:
  ```python
  output_config = {
      "effort": "xhigh",
      "task_budget": {"type": "tokens", "total": 128000},
  }
  ```
  Minimum 20k tokens. Don't use for open-ended quality-over-speed tasks. Differs from `max_tokens`: `task_budget` is advisory and the model is aware of it; `max_tokens` is a hard ceiling not visible to the model.
- **Remove now-superfluous scaffolding.** Drop "double-check the slide layout before returning" / "verify your output before responding" patterns -- 4.7 is better at this natively. Re-baseline first.

### Migration checklist

- [ ] Update model ID `claude-opus-4-6` → `claude-opus-4-7`.
- [ ] Remove `temperature`, `top_p`, `top_k` from request payloads.
- [ ] Replace `thinking: {type: "enabled", budget_tokens: N}` with `thinking: {type: "adaptive"}` plus `output_config: {effort: ...}`.
- [ ] Remove any assistant-message prefills.
- [ ] If your UI displays thinking content, set `thinking.display: "summarized"`.
- [ ] Re-tune `max_tokens` for the new tokenizer (bump headroom, including compaction triggers).
- [ ] Re-test any client-side token estimation.
- [ ] If using `xhigh`/`max` effort, raise `max_tokens` to at least 64k.
- [ ] Re-budget for high-resolution image tokens (or downsample) if you send images.
- [ ] If consuming pointing/bounding-box coordinates, remove scale-factor conversion (1:1 with pixels on 4.7).
- [ ] Re-baseline response length with existing length-control prompts removed, then tune explicitly.
- [ ] Audit prompts for the behavior changes above (literalism, verbosity, tone, progress updates, subagents, effort, tool triggering).
- [ ] Consider adopting task budgets (beta) for agentic workflows.
- [ ] If your product does legitimate security work, apply to the Cyber Verification Program.

---

## Opus 4.5 (or earlier) directly to Opus 4.7

Apply all the 4.6 → 4.7 changes above plus these cumulative items:

1. **Remove now-GA beta headers:**
   - `effort-2025-11-24` -- effort is GA.
   - `fine-grained-tool-streaming-2025-05-14` -- GA.
   - `interleaved-thinking-2025-05-14` -- adaptive thinking enables interleaved thinking automatically on 4.6 and 4.7.

2. **Move from `client.beta.messages.create` to `client.messages.create`** for adaptive thinking and effort (now GA, no beta SDK namespace needed).

3. **`output_format` → `output_config.format`** for structured outputs. The old field still works but is deprecated.

4. **Tool parameter JSON escaping may differ slightly** in 4.6+ models (e.g., Unicode/forward-slash handling). Standard JSON parsers handle this; only an issue if you parse `tool_use.input` as a raw string.

If migrating from Claude 3.x or 4.1:
- Update to latest tool versions: `text_editor_20250728` / `str_replace_based_edit_tool`; `code_execution_20250825`.
- Remove any code using the `undo_edit` command.
- Handle the `refusal` stop reason.
- Handle the `model_context_window_exceeded` stop reason (Claude 4.5+).
- Verify tool parameter handling preserves trailing newlines (Claude 4.5+ stops stripping them).
- Remove legacy beta headers `token-efficient-tools-2025-02-19` and `output-128k-2025-02-19` (now built-in).

---

## Sonnet 4.5 to Sonnet 4.6

Sonnet 4.6 defaults to `effort: high` (Sonnet 4.5 had no effort parameter). You may experience higher latency with the default if you don't set it explicitly.

### Recommended effort settings

| Setting | Use case |
|---------|----------|
| `medium` | Most applications |
| `low` | High-volume or latency-sensitive workloads |

Set a large `max_tokens` budget (64k recommended) at medium or high effort to give room for thinking.

**When to use Opus 4.7 instead:** For the hardest, longest-horizon problems (large-scale migrations, deep research, extended autonomous work).

### If you're not using extended thinking

Continue without it. Set effort explicitly. At `low` effort with thinking disabled, expect similar or better performance vs. Sonnet 4.5 with no extended thinking.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "disabled"},
    output_config={"effort": "low"},
    messages=[...],
)
```

### If you're using extended thinking

`budget_tokens` is functional on Sonnet 4.6 but **deprecated**. Migrate to adaptive thinking with the effort parameter.

### Migrating to adaptive thinking

Adaptive thinking is the recommended target. Particularly suited to:

- **Autonomous multi-step agents** (coding agents, data analysis pipelines, bug finding) -- start at `high` effort; scale down to `medium` if needed.
- **Computer use agents** -- best-in-class accuracy in Anthropic's evals using adaptive mode.
- **Bimodal workloads** -- mix of easy and hard tasks where adaptive skips thinking on easy ones.

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    messages=[...],
)
```

### Keeping `budget_tokens` during migration (transitional)

If you need to keep `budget_tokens` temporarily while migrating, ~16k provides headroom. This is deprecated; don't make it long-term.

For coding workflows, start with `medium` effort:

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "medium"},
    messages=[...],
)
```

For chat / non-coding, start with `low` effort:

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "enabled", "budget_tokens": 16384},
    output_config={"effort": "low"},
    messages=[...],
)
```

---

## Vision capabilities

Opus 4.7 introduces high-resolution image support: up to 2576px on the long edge / 3.75MP (up from 1568px / 1.15MP). Coordinates returned by the model (pointing, bounding boxes) are 1:1 with actual image pixels -- no scale-factor math required. Improvements also apply to low-level perception (pointing, measuring, counting) and image localization.

High-res images can use ~3x more image tokens (up to ~4,784 per image, vs. ~1,600 previously). If you don't need the fidelity, downsample before sending.

Effective technique across all 4.x models: give Claude a crop tool or skill so it can "zoom" into relevant regions of an image. Anthropic publishes a cookbook for this.

---

## Frontend design

Claude Opus 4.5, 4.6, and 4.7 excel at building complex web applications with strong frontend design. Without guidance, they may default to generic patterns ("AI slop" aesthetic).

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
