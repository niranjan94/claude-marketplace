---
name: prompt-engineering
description: >
  Comprehensive guide to prompt engineering for Claude's latest models (Opus 4.6, Sonnet 4.6, Haiku 4.5).
  Use this skill whenever the user asks about writing, improving, or debugging prompts for the Claude API,
  building system prompts, optimizing tool use, configuring thinking/effort, structuring agentic systems,
  or migrating prompts between Claude model versions. Also trigger when the user mentions "prompt engineering",
  "system prompt", "few-shot examples", "prompt optimization", "Claude API prompting", "prompt best practices",
  or asks how to get better results from Claude -- even if they don't use those exact terms.
---

# Prompt Engineering for Claude

This skill covers prompt engineering techniques for Claude's latest models: Claude Opus 4.6, Claude Sonnet 4.6, and Claude Haiku 4.5. Use it to help users write, improve, debug, or migrate prompts for the Claude API.

## How to use this skill

The core principles below cover the most common prompting patterns. For deeper dives into specific topics, read the relevant reference file:

- `references/output-formatting.md` -- Communication style, markdown control, LaTeX, document creation, prefill migration
- `references/tool-use.md` -- Explicit action prompts, parallel tool calling, proactive vs conservative action
- `references/thinking-reasoning.md` -- Adaptive thinking, extended thinking, effort parameter, overthinking control
- `references/agentic-systems.md` -- Long-horizon reasoning, state tracking, subagent orchestration, safety, research patterns
- `references/migration-guide.md` -- Migrating from older Claude models, Sonnet 4.5 to 4.6, effort settings

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

### 4. Structure with XML tags

XML tags help Claude parse complex prompts unambiguously. Wrap each content type in its own tag (e.g., `<instructions>`, `<context>`, `<input>`) to reduce misinterpretation.

- Use consistent, descriptive tag names
- Nest tags when content has natural hierarchy (e.g., `<documents>` containing `<document index="n">`)

### 5. Give Claude a role

A role in the system prompt focuses behavior and tone:

```python
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
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
The assistant is Claude, created by Anthropic. The current model is Claude Opus 4.6.
```

For LLM-powered apps needing model strings:

```
When an LLM is needed, default to Claude Opus 4.6. The exact model string is claude-opus-4-6.
```

Available model IDs: `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5-20251001`.

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

```xml
Avoid over-engineering. Only make changes that are directly requested or clearly necessary.
Keep solutions simple and focused:
- Don't add features, refactor code, or make "improvements" beyond what was asked.
- Don't add docstrings, comments, or type annotations to code you didn't change.
- Don't add error handling for scenarios that can't happen.
- Don't create helpers or abstractions for one-time operations.
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

## Key behavioral notes for Claude 4.6

These are important behavioral differences in Claude 4.6 that affect prompt design:

1. **More concise by default** -- may skip verbal summaries after tool calls. If you want summaries, ask explicitly.
2. **Prefilled responses deprecated** -- no more prefilling the last assistant turn. Use structured outputs, XML tags, or direct instructions instead. See `references/output-formatting.md`.
3. **Adaptive thinking** -- uses `thinking: {type: "adaptive"}` instead of manual `budget_tokens`. Control depth via the `effort` parameter. See `references/thinking-reasoning.md`.
4. **More proactive tool use** -- prompts that previously encouraged aggressive tool use may now cause overtriggering. Dial back "MUST use this tool" language.
5. **Strong subagent instincts** -- may spawn subagents when a direct approach would suffice. Guide when subagents are/aren't warranted. See `references/agentic-systems.md`.
6. **Overengineering tendency** -- may add abstractions, extra files, or flexibility that wasn't requested. Use the minimize-overengineering prompt pattern above.

---

## Applying this skill

When helping a user with prompts:

1. **Understand their goal** -- what model, what task, what integration (API, Claude Code, chat)?
2. **Diagnose the issue** -- is the output too verbose? Wrong format? Not using tools? Hallucinating? Refusing?
3. **Apply the relevant principle** -- start with core principles, then check reference files for the specific domain.
4. **Show, don't just tell** -- provide a concrete rewritten prompt, not just abstract advice. Explain what changed and why.
5. **Consider the model** -- techniques differ between Opus 4.6, Sonnet 4.6, and Haiku 4.5. Check `references/migration-guide.md` for model-specific guidance.
