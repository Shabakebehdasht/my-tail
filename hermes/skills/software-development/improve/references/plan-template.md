# Handoff Plan Template

Every plan is written for an executor model with **zero context**. Self-contained, with verification gates and hard boundaries.

## Template

```markdown
# Plan NNN: <Imperative title>

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in "STOP conditions" occurs, stop and report.

## Status
- **Priority**: P1 | P2 | P3
- **Effort**: S | M | L
- **Risk**: LOW | MED | HIGH
- **Depends on**: plans/NNN-*.md (or "none")
- **Category**: bug | security | perf | tests | tech-debt | migration | dx | docs | direction
- **Planned at**: commit `<short SHA>`, <YYYY-MM-DD>

## Why this matters
2–5 sentences on the problem, its cost, and what improves.

## Current state
- Relevant files with their roles.
- Code excerpts with file:line markers.
- Repo conventions to match.

## Commands you will need
| Purpose | Command | Expected |
|---------|---------|----------|
| Install | ... | exit 0 |
| Test | ... | all pass |

## Scope
**In scope**: list of files.
**Out of scope**: list of files not to touch.

## Git workflow
- Branch naming, commit style.

## Steps
### Step 1: <title>
What to do. **Verify**: `<command>` → expected result.

## Test plan
- New tests, which file, which cases.

## Done criteria
Machine-checkable. ALL must hold.

## STOP conditions
Stop and report if:
- Code doesn't match excerpts.
- Verification fails twice.
- Fix requires out-of-scope file.
```