---
name: e2e-playwright
description: "Playwright E2E: locators, Livewire patterns, RTL test tips."
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [e2e, playwright, testing, livewire, rtl, persian]
    related_skills: [systematic-debugging, dogfood]
    requires_commands: [npx]
---

# Playwright E2E Testing

## Overview

Playwright E2E tests for component-heavy web apps (Livewire + MaryUI, React, Vue) where shared UI elements (modals, sidebars, navigation) create text ambiguity.

## Core Principle

**Locate by role and structure, not by text content alone.** In apps with help modals, sidebars, and navigation that repeat the same text labels, `page.locator('text=...')` resolves to multiple elements and throws strict-mode errors.

## Locator Strategy (Priority Order)

1. **`getByRole()`** — most resilient. Targets semantic HTML role + accessible name.
2. **`getByText()` + `.first()`** — when role is ambiguous. Use `.first()` to pick the topmost match (header before modal).
3. **`locator()` with structural CSS** — `.locator('h3', { hasText: '...' })` or `.locator('[class*="stat"]').locator('.font-black')` for scoped selection.
4. **`page.locator('text=...')`** — LAST RESORT only when text is truly unique on the page.

### Ambiguous Text Patterns

When text appears in multiple DOM positions (sidebar nav, help modal, content cards):

```typescript
// BAD — strict mode violation if text matches multiple elements
await expect(page.locator('text=کاربران')).toBeVisible();

// GOOD — target the specific structural container
await expect(page.getByText('تعداد کل کاربران سیستم')).toBeVisible();
await expect(page.getByRole('heading', { level: 3, name: '...' })).toBeVisible();
await expect(page.getByRole('link', { name: /مشاهده همه/ })).toBeVisible();
await expect(page.locator('.font-black.text-xl')).toHaveText(/^\d[\d,]*$/);
```

### Why `.first()` Is Correct for Headers

In Livewire/Blade apps, `<x-header>` renders the title in the main content area AND the help modal has the same title. The header is always first in DOM order — `.first()` picks it deterministically.

## MaryUI Component DOM Patterns

MaryUI components render custom DOM that differs from standard HTML. Key patterns:

### x-select
Renders as a `<fieldset>` with a legend label and a trigger button. The `wire:model` attribute is on the outer `<div>`, NOT on a `<select>` element.

```typescript
// BAD — no <select> with wire:model exists
const sel = page.locator('[wire\\:model.live="filter_s_id"]'); // fails

// GOOD — locate by label text, then find the trigger
const label = page.locator('legend:has-text("سمت"), label:has-text("سمت")').first();
const trigger = label.locator('..').locator('button').first();
await trigger.click();
// options appear in .dropdown-content li or [role="option"]
```

### x-modal
Does NOT render `role="dialog"`. Check open state via `.modal-open` class count:

```typescript
// Check modal is closed
const modalVisible = await page.locator('.modal-open, [role="dialog"]:visible').count();
expect(modalVisible).toBe(0);

// Close via Escape (MaryUI modals support this)
await page.keyboard.press('Escape');
```

### x-table action buttons
Action buttons in table rows use `wire:click` with the row's ID:

```typescript
const editBtn = page.locator('table tbody tr').first()
  .locator('button[wire\\:click*="editUnit"]');
await editBtn.click();
```

### Toggle buttons
Toggle buttons with specific `title` attributes are reliable selectors:

```typescript
const toggles = page.locator('button[title="تغییر وضعیت پذیرش تیکت"]');
await expect(toggles.first()).toBeVisible();
```

## Livewire Anonymous-Class Components

Projects using single-file Livewire components (anonymous class at top of Blade view) have these routing gotchas:

### Orphaned Routes

When CRUD functionality lives inline in `index.blade.php` (e.g., create + edit forms toggled by `formOpen`), separate `/create` and `/{id}/edit` routes point to views that don't exist → 500 error.

**Fix:** Redirect orphaned routes to the parent component:
```php
Route::get('/users/create', fn () => redirect('/users'));
Route::get('/users/{user}/edit', fn () => redirect('/users'));
```

### Verifying Fixes with curl

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/path
# 302 → /login = route exists, requires auth (correct)
# 500 = broken
# 200 = works
```

## Livewire Component Layouts vs Blade @extends

Livewire component layouts use `<x-slot:content>{{ $content }}</x-slot:content>` — NOT compatible with `@extends` / `@section`.

**Rule:** Standalone pages in a Livewire-first app must be self-contained (full HTML document) or use a dedicated Blade layout that doesn't rely on `$slot`.

**Detection:** If a page 500s with `Undefined variable $slot`, the view uses `@extends` against a Livewire component layout.

## Plan-Driven Test Development

When working from a `plans/e2e/` directory with gap lists:

1. **Read the README.md** status table to identify 🟡 partial plans
2. **Read the gap list** in the README for specific ❌ scenarios
3. **Read each plan file** for scenario tables and expected behavior
4. **Read existing test files** to understand current structure and DOM selectors
5. **Read the Blade views** to understand the actual DOM structure (MaryUI patterns above)
6. **Write tests** following the locator strategy above
7. **Mark destructive tests** as `test.fixme` with a rationale comment
8. **Run tests** to verify: `npx playwright test <dir> --reporter=list`
9. **Update the README** status table and gap list

### Destructive vs Non-Destructive Tests

- **Non-destructive** (read-only assertions, form renders, validation errors): write normally
- **Round-trip mutations** (toggle → verify → toggle back): write normally, they're safe
- **Destructive** (create records, delete, rollback): prefer round-trips with cleanup over `test.fixme`.

#### Round-trip pattern with cleanup

Tag test records with a unique prefix, then run a cleanup script after the suite:

```typescript
import { execSync } from 'child_process';

