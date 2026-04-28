---
name: prompt-engineering
description: >
  Comprehensive guide to prompt engineering for Claude's latest models (Opus 4.7, Sonnet 4.6, Haiku 4.5).
  Use this skill whenever the user asks about writing, improving, or debugging prompts for the Claude API,
  building system prompts, optimizing tool use, configuring thinking/effort, structuring agentic systems,
  migrating from Opus 4.6 to Opus 4.7, or migrating prompts between any Claude model versions. Also trigger
  when the user mentions "prompt engineering", "system prompt", "few-shot examples", "prompt optimization",
  "Claude API prompting", "prompt best practices", "adaptive thinking", "effort", "task budgets",
  "xhigh effort", or asks how to get better results from Claude -- even if they don't use those exact terms.
---

# Prompt Engineering for Claude

This skill covers prompt engineering techniques for Claude's latest models: Claude Opus 4.7, Claude Sonnet 4.6, and Claude Haiku 4.5. Use it to help users write, improve, debug, or migrate prompts for the Claude API.

Opus 4.7 is the current most-capable Anthropic model and supersedes Opus 4.6. It introduces breaking API changes (extended thinking with `budget_tokens` removed, sampling parameters removed, thinking content omitted by default, new tokenizer) and meaningful behavior changes (more literal instruction following, verbosity calibration, fewer tool calls and subagents by default, more direct tone). For full migration steps see `references/migration-guide.md`.

## How to use this skill

The core principles below cover the most common prompting patterns. For deeper dives into specific topics, read the relevant reference file:

- `references/output-formatting.md` -- Communication style, verbosity calibration, markdown control, LaTeX, document creation, prefill migration
- `references/tool-use.md` -- Explicit action prompts, parallel tool calling, proactive vs conservative action, fewer-tools-by-default behavior on Opus 4.7
- `references/thinking-reasoning.md` -- Adaptive thinking on Opus 4.7 (only supported mode, off by default), effort levels including `xhigh`, `display: "summarized"`, overthinking control
- `references/agentic-systems.md` -- Long-horizon reasoning, state tracking, subagent orchestration (fewer subagents on 4.7), task budgets, memory, safety, research patterns, overengineering controls
- `references/migration-guide.md` -- Opus 4.6 to 4.7 migration (breaking + behavior changes), Opus 4.5 or earlier to 4.7, Sonnet 4.5 to 4.6

Read these on demand based on the user's question. Do not load all of them upfront.

---

## Core principles

### 1. Be clear and direct

Claude responds best to explicit instructions. Think of it as a brilliant new employee who lacks context on your norms. The more precisely you explain what you want, the better the result.

**Golden rule:** If a colleague with minimal context would be confused by your prompt, Claude will be too.

- Be specific about desired output format and constraints
- Use numbered lists or bullet points when order or completeness matters
- If you want "above and beyond" behavior, explicitly request it

**Example -- creating an analytics dashboard:**

Less effective:
```
Create an analytics dashboard
```

More effective:
```
Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to create a fully-featured implementation.
```

### 2. Add context and motivation

Explaining *why* behind your instructions helps Claude generalize and deliver more targeted responses.

Less effective:
```
NEVER use ellipses
```

More effective:
```
Your response will be read aloud by a text-to-speech engine, so never use ellipses since the text-to-speech engine will not know how to pronounce them.
```

Claude is smart enough to generalize from the explanation -- it will likely avoid other TTS-unfriendly punctuation too.

### 3. Use examples (few-shot prompting)

Examples are one of the most reliable ways to steer output format, tone, and structure. 3-5 well-crafted examples dramatically improve accuracy and consistency.

Make examples:
- **Relevant** -- mirror the actual use case
- **Diverse** -- cover edge cases; vary enough to avoid unintended pattern-matching
- **Structured** -- wrap in `<example>` tags (multiple in `<examples>`) so Claude distinguishes them from instructions

Tip: you can ask Claude to evaluate your examples for relevance and diversity, or to generate additional ones based on your initial set.

### 4. Structure with XML tags

XML tags help Claude parse complex prompts unambiguously. Wrap each content type in its own tag (e.g., `<instructions>`, `<context>`, `<input>`) to reduce misinterpretation.

