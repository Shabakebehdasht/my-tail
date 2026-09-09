# Closing the Loop — execute, reconcile, issues

The advisor never edits source code. In `execute`, a separate executor subagent edits code in an isolated git worktree.

---

## `execute <plan>` — dispatch and review

### Preconditions
- Git repo (worktree isolation requires it).
- Plan file exists and dependencies show DONE in `plans/README.md`.
- Run the plan's drift check yourself first.

### Dispatch

Spawn one subagent with worktree isolation. The subagent prompt must contain:
1. The full plan file text, inlined.
2. The executor preamble.
3. Report format: STATUS, STEPS, STOPPED BECAUSE, FILES CHANGED, NOTES.

### Review

1. Re-run every done criterion in the worktree.
2. Scope compliance: diff against in-scope list.
3. Read the full diff.
4. Audit new tests.

### Verdict

| Verdict | When | Action |
|---|---|---|
| APPROVE | Criteria pass, scope clean | Update index to DONE. Never merge/push. |
| REVISE | Fixable gaps | Send specific feedback. Max 2 rounds, then BLOCK. |
| BLOCK | STOP condition or scope violation | Mark BLOCKED, refine plan. |

---

## `reconcile` — keep plans/ alive

Process what happened since last session:
- DONE: spot-check criteria still hold.
- BLOCKED: investigate, rewrite or mark REJECTED.
- IN PROGRESS (stale): flag to user.
- TODO: run drift check, refresh if drifted.

---

## `--issues` — publish as GitHub issues

1. Preflight: `gh auth status` succeeds.
2. Visibility check: warn if public repo.
3. Per plan: `gh issue create`.
4. Record issue URL in plan and index.