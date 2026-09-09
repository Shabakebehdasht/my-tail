---
name: improve
description: "Use when auditing codebases and writing improvement plans."
version: 1.0.0
author: shadcn (https://github.com/shadcn/improve)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [audit, codebase, improvement, planning, code-quality]
    category: software-development
---

# Improve

An agent skill that audits any codebase and writes implementation plans for other agents to execute. The idea: use your most capable model for the part where intelligence compounds — understanding the codebase, judging what's worth doing, writing the spec — and hand execution to cheaper models. The skill never implements anything itself. The plan is the product.

```
you          →  /improve                    (expensive model, advises)
plans/       →  001-fix-n-plus-one.md       (self-contained specs)
other agent  →  implements, tests, ships    (cheap model, executes)
```

## Hard Rules
1. **Never modify source code yourself.** The ONLY files you may create or modify live under `plans/` in the repo root — or under `advisor-plans/` when `plans/` already exists for an unrelated purpose.
The `execute` variant dispatches a *separate executor subagent* that edits code in an isolated git worktree — you review its diff and render a verdict.
2. **Never run commands that mutate the user's working tree** — no installs, no builds, no git commits, no formatters. Read, search, and run read-only analysis only.
3. **Every plan must be fully self-contained.** The executor has not seen this conversation.
4. **If the user asks you to implement directly, decline and point at the plan.**

## Workflow

### Phase 1 — Recon (always)
- Read `README`, `CLAUDE.md`/`AGENTS.md`, root config files, CI config, directory structure.
- Identify: language(s), framework(s), package manager, build/test/lint commands, test coverage, deployment target.
- Note repo conventions: code style, naming, folder layout, error-handling patterns.
- Ingest intent & design docs where present.

### Phase 2 — Audit (parallel)
- Scope findings with evidence (`file:line`), impact, effort (S/M/L), risk, confidence.

### Phase 3 — Present findings table

### Phase 4 — Write plans for selected findings
- Use the plan template in `references/plan-template.md`.
- Plans go in `plans/` with a `plans/README.md` index.

## Invocation variants
- `/improve` — full audit
- `/improve quick` — cheap pass: hotspots, top findings only
- `/improve deep` — exhaustive
- `/improve security` — focused security audit (also: perf, tests, bugs, etc.)
- `/improve branch` — audit only what the current branch changes
- `/improve next` — feature suggestions
- `/improve plan <description>` — skip audit, spec one thing
- `/improve review-plan <file>` — critique an existing plan
- `/improve execute <plan>` — dispatch executor, review work
- `/improve reconcile` — refresh the backlog
- `/improve ... --issues` — publish plans as GitHub issues

## Key references
- `references/audit-playbook.md` — what to look for per category, finding format
- `references/plan-template.md` — handoff plan template for executors
- `references/closing-the-loop.md` — execute, reconcile, issues workflows