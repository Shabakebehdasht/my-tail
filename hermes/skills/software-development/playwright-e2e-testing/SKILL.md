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
- **Fill independent spec gaps in parallel.** Spec files/dirs are independent work —
  dispatch one worker per file or directory with its probe facts in context, then
  verify each green before the final full run. Sequential gap-filling is the slow path.
- **Full-suite runs go to the background.** A whole-suite Playwright run takes minutes
  and outlives foreground command timeouts — launch it as a background process that
  notifies on completion and verify from its log. Keep per-file runs foreground.
- **Blocked runner wrapper → call the local binary.** If `npx playwright` fails,
  `./node_modules/.bin/playwright` is the same CLI without the resolution step —
  identical args, pinned version.
- **Narrow nullable extractions before asserting.** Specs are type-checked:
  `innerText().match(/.../)?.[0]` is `string | undefined` and fails
  `toContainText`/`toMatch` typing. Assert `not.toBeNull()` first, then bind
  `const x: string = m![0]`.
- **Probe scripts must live inside the project directory.** Node ESM resolves
  `@playwright/test` by walking up from the script file, so a probe saved to
  `/tmp` fails with `ERR_MODULE_NOT_FOUND` — copy it into the repo
  (e.g. `probe-*.mjs`), run it with `node`, and delete it after.
- **A 500 on a nested route with a working index page usually means a missing view,
  not a broken feature.** Grep the views directory for the component name and probe
  each URL's status separately before declaring the feature dead — the list page's
  inline create/edit form may fully work while the standalone route 500s. When the
  view never existed and nothing references the route (no links, templates, or
  tests), delete the dead route and its orphaned view instead of scaffolding a
  duplicate page — then verify the dead URL returns a clean 404 and the index
  page still returns 200.
- **Plan-claimed features are claims, not facts.** Before writing a test for a
  query param, filter, or toggle from a plan doc, probe whether it exists and does
  anything (param changes the page? filter present on this page vs only inside a
  create modal?). Scenarios that are unimplementable without code changes or data
  mutation get `test.fixme` with the defect named, never a faked pass.