- Use consistent, descriptive tag names
- Nest tags when content has natural hierarchy (e.g., `<documents>` containing `<document index="n">`)

### 5. Give Claude a role

A role in the system prompt focuses behavior and tone:

```python
client.messages.create(
    model="claude-opus-4-7",
    max_tokens=8192,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    system="You are a helpful coding assistant specializing in Python.",
    messages=[{"role": "user", "content": "How do I sort a list of dictionaries by key?"}],
)
```

### 6. Long context prompting (20k+ tokens)

When working with large documents:

- **Put longform data at the top** of your prompt, above queries and instructions. Queries at the end can improve response quality by up to 30%.
- **Wrap documents in XML** with metadata:
  ```xml
  <documents>
    <document index="1">
      <source>annual_report_2023.pdf</source>
      <document_content>{{ANNUAL_REPORT}}</document_content>
    </document>
  </documents>
  ```
- **Ground responses in quotes** -- ask Claude to quote relevant parts before answering. This cuts through noise in long documents.

### 7. Model self-knowledge

For apps that need Claude to identify itself or use specific model strings:

```
The assistant is Claude, created by Anthropic. The current model is Claude Opus 4.7.
```

For LLM-powered apps needing model strings:

```
When an LLM is needed, default to Claude Opus 4.7. The exact model string is claude-opus-4-7.
```

Available model IDs (current generation):
- `claude-opus-4-7` -- most capable, best for long-horizon agentic work, knowledge work, vision, memory tasks. 1M token context at standard pricing.
- `claude-sonnet-4-6` -- strong intelligence + fast performance, ideal for everyday coding/analysis/content
- `claude-haiku-4-5-20251001` -- fast, low-cost workloads
- `claude-opus-4-6` -- previous-generation Opus, still supported

Note that on Opus 4.7, `temperature`, `top_p`, and `top_k` return 400 errors if set to non-default values. Omit them and rely on prompting and `effort` to shape behavior.

---

## Quick-reference prompt patterns

These are copy-pasteable prompt snippets for common needs. For the full context and alternatives, see the relevant reference file.

### Encourage proactive action (tool use)

```xml
<default_to_action>
By default, implement changes rather than only suggesting them. If the user's intent is unclear,
infer the most useful likely action and proceed, using tools to discover any missing details
instead of guessing.
</default_to_action>
```

See `references/tool-use.md` for the conservative alternative.

### Maximize parallel tool calling

```xml
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the tool calls,
make all of the independent tool calls in parallel. Prioritize calling tools simultaneously
whenever the actions can be done in parallel rather than sequentially. Never use placeholders
or guess missing parameters in tool calls.
</use_parallel_tool_calls>
```

### Minimize overengineering

```
Avoid over-engineering. Only make changes that are directly requested or clearly necessary.
Keep solutions simple and focused:
- Scope: Don't add features, refactor code, or make "improvements" beyond what was asked.
- Documentation: Don't add docstrings, comments, or type annotations to code you didn't change.
- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios that
  can't happen. Only validate at system boundaries (user input, external APIs).
- Abstractions: Don't create helpers or abstractions for one-time operations. Don't design
  for hypothetical future requirements.
```

### Minimize hallucinations in agentic coding

```xml
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file,
you MUST read the file before answering. Investigate and read relevant files BEFORE
answering questions about the codebase. Never make claims about code before investigating.
</investigate_before_answering>
```

### Prevent hard-coding and test-fixation

```
Write a high-quality, general-purpose solution using standard tools. Do not hard-code values
or create solutions that only work for specific test inputs. Implement the actual logic that
solves the problem generally. If the task is unreasonable or tests are incorrect, inform me
rather than working around them.
```

### Guide thinking behavior

```
After receiving tool results, carefully reflect on their quality and determine optimal next
steps before proceeding. Use your thinking to plan and iterate based on this new information.
```

To reduce excessive thinking:
```
Extended thinking adds latency and should only be used when it will meaningfully improve
answer quality -- typically for problems that require multi-step reasoning. When in doubt,
respond directly.
```

See `references/thinking-reasoning.md` for adaptive thinking configuration and effort parameter guidance.

