---
name: playwright-e2e-testing
description: Use when writing Playwright E2E tests. Probe real DOM first.
---

# Playwright E2E Test Authoring

Writing browser E2E tests that pass against a live app. The core discipline: the
rendered DOM is the source of truth, not the template/Blade source. JS frameworks
and component libs (Livewire, MaryUI, Alpine) transform markup into nodes you can
only know by looking at the running page.

## Workflow (in order)

1. **Baseline first.** Run the existing spec before touching it, e.g.
   `npx playwright test tests/e2e/<dir>/ --reporter=list`. You want to see what
   *actually* passes vs. what the plan claims, and catch environment issues (missing
   browser, wrong baseURL) before writing anything new.

2. **Probe the real DOM with a throwaway script.** Before writing or fixing a
   locator, run a short Node script (`import { chromium } from '@playwright/test'`)
   that drives the page and prints `innerText()`, `.count()`, and
   `evaluateAll()` of element attributes. Delete it after. This turns "guess the
   selector" into "read the selector off the rendered node". See
   `references/livewire-maryui-quirks.md` for the attribute shapes to expect.
   For multi-role authorization coverage see `references/rbac-multi-role-login.md`.

3. **Write locators from the probe output.** Prefer, in order: stable `#id`,
   `[wire\:model="..."]` attribute selectors, `getByRole('name', { name: '…' })`,
   visible Persian/RTL text. Do NOT assume `data-testid` exists — check first.

4. **Run green, then commit + push** per the repo's convention.

## Pitfalls (each cost real time)

- **Validation/error text renders in TWO places** (inline under the field AND in a
  summary/`x-errors` box). A bare `locator('text=...')` then throws strict-mode
  violation. Use `.first()` or `toContainText` on a scoped locator.
- **URL assertions against a `baseURL`:** `toHaveURL('**/login')` is unreliable —
  the glob mixes with the base into a wrong absolute URL. Use a predicate:
  `toHaveURL((url) => url.pathname === '/login')` (and `waitForURL` the same way).
- **`required` fields trigger native HTML5 validation, not server-side.** An empty
  submit stays on the page with `validity.valueMissing === true` and NO round-trip,
  so there is no Persian "required" message to assert. Assert the validity flag
  (`toHaveJSProperty('validity.valueMissing', true)`) instead of expecting a
  translated error string.
- **Stateful side effects must be reverted in the same test.** A password-change
  test that alters a shared seeded account's password must change it back inside
  the test, or the next run starts with a broken login.
- **`npm install` bumps patch versions in package-lock.json** (unrelated deps get
  `^`-range updates). That diff is real and harmless — commit it, don't fight it.
- **Non-destructive testing is the default.** E2E suites share seeded data across runs,
  so no test may alter it. For destructive features (bulk ops, settings toggles,
  archive/clear tools, record CRUD) assert the trigger *exists and is guarded*, never
  the mutation if it would persist. When a feature is genuinely broken (a route
  500s, a CRUD view is missing), do NOT fake a pass — mark the test `test.fixme`
  with a one-line comment naming the defect, so the blockage is visible, not hidden.
- **Forbidden = HTTP 403, not a redirect.** Spatie/Livewire route-level authorization
  answers unauthorized navigation with a 403 response; the user is not bounced to
  `/login` or a permission page. Make the status code the RBAC signal
  (`response.status()` after `page.goto`, or `page.request.get(url)`), not a URL
  change or an error string — those don't change for a 403.
- **Placeholder / UI hint text is not behavior.** A search box placeholder that says
  "min 2 characters" does not mean 2 characters are enforced (the debounce may fire
  on 1). Probe what actually happens — fill 1 char and observe — before asserting
  the hint as a gate. The hint can be locally displayed even when the constraint
  doesn't exist.
- **Framework / viz libraries render their own DOM, not your assumptions.** Highcharts
  emits `.highcharts-*` SVG, Leaflet markers are frequently custom `divIcon`s
  (e.g. `.unit-marker`) rather than the stock `.leaflet-marker-icon`. Probe the live
  container for the real classes (`locator('.leaflet-container')`, then `evaluateAll`
  for actual marker/layer classes) before writing locators.
- **Smoke-link loop: use `page.request.get`, not page navigation.** To verify "no
  broken links", collect hrefs once from the drawer (`evaluateAll` deduping + filtering
  `href.startsWith('/') && !href.startsWith('//')`), then issue each as
  `page.request.get(href)` — the APIRequestContext shares the browser's auth
  session/cookies. Only `status >= 500` is a real break; 302 → /login and 404 mean
  "route exists but redirects/missing", not a crash, so tolerate them.
