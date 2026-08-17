# Luna task packets and queues

Read this file before the first Luna dispatch, a repair round, or a continuous queue.

## Spawn invariant

Always use:

```text
model: "gpt-5.6-luna"
reasoning_effort: "xhigh"
fork_turns: "none"
```

Use a unique lowercase task name. Never let a worker spawn nested agents.

Before a wave, use `list_agents` and the current runtime's declared collaboration capacity. Count the primary agent as one occupied slot. Use `wait_agent` for bounded waiting after dispatch.

## Initial task packet

```text
Worker / task / wave:
<worker ID> / <task ID> / <wave number>

Workspace and base:
<absolute path> / <base SHA> / <branch or detached state>

Goal:
<one bounded outcome>

Facts and plan reference:
<verified current facts, relevant plan section, dependencies>

Allowed reads and writes:
<exact paths; distinguish read scope from write scope>

Owned resources:
<build output, ports, database/schema, cache, generated files, temp paths>

Invariants and acceptance criteria:
<behavior, state, API, UI, data, security, compatibility, rollback>

Forbidden actions:
- Do not modify paths outside the write set.
- Do not commit, push, deploy, publish, message externally, or change production state.
- Do not reset, checkout, switch, restore, clean, stash, rebase, merge, cherry-pick, delete user data, or remove worktrees.
- Do not overwrite unrelated or pre-existing user changes.
- Do not spawn nested agents or delegate this task.
- Stop without editing if the base or workspace differs from this packet.

Implementation:
<concrete steps and non-goals>

Verification Luna may run:
<safe focused commands and resource restrictions>

Required final report:
- Changed files and concise rationale
- Commands run and exact results
- Remaining risks or unknowns
- Out-of-scope observations without edits
- Processes, ports, temp paths, and generated artifacts left behind
```

Repeat critical upstream contracts verbatim. Do not rely on inherited chat history.

## Repair packet

Reuse the same Luna worker with `followup_task` after it becomes idle:

```text
Repair only these verified findings:
1. <file:line, observed problem, required correction>
2. <file:line, observed problem, required correction>

Preserve:
<accepted behavior and files>

Do not touch:
<exact paths and non-goals>

Rerun:
<focused verification>

Report the new diff, results, risks, and remaining resources.
```

Do not ask Luna to “review itself.” Give concrete findings.

## Queue table

```markdown
| Order | Wave | Task | Depends On | Goal | Write Set | Worktree / Base | Resources | Verification | Status |
| ---: | ---: | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | T01 | none | <bounded outcome> | <paths> | <path / SHA> | <owned resources> | <command> | Not Started |
```

Use this transition:

```text
Not Started -> In Progress -> Review -> Done
                              -> Fixing -> Review
                              -> Blocked
```

Select only dependency-satisfied items. Mark an item `In Progress` before spawning and `Done` only after primary-agent review and independent verification.

Stop the queue when it is empty, blocked, interrupted, outside scope, missing acceptance evidence, or still failing after 2-3 focused repair rounds.