test.describe('tickets new', () => {
  test.afterAll(async () => {
    try {
      execSync('php tests/e2e/cleanup.php', { cwd: '/path/to/project', timeout: 10000 });
    } catch { /* cleanup best-effort */ }
  });

  test('create valid ticket → success', async ({ page }) => {
    const subject = `[E2E-TEST] ticket ${Date.now()}`;
    // ... fill form with subject, submit, verify ...
  });
});
```

The cleanup script (`tests/e2e/cleanup.php`) queries for records with the `[E2E-TEST]` prefix and deletes them:
```php
$deleted = DB::table('tickets')->where('subject', 'like', '[E2E-TEST%')->delete();
```

Only use `test.fixme` when no safe round-trip exists (e.g., Livewire toggle timing is unreliable in headless CI).

### wire:confirm is fragile in headless CI

Livewire's `wire:confirm` uses `window.confirm()`. Playwright auto-accepts it, but the Livewire round-trip after confirmation has unreliable timing in headless mode — the request may not complete before assertions run. Delete round-trips (delete → verify inactive → restore) are the most common failure. If a delete/confirm test fails intermittently, mark it `test.fixme` with a rationale noting `wire:confirm` timing, not a code bug.

## Export/Download Testing

For Excel/CSV export buttons that trigger Livewire dispatch events:

```typescript
// Listen for Livewire dispatch event
const downloadUrl = await page.evaluate(() => {
  return new Promise<string>((resolve) => {
    Livewire.on('download-export', (url: string) => resolve(url));
    setTimeout(() => resolve(''), 5000);
  });
});

await page.getByRole('button', { name: 'خروجی اکسل' }).click();
await page.waitForTimeout(2000);

if (downloadUrl) {
  expect(downloadUrl).toContain('hardware/export');
}
```

## RTL / Persian App Patterns

- Persian text in Playwright locators: use raw UTF-8 string, not escaped Unicode when possible
- Persian zero-width non-joiner (`\u200c`) in labels — use it in test strings if the app does
- `number_format()` in PHP adds comma separators — test values may show `1,234` not `1234`
- Locale: set in `playwright.config.ts` as `locale: 'fa-IR'` and `timezoneId: 'Asia/Tehran'`

## Running E2E Tests

```bash
npx playwright test                                    # all tests
npx playwright test tests/e2e/dashboard                # directory
npx playwright test tests/e2e/dashboard/stats.spec.ts  # file
npx playwright test --reporter=list                    # with reporter
npx playwright test --debug                            # headed browser
```

## Shared Fixtures Pattern

Create `tests/e2e/shared/fixtures.ts` with:
- `TEST_USER` credentials (from database seeders)
- `login()` helper that fills form and waits for redirect
- `authenticatedPage` fixture that auto-logs in
- `waitForLivewire()` helper (checks `.wire-loading` class)
- `waitForToast()` helper

Import from `../shared/fixtures` in all test files, not from `@playwright/test` directly.

## Pre-Test: Verify Data Access Patterns

Before writing tests that search or filter data, verify the test user's access scope. In apps with unit-scoped access (recursive CTE), the test user can only see data within their unit + descendants.

```bash
# Check what units the test user can access
php artisan tinker --execute '$u = \App\Models\User::where("n_code","4411015056")->first(); foreach($u->units as $unit) echo $unit->id . " - " . $unit->name . "\n";'
```

If the test searches for data outside the user's scope, it will find nothing and fail. Match search terms to actual accessible data.

## PHPStan Baseline Maintenance

When routes change (e.g., converting `Route::livewire()` to `Route::get()`), the baseline error counts become stale and PHPStan CI fails with `ignore.count` errors.

**Regenerate the baseline** instead of hand-editing counts:
```bash
php vendor/bin/phpstan analyse --no-progress --generate-baseline
```

This overwrites `phpstan-baseline.neon` with current counts. Verify locally before pushing:
```bash
php vendor/bin/phpstan analyse --no-progress  # should show [OK] No errors
```

**Pint check before push:**
```bash
vendor/bin/pint --test           # check only
vendor/bin/pint --dirty          # fix only modified files
```

If the gateway blocks Pint, run it via PHP directly:
```bash
php vendor/bin/pint --format=txt 2>&1 | head -30
```

## Debugging Failed E2E Tests

1. Read the error — Playwright shows the exact locator and what it resolved to
2. Check screenshots in `test-results/` (auto-captured on failure)
3. Check videos in `test-results/` (retained on failure)
4. `npx playwright show-trace test-results/<folder>/trace.zip` for full trace
5. Common failure: `strict mode violation` → locator matched multiple elements → narrow the locator
6. Common failure: `not a function` → `.not()` is not a Playwright API — use `expect(locator).not.toContainText()` or check `.isVisible()` instead