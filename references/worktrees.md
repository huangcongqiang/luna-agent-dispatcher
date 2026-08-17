# Worktree and isolation rules

Read this file before creating a worktree, dispatching parallel editors, integrating worker output, or cleaning temporary resources.

## Isolation choice

| Situation | Required choice |
| --- | --- |
| Read-only work | Workers may share a repository only when no mutating command or build is allowed. |
| One edit worker on a clean target | The worker may use the target worktree after its baseline is recorded. |
| Parallel edit workers | Use one separate worktree per worker from the same committed base SHA. |
| Edit depends on dirty or untracked source | Do not create an ordinary worktree and assume it contains those changes. Use read-only analysis plus primary-agent edits, or obtain authority for a validated local snapshot. |
| No enforced or recoverable boundary | Keep editing serial. |

Built-in agents share the host filesystem. A worktree isolates the Git index and normal in-tree outputs; it does not create an OS security boundary. Never permit same-worktree parallel editing.

## Resource ledger

Record one row per worker before spawning:

| Field | Meaning |
| --- | --- |
| Worker / task | Unique agent name and queue item |
| Agent ID | ID returned by `spawn_agent` |
| Base | Source repository, base SHA, branch or detached state |
| Workspace | Exact absolute worktree path |
| Write set | Exact permitted files or directories |
| Mutable resources | Build outputs, ports, database/schema, cache, generated files, temp path |
| Integration | Review order and approved transfer mechanism |
| Cleanup | Owner, current state, and removal condition |

Keep the ledger in the existing plan/progress file when one exists. Otherwise keep it in the primary agent's task state; do not create a new project document solely for a single worker.

## Creation gate

Before `git worktree add`:

1. Inspect source HEAD, staged, unstaged, untracked, and relevant ignored state.
2. Confirm the base SHA contains every input the worker requires.
3. Select an explicit task-owned path; never use a broad directory or unresolved variable.
4. Record whether the worktree is detached or on a task branch.
5. Verify that build output, ports, databases, caches, and temp paths are not shared with another active editor.
6. Put the exact workspace path in the Luna task packet and require all commands to run there.

Do not silently stash, commit, copy, or discard user changes to manufacture a baseline.

## Integration gate

Review worker output in its own worktree before transfer. Inspect:

- tracked and staged diffs;
- untracked files and their contents;
- task-relevant ignored files;
- mode changes, symlinks, binaries, generated outputs, and deletions;
- processes or services still using the worktree.

Choose the integration mechanism before a parallel edit wave:

1. Use a user-authorized local commit created by the primary agent after review, then integrate it in the recorded order; or
2. Use an exact, tested content-transfer procedure that includes tracked, untracked, binary, and required ignored files and verifies resulting hashes.

If neither mechanism is authorized and reliable, do not run parallel edit workers. Run serially in the integration worktree instead.

After each accepted integration, check semantic/API conflicts with remaining results. Run the recorded cross-wave verification only after all accepted results are present.

## Cleanup gate

Before removing a worktree:

1. Confirm the worker is finished and no task-owned process uses the path.
2. Inspect staged, unstaged, untracked, ignored, conflict, and branch state.
3. Confirm accepted content is integrated or rejection is explicitly recorded.
4. Remove only the exact recorded worktree path without force.
5. Run `git worktree prune` only after explicit removals.

If any check is uncertain, preserve the worktree and report its path, state, owner, and next recovery action.