### Autonomy and safety balance

```
Consider the reversibility and potential impact of your actions. Take local, reversible
actions freely, but for actions that are hard to reverse, affect shared systems, or could
be destructive, ask the user before proceeding.
```

See `references/agentic-systems.md` for the full version with examples.

### Context window management (agentic systems)

```
Your context window will be automatically compacted as it approaches its limit, allowing
you to continue working indefinitely. Do not stop tasks early due to token budget concerns.
Save progress and state before the context window refreshes. Be as persistent and autonomous
as possible.
```

### Frontend design quality

```xml
<frontend_aesthetics>
Avoid generic "AI slop" aesthetics. Make creative, distinctive frontends.
Focus on: typography (avoid generic fonts), color (commit to a cohesive aesthetic),
motion (CSS animations, staggered reveals), backgrounds (gradients, patterns, depth).
Interpret creatively and make unexpected choices.
</frontend_aesthetics>
```

---

## Key behavioral notes for Claude Opus 4.7

These are the behavioral differences in Opus 4.7 (vs. Opus 4.6) that most often require prompt updates:

1. **More literal instruction following** -- 4.7 will not silently generalize an instruction from one item to another, and will not infer requests you didn't make. Vague first-turn prompts produce conservative, incomplete work. Be specific about intent, constraints, acceptance criteria, and (for coding) file paths. Effect is strongest at `low` and `medium` effort.

2. **Verbosity calibrates to task complexity** -- shorter answers on simple lookups, longer ones on open-ended analysis. If your product depends on a particular length, instruct it explicitly. Positive examples of the desired voice work better than "don't do X" instructions.

3. **More direct, opinionated tone** -- less validation-forward phrasing and fewer emoji than Opus 4.6's warmer style. Add explicit tone directives if you need warmth.

4. **Fewer tool calls by default** -- 4.7 uses reasoning more, tools less. To increase tool usage, raise `effort` to `high` or `xhigh`, or add explicit instructions about when to call tools.

5. **Fewer subagents by default** -- 4.7 is more judicious. If your workflow benefits from parallel subagents, spell out when delegation is desired. See `references/agentic-systems.md`.

6. **Stricter effort calibration** -- at `low`/`medium`, 4.7 scopes work tightly to what was asked. On moderately complex tasks at `low` there's a risk of under-thinking; raise effort to `high` or `xhigh` rather than prompting around it.

7. **Built-in progress updates** -- 4.7 emits more regular status updates during long agentic traces. If you previously added scaffolding like "After every 3 tool calls, summarize progress", try removing it.

8. **Adaptive thinking is OFF by default** -- on Opus 4.7, requests with no `thinking` field run without thinking (different from Opus 4.6). Set `thinking: {type: "adaptive"}` explicitly to enable. `budget_tokens` returns a 400 error.

9. **Thinking content omitted by default** -- thinking blocks appear in the response stream but the `thinking` field is empty. To surface reasoning to users, set `thinking.display: "summarized"`.

10. **New tokenizer** -- same text uses ~1.0x to 1.35x more tokens than Opus 4.6. Bump `max_tokens` and re-baseline with `count_tokens()`.

11. **Overengineering tendency (carries over from 4.6)** -- may add abstractions, extra files, or flexibility that wasn't requested. Use the minimize-overengineering prompt pattern above.

12. **Real-time cybersecurity safeguards** -- requests touching prohibited or high-risk topics may now refuse. Legitimate security work can apply to Anthropic's Cyber Verification Program.

---

## Applying this skill

When helping a user with prompts:

1. **Understand their goal** -- what model, what task, what integration (API, Claude Code, chat)?
2. **Diagnose the issue** -- is the output too verbose? Wrong format? Not using tools? Hallucinating? Refusing?
3. **Apply the relevant principle** -- start with core principles, then check reference files for the specific domain.
4. **Show, don't just tell** -- provide a concrete rewritten prompt, not just abstract advice. Explain what changed and why.
5. **Consider the model** -- techniques differ between Opus 4.7, Opus 4.6, Sonnet 4.6, and Haiku 4.5. Check `references/migration-guide.md` for model-specific guidance, especially when migrating to 4.7 (breaking API changes apply).
