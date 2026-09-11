---
name: laravel-ci-checks
description: Fix failing Laravel Pint/PHPStan CI checks.
---

# Laravel CI Checks (Pint + PHPStan)

Keeping a Laravel PR green when the style/static-analysis jobs fail. The
core discipline: reproduce the exact CI command locally before touching code —
CI runs the whole-repo check, not just your diff.

## Workflow (in order)

1. **Identify the failing job** with `gh pr checks` (or the repo's equivalent).
   Note whether the flagged files are yours or pre-existing — both fail your PR.

2. **Reproduce locally with the CI command**, not a subset:
   - Style: `php vendor/bin/pint --test` (whole repo; `--test` on one file can
     pass while the repo fails).
   - Analysis: `php vendor/bin/phpstan analyse --no-progress`. Raw format
     (`--error-format=raw`) gives grep-able `path:line` lines.

3. **Pint failures:** run `php vendor/bin/pint <flagged-files>`, then inspect
   the diff — it must be cosmetic only (import ordering, FQCN cleanup,
   whitespace). Re-run `pint --test` to confirm clean.

4. **PHPStan `ignore.count` failures** (`expected to occur N times, but
   occurred M`): this is baseline count drift — code changed without updating
   `phpstan-baseline.neon`. Grep the baseline for the message, set `count:` to
   the actual occurrences. Typical causes: deleted routes lower a
   `Route::livewire` count; added `auth()->user()` calls raise a per-file count.

5. **PHPStan new uncovered error** (no baseline entry, e.g. from another
   commit's feature): add a baseline entry copying an adjacent entry's exact
   shape, or regenerate the baseline — never add `@phpstan-ignore` comments or
   widen types to silence it.

6. **Commit, push, re-watch** the checks to green.

## Pitfalls (each cost real time)

- **Whole-repo checks fail your PR for other people's files.** Fix them anyway
  after verifying the diff is cosmetic — arguing ownership doesn't turn CI green.
- **Deleting routes/views changes baseline counts.** After removing Livewire
  routes, recount the `Route::livewire` baseline entry instead of chasing ghosts.
- **Hand-edited `.neon` regexes need byte-exact backslash escaping.** One extra
  backslash silently fails to match; verify the edited line with `od -c`
  against a neighboring entry, or regenerate the baseline instead of hand-editing.
