# Closing the Loop — execute, reconcile, issues

## `execute <plan>` — dispatch and review

### Preconditions
- Git repository (worktree isolation requires it).
- Plan file exists and dependencies show DONE in `plans/README.md`.
- Run drift check first.

### Dispatch
Spawn one `general-purpose` subagent with `isolation: "worktree"`. Inline the full plan file text in the subagent prompt.

### Review
1. Re-run every done criterion in the worktree.
2. Scope compliance: `git diff --stat` against plan's in-scope list.
3. Read the full diff.
4. Audit the new tests.

### Verdict
| Verdict | When | Action |
|---|---|---|
| APPROVE | Criteria pass, scope clean | Update index to DONE. Present diff summary. |
| REVISE | Fixable gaps | SendMessage with specific feedback. Max 2 rounds. |
| BLOCK | STOP condition, scope violation, revisions exhausted | Mark BLOCKED. Refine plan. |

## `reconcile` — keep plans/ alive
- DONE: spot-check done criteria on current HEAD.
- BLOCKED: investigate, rewrite or reject.
- IN PROGRESS (stale): flag to user.
- TODO: run drift check.

## `--issues` — publish plans as GitHub issues
Modifier on any planning invocation. Creates `gh issue create` per plan.