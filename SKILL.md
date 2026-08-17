---
name: luna-agent-dispatcher
description: Delegate bounded implementation, cleanup, review, or continuous queue work from Codex to built-in GPT-5.6 Luna subagents pinned to xhigh reasoning, while Codex owns scoping, worktree isolation, diff review, verification, integration, and cleanup. Use when the user asks for Luna subagents, lower-cost built-in workers, controlled parallel workers, or worktree-isolated delegated execution.
---

# Luna Agent Dispatcher

Use GPT-5.6 Luna as the execution worker while the primary Codex agent remains the sole delivery owner.

## Non-negotiable contract

- Spawn every worker with `model: "gpt-5.6-luna"` and `reasoning_effort: "xhigh"`.
- Default to `fork_turns: "none"` and send a self-contained task packet. Use a small positive turn count only when recent dialogue is essential. Never use `fork_turns: "all"` when overriding model or effort.
- Do not silently substitute another execution backend, model, or reasoning level. If Luna is unavailable, report the blocker or let the primary agent take over when that remains within the user's request.
- Keep one worker by default. Use a parallel wave only for proven-independent tasks with disjoint ownership and sufficient collaboration slots.
- Assign at most one worker to a queue item. Tell every worker not to spawn nested agents or delegate its task.
- Let Luna implement or inspect only the assigned slice. Keep architecture, prioritization, integration, review, verification, and final acceptance with the primary agent.
- Do not trust a worker summary. Inspect repository facts and run verification independently.
- Preserve user changes. Do not commit, push, deploy, publish, message externally, or run destructive Git/filesystem operations without explicit authorization.

## Choose the execution shape

1. Execute a clear, cohesive slice with one Luna worker.
2. Create a bounded queue when the user requests continuous work or the task contains dependent slices.
3. Use parallel workers only when all ready items pass the parallel gate below.
4. Keep trivial or strongly context-coupled work in the primary agent unless the user explicitly requests Luna.

For broad work, reuse an existing task plan or LoopX todo. If no executable plan exists, create one before dispatching. Planning must identify scope, invariants, acceptance criteria, verification, rollback, and task dependencies.

## Preflight

Before an editing worker starts, establish the repository and resource baseline:

```bash
pwd
git rev-parse --show-toplevel
git rev-parse HEAD
git branch --show-current
git status --short --untracked-files=all
git worktree list --porcelain
```

Also:

- Use `list_agents` to inspect live agents and do not duplicate a running queue item. Count the primary agent as one occupied collaboration slot; never infer capacity from the number of children alone.
- Record the worker ID, task ID, base SHA, target path, allowed write set, ports, databases, build outputs, temp paths, and cleanup owner.
- Require a clean target worktree or a verified recovery record covering staged, unstaged, untracked, and required ignored inputs.
- If an edit depends on uncommitted source changes, do not assume a new worktree contains them. Use a read-only Luna worker and apply controlled edits in the primary worktree, or request authority for a validated local snapshot.
- Read [worktrees.md](references/worktrees.md) before creating a worktree, running parallel edit workers, integrating results, or cleaning resources.
- Read [task-packets.md](references/task-packets.md) before the first dispatch or when creating a queue.

## Build and spawn the worker

Create a self-contained packet with exact facts, allowed paths, forbidden actions, invariants, acceptance criteria, and verification commands. Use absolute paths. State that the worker must operate only inside the assigned workspace and must stop if the recorded baseline differs.

Spawn with this fixed configuration:

```text
spawn_agent({
  task_name: "<unique-lowercase-name>",
  fork_turns: "none",
  model: "gpt-5.6-luna",
  reasoning_effort: "xhigh",
  message: "<self-contained task packet>"
})
```

Do not omit either model field. Do not rely on inherited conversation context to carry constraints.

## Wait and communicate

- Use `wait_agent` instead of repeatedly polling. Prefer approximately 1-2 minutes for a small task, 5 minutes for a medium task, and 10-15 minutes for a large bounded task.
- Keep user updates sparse unless status is requested.
- Use `send_message` only to add information while a worker is running.
- Use `followup_task` for a focused repair after the worker becomes idle.
- Interrupt only when the user changes direction, scope safety fails, or the worker is causing active harm.
- Never start a second worker for the same queue item merely because the first is quiet.

## Independent review gate

After Luna finishes:

1. Recheck the recorded HEAD and the complete worktree state.
2. Inspect tracked, staged, untracked, and task-relevant ignored changes:

```bash
git status --short --untracked-files=all
git diff --stat
git diff
git diff --cached
git ls-files --others --exclude-standard
```

3. Compare every changed path with the allowed write set. Reject or repair out-of-scope changes.
4. Read the actual implementation and validate business/state invariants; do not accept line count or a green command alone.
5. Run the smallest relevant tests, lint, build, typecheck, or runtime check independently.
6. Check task-owned generated and ignored paths explicitly when the task can create them.
7. Patch a small mechanical issue directly, or send the same Luna worker a focused repair packet with exact findings and verification.
8. Stop after 2-3 failed repair rounds and either take over or report a blocker.

Mark an item `Done` only after this gate passes.

## Queue and parallel waves

Use these queue states: `Not Started`, `In Progress`, `Review`, `Fixing`, `Done`, `Blocked`, and `Skipped`.

Run serially by default. A parallel wave requires all of the following:

- No item depends on another item's unintegrated output.
- Write sets, public interfaces, migrations, generated files, lockfiles, fixtures, ports, databases, caches, and build outputs do not overlap.
- Every editing worker has a separate worktree from the same recorded base SHA. Never run parallel editing workers in one worktree.
- The integration order and cross-wave verification command are fixed before spawning.
- The integration mechanism accounts for tracked, untracked, binary, and required ignored files. If it is not explicit and recoverable, run serially.
- The active child count stays within available collaboration slots. Default wave size is 2; never assume more capacity than the runtime reports.

After a wave, review each worktree separately, integrate accepted results one at a time, and run cross-wave verification before dispatching dependent items.

## Cleanup and delivery

- Confirm each Luna worker is idle or finished and no task-owned process, port, temp path, or worktree is orphaned.
- Remove a worktree only after accepted changes are integrated or rejected changes are safely accounted for and the worktree is clean.
- Preserve an unsafe-to-remove worktree and report its exact path and recovery state.
- Tell the user what Luna did, what the primary agent independently verified, what was repaired, what remains, and which resources were removed or preserved.

Never present Luna's summary as verified fact.
