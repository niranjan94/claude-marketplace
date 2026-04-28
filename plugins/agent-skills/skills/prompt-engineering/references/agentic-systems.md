# Agentic Systems

Guidance on building agentic systems with Claude, covering long-horizon reasoning, state tracking, subagent orchestration, safety, research, and common pitfalls.

## Table of contents

- [Long-horizon reasoning and state tracking](#long-horizon-reasoning-and-state-tracking)
- [Literal instruction following on Opus 4.7](#literal-instruction-following-on-opus-47)
- [Task budgets (Opus 4.7 beta)](#task-budgets-opus-47-beta)
- [Memory tool and filesystem-based memory](#memory-tool-and-filesystem-based-memory)
- [Context awareness and multi-window workflows](#context-awareness-and-multi-window-workflows)
- [Multi-context window workflows](#multi-context-window-workflows)
- [State management best practices](#state-management-best-practices)
- [Balancing autonomy and safety](#balancing-autonomy-and-safety)
- [Research and information gathering](#research-and-information-gathering)
- [Subagent orchestration](#subagent-orchestration)
- [Chain complex prompts](#chain-complex-prompts)
- [Reducing file creation](#reducing-file-creation)
- [Overeagerness and overengineering](#overeagerness-and-overengineering)
- [Preventing hard-coding and test-fixation](#preventing-hard-coding-and-test-fixation)
- [Minimizing hallucinations](#minimizing-hallucinations)

---

## Long-horizon reasoning and state tracking

Claude's latest models excel at long-horizon tasks with exceptional state tracking. Claude maintains orientation across extended sessions by focusing on incremental progress -- making steady advances on a few things at a time rather than attempting everything at once. Opus 4.7 is the strongest of the family on long-horizon agentic work, knowledge work, vision, and memory tasks.

## Literal instruction following on Opus 4.7

Opus 4.7 interprets prompts more literally than 4.6, especially at lower effort levels. It will not silently generalize an instruction from one item to another, and will not infer requests you didn't make.

The upside is precision -- less thrash, more predictable behavior in pipelines and structured extraction. The cost is that vague or sloppy prompts that "worked" on 4.6 by relying on the model to fill in intent now produce conservative, incomplete output.

Practical implications for agent prompts:

- **Specify intent, constraints, acceptance criteria, and (for coding) file paths upfront.** Don't expect 4.7 to figure out what "fix the bug" means from context.
- **First-turn detail compounds over a long trace.** A specific first turn → full implementation in one shot. A vague first turn → conservative work that requires several follow-ups.
- **Audit pipelines that relied on 4.6's implicit intent-filling.** Anywhere your harness depended on phrases like "do whatever makes sense" is a likely tuning target.

## Task budgets (Opus 4.7 beta)

Opus 4.7 introduces task budgets: an advisory token allowance for the full agentic loop (thinking + tool calls + tool results + final output). The model sees a running countdown and uses it to prioritize and finish gracefully as the budget is consumed.

Set the beta header `task-budgets-2026-03-13` and add to `output_config`:

```python
response = client.beta.messages.create(
    model="claude-opus-4-7",
    max_tokens=128000,
    output_config={
        "effort": "high",
        "task_budget": {"type": "tokens", "total": 128000},
    },
    messages=[{"role": "user", "content": "Review the codebase and propose a refactor plan."}],
    betas=["task-budgets-2026-03-13"],
)
```

Notes:
- Minimum task budget is 20k tokens.
- A task budget is **advisory**, not a hard cap -- the model is aware of it and paces itself, but won't hard-stop.
- `max_tokens` is the hard per-request ceiling. The model is not aware of `max_tokens`. Use `task_budget` for self-moderation, `max_tokens` to cap actual usage.
- Don't set a task budget for open-ended quality-over-speed work. If the budget is too restrictive, the model may complete the task less thoroughly or refuse it entirely.
- Reserve task budgets for workloads where you genuinely need the model to scope to a token allowance.

## Memory tool and filesystem-based memory

Opus 4.7 is meaningfully better at writing and using filesystem-based memory than 4.6. If your agent maintains a scratchpad, notes file, or structured memory store across turns, expect improvements at both note-taking and using prior notes.

Two paths:
- **Anthropic's client-side memory tool** -- managed scratchpad without building your own. Pairs naturally with context awareness.
- **Roll your own** -- structured `tests.json` / `progress.txt` / git commits in the agent's working directory. Claude is good at discovering this state from the filesystem when starting a fresh context window.

Example prompt for a fresh-window agent:
```
Call pwd; you can only read and write files in this directory. Review progress.txt,
tests.json, and the git log. Manually run a fundamental integration test before
implementing new features.
```

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

The picture changed between 4.6 and 4.7:

- **Opus 4.6** has strong native subagent instincts and may *over*-spawn subagents -- including for tasks where a direct approach (e.g., a single grep) would be faster.
- **Opus 4.7** is more judicious by default and spawns *fewer* subagents than 4.6.

This means the prompting recommendation is now bidirectional depending on the model:

**On 4.6 (curb overuse):**

```
Use subagents when tasks can run in parallel, require isolated context, or involve
independent workstreams that don't need to share state. For simple tasks, sequential
operations, single-file edits, or tasks where you need to maintain context across steps,
work directly rather than delegating.
```

**On 4.7 (encourage when appropriate):**

If your workflow benefits from parallel subagents -- fanning out across files, processing independent items, parallel research threads -- spell that out:

```
When you have N independent items to process (files, URLs, search queries), fan them out
across N subagents in parallel rather than processing sequentially. Subagents are
appropriate for: independent file edits, parallel research, isolated computation, and
tasks that don't need to share state.
```

In both cases the goal is calibration: get the model to delegate when delegation is faster, and not when it isn't.

## Chain complex prompts

With adaptive thinking and subagent orchestration, Claude handles most multi-step reasoning internally. Explicit prompt chaining (breaking into sequential API calls) is still useful when you need to inspect intermediate outputs or enforce a specific pipeline.

Most common pattern: **self-correction** -- generate draft, review against criteria, refine. Each step as a separate API call for logging, evaluation, or branching.

## Reducing file creation

Claude may create temporary files as a "scratchpad" during iteration. To minimize:

```
If you create any temporary new files, scripts, or helper files for iteration, clean up
these files by removing them at the end of the task.
```

## Overeagerness and overengineering

Claude Opus 4.5 and Opus 4.6 have a tendency to overengineer -- creating extra files, adding unnecessary abstractions, or building flexibility that wasn't requested. Add specific guidance to keep solutions minimal:

```
Avoid over-engineering. Only make changes that are directly requested or clearly necessary.
Keep solutions simple and focused:

- Scope: Don't add features, refactor code, or make "improvements" beyond what was asked.
  A bug fix doesn't need surrounding code cleaned up. A simple feature doesn't need extra
  configurability.

- Documentation: Don't add docstrings, comments, or type annotations to code you didn't
  change. Only add comments where the logic isn't self-evident.

- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios that
  can't happen. Trust internal code and framework guarantees. Only validate at system
  boundaries (user input, external APIs).

- Abstractions: Don't create helpers, utilities, or abstractions for one-time operations.
  Don't design for hypothetical future requirements. The right amount of complexity is the
  minimum needed for the current task.
```

The four-area framing (Scope / Documentation / Defensive coding / Abstractions) tends to land better than a generic "don't over-engineer" instruction because each category names a specific behavior the model can recognize and suppress.

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
