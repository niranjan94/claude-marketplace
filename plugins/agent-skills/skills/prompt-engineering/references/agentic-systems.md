# Agentic Systems

Guidance on building agentic systems with Claude, covering long-horizon reasoning, state tracking, subagent orchestration, safety, research, and common pitfalls.

## Table of contents

- [Long-horizon reasoning and state tracking](#long-horizon-reasoning-and-state-tracking)
- [Context awareness and multi-window workflows](#context-awareness-and-multi-window-workflows)
- [Multi-context window workflows](#multi-context-window-workflows)
- [State management best practices](#state-management-best-practices)
- [Balancing autonomy and safety](#balancing-autonomy-and-safety)
- [Research and information gathering](#research-and-information-gathering)
- [Subagent orchestration](#subagent-orchestration)
- [Chain complex prompts](#chain-complex-prompts)
- [Reducing file creation](#reducing-file-creation)
- [Preventing hard-coding and test-fixation](#preventing-hard-coding-and-test-fixation)
- [Minimizing hallucinations](#minimizing-hallucinations)

---

## Long-horizon reasoning and state tracking

Claude's latest models excel at long-horizon tasks with exceptional state tracking. Claude maintains orientation across extended sessions by focusing on incremental progress -- making steady advances on a few things at a time rather than attempting everything at once.

## Context awareness and multi-window workflows

Claude 4.6 and 4.5 models feature context awareness, enabling the model to track its remaining context window throughout a conversation. If using an agent harness that compacts context:

```
Your context window will be automatically compacted as it approaches its limit, allowing
you to continue working indefinitely from where you left off. Do not stop tasks early due
to token budget concerns. As you approach your token budget limit, save your current progress
and state to memory before the context window refreshes. Always be as persistent and
autonomous as possible and complete tasks fully.
```

The memory tool pairs naturally with context awareness for seamless context transitions.

## Multi-context window workflows

For tasks spanning multiple context windows:

1. **First context window is special** -- use it to set up a framework (write tests, create setup scripts), then iterate on a todo-list in future windows.

2. **Structured test tracking** -- ask Claude to create tests before starting work in a structured format (e.g., `tests.json`). Remind Claude: "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."

3. **Quality-of-life tools** -- encourage creation of setup scripts (e.g., `init.sh`) to gracefully start servers, run test suites, and linters. Prevents repeated work across windows.

4. **Starting fresh vs compacting** -- Claude's latest models are effective at discovering state from the local filesystem. Sometimes a fresh context window beats compaction. Be prescriptive about how to start:
   - "Call pwd; you can only read and write files in this directory."
   - "Review progress.txt, tests.json, and the git logs."
   - "Manually run a fundamental integration test before implementing new features."

5. **Verification tools** -- for long autonomous tasks, Claude needs to verify correctness without human feedback. Tools like Playwright MCP or computer use help for testing UIs.

6. **Encourage complete usage of context:**

```
This is a very long task, so plan your work clearly. Spend your entire output context
working on the task -- just make sure you don't run out of context with significant
uncommitted work. Continue working systematically until complete.
```

## State management best practices

- **Structured formats for state data** -- JSON for test results, task status, schema-dependent info
- **Unstructured text for progress notes** -- freeform notes for tracking general progress
- **Git for state tracking** -- provides a log of what's been done and checkpoints. Claude performs especially well using git across sessions.
- **Emphasize incremental progress** -- explicitly ask Claude to track progress and focus on incremental work

Example structured state:
```json
{
  "tests": [
    {"id": 1, "name": "authentication_flow", "status": "passing"},
    {"id": 2, "name": "user_management", "status": "failing"},
    {"id": 3, "name": "api_endpoints", "status": "not_started"}
  ],
  "total": 200, "passing": 150, "failing": 25, "not_started": 25
}
```

Example progress notes:
```
Session 3 progress:
- Fixed authentication token validation
- Updated user model to handle edge cases
- Next: investigate user_management test failures (test #2)
- Note: Do not remove tests as this could lead to missing functionality
```

## Balancing autonomy and safety

Without guidance, Claude Opus 4.6 may take hard-to-reverse actions (deleting files, force-pushing, posting to external services).

```
Consider the reversibility and potential impact of your actions. You are encouraged to take
local, reversible actions like editing files or running tests, but for actions that are hard
to reverse, affect shared systems, or could be destructive, ask the user before proceeding.

Examples of actions that warrant confirmation:
- Destructive operations: deleting files or branches, dropping database tables, rm -rf
- Hard to reverse operations: git push --force, git reset --hard, amending published commits
- Operations visible to others: pushing code, commenting on PRs/issues, sending messages,
  modifying shared infrastructure

When encountering obstacles, do not use destructive actions as a shortcut. For example, don't
bypass safety checks (e.g. --no-verify) or discard unfamiliar files that may be in-progress work.
```

## Research and information gathering

Claude's latest models demonstrate exceptional agentic search capabilities. For optimal research:

1. **Provide clear success criteria** -- define what constitutes a successful answer
2. **Encourage source verification** -- verify across multiple sources
3. **Structured approach for complex research:**

```
Search for this information in a structured way. As you gather data, develop several
competing hypotheses. Track your confidence levels in your progress notes. Regularly
self-critique your approach and plan. Update a hypothesis tree or research notes file.
Break down this complex research task systematically.
```

## Subagent orchestration

Claude 4.6 has significantly improved native subagent orchestration. It can recognize when tasks benefit from delegation and do so proactively.

To take advantage:
1. Have well-defined subagent tools available
2. Let Claude orchestrate naturally -- it will delegate without explicit instruction
3. Watch for overuse -- Claude may spawn subagents when a simpler approach suffices

If seeing excessive subagent use:

```
Use subagents when tasks can run in parallel, require isolated context, or involve
independent workstreams that don't need to share state. For simple tasks, sequential
operations, single-file edits, or tasks where you need to maintain context across steps,
work directly rather than delegating.
```

## Chain complex prompts

With adaptive thinking and subagent orchestration, Claude handles most multi-step reasoning internally. Explicit prompt chaining (breaking into sequential API calls) is still useful when you need to inspect intermediate outputs or enforce a specific pipeline.

Most common pattern: **self-correction** -- generate draft, review against criteria, refine. Each step as a separate API call for logging, evaluation, or branching.

## Reducing file creation

Claude may create temporary files as a "scratchpad" during iteration. To minimize:

```
If you create any temporary new files, scripts, or helper files for iteration, clean up
these files by removing them at the end of the task.
```

## Preventing hard-coding and test-fixation

Claude can focus too heavily on making tests pass at the expense of general solutions:

```
Write a high-quality, general-purpose solution using standard tools. Do not hard-code values
or create solutions that only work for specific test inputs. Implement the actual logic that
solves the problem generally. Tests verify correctness, not define the solution. If the task
is unreasonable or tests are incorrect, inform me rather than working around them.
```

## Minimizing hallucinations

```xml
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file,
you MUST read the file before answering. Investigate and read relevant files BEFORE answering
questions about the codebase. Never make claims about code before investigating.
</investigate_before_answering>
```
