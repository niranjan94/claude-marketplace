# Tool Use

Guidance on prompting Claude to use tools effectively, including action bias, parallel execution, and proactive vs conservative modes.

## Table of contents

- [Explicit action instructions](#explicit-action-instructions)
- [Proactive vs conservative action](#proactive-vs-conservative-action)
- [Parallel tool calling](#parallel-tool-calling)
- [Overtriggering in Claude 4.6](#overtriggering-in-claude-46)

---

## Explicit action instructions

Claude's latest models follow instructions precisely. If you say "can you suggest some changes," Claude may provide suggestions rather than making changes, even if acting is what you intended.

Be explicit about whether you want action or advice:

Less effective (Claude will only suggest):
```
Can you suggest some changes to improve this function?
```

More effective (Claude will act):
```
Change this function to improve its performance.
```

Or:
```
Make these edits to the authentication flow.
```

## Proactive vs conservative action

### Proactive (default to action)

```xml
<default_to_action>
By default, implement changes rather than only suggesting them. If the user's intent is
unclear, infer the most useful likely action and proceed, using tools to discover any
missing details instead of guessing. Try to infer the user's intent about whether a tool
call (e.g., file edit or read) is intended or not, and act accordingly.
</default_to_action>
```

### Conservative (default to advice)

```xml
<do_not_act_before_instructions>
Do not jump into implementation or change files unless clearly instructed to make changes.
When the user's intent is ambiguous, default to providing information, doing research, and
providing recommendations rather than taking action. Only proceed with edits, modifications,
or implementations when the user explicitly requests them.
</do_not_act_before_instructions>
```

## Parallel tool calling

Claude's latest models excel at parallel tool execution -- running multiple speculative searches, reading several files at once, executing bash commands simultaneously.

### Maximize parallelism

```xml
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the tool calls,
make all of the independent tool calls in parallel. Prioritize calling tools simultaneously
whenever the actions can be done in parallel rather than sequentially. For example, when
reading 3 files, run 3 tool calls in parallel to read all 3 files into context at the same
time. Maximize use of parallel tool calls where possible to increase speed and efficiency.
However, if some tool calls depend on previous calls to inform dependent values like the
parameters, do NOT call these tools in parallel and instead call them sequentially. Never
use placeholders or guess missing parameters in tool calls.
</use_parallel_tool_calls>
```

### Reduce parallelism

```
Execute operations sequentially with brief pauses between each step to ensure stability.
```

## Overtriggering in Claude 4.6

Claude Opus 4.5 and Opus 4.6 are more responsive to system prompts than previous models. Prompts designed to reduce undertriggering on tools may now cause overtriggering.

The fix: dial back aggressive language.

- Instead of: "CRITICAL: You MUST use this tool when..."
- Use: "Use this tool when..."

Normal, conversational prompting works better with these models than forceful directives.
