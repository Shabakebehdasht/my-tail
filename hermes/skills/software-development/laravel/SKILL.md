---
name: laravel
description: "Laravel pitfalls: PHPStan baseline, Pest tests, CI patterns."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [laravel, phpstan, pest, ci, livewire, baseline, testing]
    category: software-development
    related_skills: [livewire-debugging, improve, systematic-debugging]
---

# Laravel Development

Workflow and pitfalls for Laravel projects. Not a duplicate of AGENTS.md — this covers the non-obvious traps and procedures that cost time when missed.

## PHPStan Baseline Management

**Rule:** After any code change that fixes or alters PHPStan-ignored errors, regenerate the baseline BEFORE pushing.

The `phpstan-baseline.neon` file contains patterns that match known errors. When you fix code that had an ignored error, the pattern no longer matches — and PHPStan treats *unmatched baseline patterns* as errors themselves. This causes CI to fail even though your code is now *better*.

### Procedure

1. Make your code changes
2. Run `vendor/bin/pint --dirty --format agent` (format first)
3. Run `XDEBUG_MODE=off php vendor/bin/phpstan analyse --generate-baseline` to regenerate
4. Commit BOTH the code change AND `phpstan-baseline.neon` together

```bash
# One-liner: format + baseline regen
cd /path/to/project && \
  XDEBUG_MODE=off php vendor/bin/pint --dirty --format agent && \
  XDEBUG_MODE=off php vendor/bin/phpstan analyse --generate-baseline
```

### Symptom

CI fails with errors like:
```
Ignored error pattern #^...$# in path ... was not matched in reported errors.
```

This means the baseline has patterns for code you just fixed. Regenerate.

### Pitfall

- **Never commit a code fix that changes PHPStan-ignored behavior without also regenerating the baseline.** The baseline and code must stay in sync. If you only commit the code fix, CI will fail on the now-stale baseline patterns.
- **Run pint BEFORE phpstan baseline regen.** Pint may change code that affects PHPStan analysis. Baseline must reflect the final formatted state.

## Pest Testing Conventions

### Browser Testing (Pest v4 + Playwright)

Pest v4 has built-in browser testing powered by Playwright. Use it for critical user flows where `Livewire::test()` is insufficient.

**Setup procedure:**
```bash
composer require pestphp/pest-plugin-browser --dev
npm install playwright@latest --save-dev
npx playwright install chromium
```

**Test location:** `tests/Browser/` directory. Add a `Browser` testsuite to `phpunit.xml`:
```xml
<testsuite name="Browser">
    <directory>tests/Browser</directory>
</testsuite>
```

**Run:**
```bash
php artisan test --testsuite=Browser
# Or: composer test:browser (if script added to composer.json)
```

**Livewire browser testing:**
```php
// Use Livewire::visit() instead of Livewire::test()
livewire('users.index')
    ->visit('/users')
    ->type('[wire:model="search"]', 'query')
    ->assertSee('Result');
```

**Gitignore additions:**
```
/test-results/
/playwright-report/
/blob-report/
/playwright/.cache/
```

**CI (GitHub Actions):**
```yaml
- name: Install Playwright browsers
  run: npx playwright install chromium --with-deps

- name: Run browser tests
  run: ./vendor/bin/pest --testsuite=Browser
```

See `references/pest-browser-testing.md` for full setup details.

### Pest Test Execution Order

1. Always `php artisan config:clear && php artisan route:clear` before running tests
2. Use `XDEBUG_MODE=off` to avoid Xdebug interference
3. `composer test` is the canonical one-command way (includes clears + XDEBUG_MODE)
4. For single file: `XDEBUG_MODE=off php artisan test tests/Feature/SomeTest.php`

## Git Push When Local ≠ Remote Branch Name

When local branch name differs from remote tracking branch (e.g., local `rebecca` tracking `origin/beta`), `git push` fails with:
```
The upstream branch of your current branch does not match the name of your current branch.
```

**Fix:** Push explicitly to the remote branch:
```bash
git push origin HEAD:rebecca           # push to named remote branch
git push origin HEAD:refs/heads/rebecca # force-create remote branch
```

For PRs, use `gh pr create --head Shabakebehdasht:rebecca --base beta`.

## CI Re-trigger Pattern

When CI fails on a flaky test and you don't have admin rights to rerun:

```bash
git commit --allow-empty -m "ci: re-run tests" && git push origin HEAD:branch-name
```

This pushes an empty commit that triggers a fresh CI run. Use sparingly — if the test fails again, investigate rather than re-triggering.

## Livewire Conventions (Quick Reference)

These are workflow rules, not the full debugging guide (see `livewire-debugging` skill):

- **Single-file components only** — class is inline anonymous at top of Blade view, no `app/Livewire/*.php` files
- **Reference by dot-name** — `'hr.dashboard'`, `'auth.login'`, not class names
- **`wire:model.live.debounce.300ms`** on search inputs — never bare `wire:model.live` for search
- **Guard computed properties** — if it queries DB, require context + min input + limit (see `livewire-debugging`)
- **RTL always** — `dir="rtl"` at root, Persian/Arabic text throughout
- **MaryUI components** — use `x-input`, `x-select`, `x-button`, `x-modal` from MaryUI, not raw HTML

## Artisan Migration Naming

New migrations: `YYYY_MM_DD_000001_description.php` — sequential daily counter. Always pass `--no-interaction`.

## Formatting Gate

Run `vendor/bin/pint --dirty --format agent` before committing ANY PHP changes. This is non-negotiable — CI runs Pint and will reject unformatted code.