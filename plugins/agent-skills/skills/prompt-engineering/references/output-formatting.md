# Output and Formatting

Detailed guidance on controlling Claude's output style, format, and structure.

## Table of contents

- [Communication style and verbosity](#communication-style-and-verbosity)
- [Controlling response format](#controlling-response-format)
- [LaTeX output](#latex-output)
- [Document creation](#document-creation)
- [Migrating away from prefilled responses](#migrating-away-from-prefilled-responses)

---

## Communication style and verbosity

Claude's latest models have a more concise and natural communication style:

- **More direct and grounded** -- fact-based progress reports rather than self-celebratory updates
- **More conversational** -- slightly more fluent and colloquial, less machine-like
- **Less verbose** -- may skip detailed summaries for efficiency unless prompted

Claude may skip verbal summaries after tool calls, jumping directly to the next action. If you prefer more visibility:

```
After completing a task that involves tool use, provide a quick summary of the work you've done.
```

## Controlling response format

Four effective techniques for steering output formatting:

### 1. Tell Claude what to do, not what to avoid

Instead of: "Do not use markdown in your response"
Try: "Your response should be composed of smoothly flowing prose paragraphs."

### 2. Use XML format indicators

"Write the prose sections of your response in `<smoothly_flowing_prose_paragraphs>` tags."

### 3. Match your prompt style to desired output

The formatting style in your prompt influences Claude's response style. Removing markdown from your prompt reduces markdown in the output.

### 4. Explicit formatting guidance

For detailed control over markdown usage:

```
<avoid_excessive_markdown_and_bullet_points>
When writing reports, documents, technical explanations, analyses, or any long-form content,
write in clear, flowing prose using complete paragraphs and sentences. Use standard paragraph
breaks for organization and reserve markdown primarily for `inline code`, code blocks, and
simple headings.

DO NOT use ordered lists or unordered lists unless: a) you're presenting truly discrete items
where a list format is the best option, or b) the user explicitly requests a list or ranking.

Instead of listing items with bullets or numbers, incorporate them naturally into sentences.
Your goal is readable, flowing text that guides the reader naturally through ideas rather than
fragmenting information into isolated points.
</avoid_excessive_markdown_and_bullet_points>
```

## LaTeX output

Claude Opus 4.6 defaults to LaTeX for mathematical expressions. For plain text output:

```
Format your response in plain text only. Do not use LaTeX, MathJax, or any markup notation
such as \( \), $, or \frac{}{}. Write all math expressions using standard text characters
(e.g., "/" for division, "*" for multiplication, and "^" for exponents).
```

## Document creation

Claude's latest models excel at creating presentations, animations, and visual documents with strong creative flair. For best results:

```
Create a professional presentation on [topic]. Include thoughtful design elements, visual
hierarchy, and engaging animations where appropriate.
```

## Migrating away from prefilled responses

Starting with Claude 4.6 models, prefilled responses on the last assistant turn are no longer supported. Here's how to migrate common use cases:

### Controlling output formatting

Previously: prefilling `{"` to force JSON output.

Now: Use the [Structured Outputs](https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs) feature, or simply ask Claude to output in your desired format. For classification, use tools with an enum field or structured outputs.

### Eliminating preambles

Previously: prefilling `Here is the requested summary:\n`

Now: "Respond directly without preamble. Do not start with phrases like 'Here is...', 'Based on...', etc." Or direct output into XML tags / use structured outputs.

### Avoiding bad refusals

Previously: prefilling to steer around unnecessary refusals.

Now: Claude is much better at appropriate refusals. Clear prompting in the user message should be sufficient.

### Continuations

Previously: prefilling with partial text to continue from.

Now: Move to the user message: "Your previous response was interrupted and ended with `[previous_response]`. Continue from where you left off." Or retry the request.

### Context hydration and role consistency

Previously: prefilled assistant reminders for context.

Now: Inject reminders into the user turn. For agentic systems, hydrate via tools or during context compaction.
