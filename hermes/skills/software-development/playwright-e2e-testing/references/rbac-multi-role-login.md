# RBAC / multi-role E2E login pattern

Testing authorization across seeded roles requires logging in as each role, not just
the admin. This covers the reusable shape.

## Login as a specific role

The seed has one shared password for every user (in this app `12345678`). Log in by
national-code + that password, then assert the *per-role* menu/route after login.
Query the role→user mapping first (here via `boost_tool.php database-query` on
`model_has_roles` JOIN `roles` JOIN `users`) so you pick a concrete user per role
instead of guessing IDs.

```ts
// per-role login helper — reuse the shared password, vary the n_code
async function loginAs(page, nCode) {
  await page.goto('/login');
  await page.fill('#n_code', nCode);
  await page.fill('#password', '12345678');
  await page.click('button[type="submit"]');
  await page.waitForURL((u) => !u.pathname.includes('/login'));
}
```

## The RBAC signal is the HTTP status, not a redirect

Spatie/Livewire route-level `can:`/middleware authorization returns **HTTP 403** to a
forbidden navigation — it does not 302 to `/login` nor render an inline error. To test
"role X cannot access /route", drive the navigation and read the status:

```ts
const resp = await page.goto('/users');
expect(resp.status()).toBe(403);              // forbidden role
// admin → 200
```

`page.goto()` returns the navigation `Response`, so `.status()` is the assertion. The
visible 403 page shows only "403 / Forbidden" — there is no Persian message or URL
delta to latch onto.

## Assert the menu, not just the route

Two orthogonal checks per role:
1. **Sidebar sections** — count / assert which top-level `<details><summary>` sections
   render (admin sees more sections than a limited role). This is `activate-by-route`
   `x-menu`; see the main livewire-maryui-quirks reference for the DOM shape.
2. **Forbidden navigation** — loop the protected routes and assert 403 per role,
   plus 200 for the admin, in one parametrized `test`.
