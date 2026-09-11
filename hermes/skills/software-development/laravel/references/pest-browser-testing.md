# Pest v4 Browser Testing Setup

## Installation

```bash
# 1. Install the Pest browser plugin
composer require pestphp/pest-plugin-browser --dev

# 2. Install Playwright
npm install playwright@latest --save-dev

# 3. Install browser binaries (chromium only for speed)
npx playwright install chromium

# For CI, include OS dependencies:
npx playwright install chromium --with-deps
```

## Configuration

### tests/Pest.php — Bind TestCase for Browser suite

**Required.** Without this, the Pest HTTP server starts an empty Laravel container and every page fails with `Target class [config] does not exist`.

```php
use Tests\TestCase;

uses(Tests\TestCase::class)->in('Browser');
```

### phpunit.xml — Add Browser testsuite

```xml
<testsuites>
    <testsuite name="Feature">
        <directory>tests/Feature</directory>
    </testsuite>
    <testsuite name="Browser">
        <directory>tests/Browser</directory>
    </testsuite>
</testsuites>
```

### composer.json — Add script

```json
"test:browser": [
    "Composer\\Config::disableProcessTimeout",
    "php artisan config:clear",
    "php artisan route:clear",
    "XDEBUG_MODE=off php artisan test --testsuite=Browser"
]
```

### .gitignore — Add Playwright artifacts

```
/test-results/
/playwright-report/
/blob-report/
/playwright/.cache/
```

## Test Database

Browser tests run against a real HTTP server connected to the real DB (not in-memory). Ensure the test database exists:

```bash
psql -h 127.0.0.1 -U <user> -d <db> -c \
  "CREATE DATABASE h_dashboard_test WITH OWNER=<user> TEMPLATE=template_postgis;"
php artisan migrate --env=testing --force
```

## Writing Browser Tests

### The `bootBrowser()` helper (REQUIRED for login-in-beforeEach)

pest-plugin-browser boots the Playwright websocket + Laravel HTTP server inside `__markAsBrowserTest`, a method proxy that runs at the START of the test body — AFTER `beforeEach`. Any `visit()`/`fill()` in `beforeEach` runs before that and dies with `Call to a member function sendText() on null` or `ServerNotFoundException`. Boot explicitly and idempotently:

```php
// tests/Browser/helpers.php
use Pest\Browser\Playwright\Client;
use Pest\Browser\ServerManager;

function bootBrowser(): void
{
    Client::instance()->connectTo(ServerManager::instance()->playwright()->url());
    ServerManager::instance()->http()->bootstrap(); // both no-op when already started
}

// login helpers return AwaitableWebpage — NOT Webpage
function loginViaBrowser(string $nCode, string $password = 'password'): \Pest\Browser\Api\AwaitableWebpage
{
    bootBrowser();
    return visit('/login')
        ->fill('#n_code', $nCode)        // id selector, NOT input[name="n_code"]
        ->fill('#password', $password)
        ->click('button[type="submit"]')
        ->waitForText('\u062f\u0627\u0634\u0628\u0648\u0631\u062f\u200c\u0645\u062f\u06cc\u0631\u06cc\u062a\u200c\u0627\u0637\u0644\u0627\u0639\u0627\u062a\u200c\u0633\u0644\u0627\u0645\u062a');
}
```

Place all shared/auth helpers in a central `tests/Browser/helpers.php`, loaded once via `require_once` in `tests/Pest.php` (avoid duplicate function declarations). Module-specific helpers (hardware/tickets/todo) stay in their own `helpers.php` with distinct function names.

### Basic page test

```php
it('loads the login page', function () {
    $page = visit('/login');

    $page->assertSee('ورود')
         ->assertSee('کد ملی')
         ->assertNoJavascriptErrors();
});
```

### Authenticated test (using a shared login helper)

```php
beforeEach(function () {
    createBrowserUser();           // seed perms + person + user
    $this->page = loginViaBrowser();
});

it('loads the users page', function () {
    $page = $this->page->navigate('/users');
    $page->assertSee('کاربران')->assertNoJavascriptErrors();
});
```

### Interaction selectors — target rendered DOM, NOT wire:model

`[wire:model.live.debounce="search"]` is an INVALID CSS selector (`querySelectorAll` throws). MaryUI `<x-input wire:model.live.debounce="search" placeholder="جستجو...">` renders a bare `<input placeholder="جستجو... ">`. Target `[placeholder="جستجو..."]`, a field `#id`, or the visible text — never the wire attribute.

```php
$page->type('[placeholder="جستجو..."]', 'محمد')->assertSee('محمد');
```

Note: assert on what is VISIBLE. A `<option>` inside a closed `<select>` is not visible — `assertSee('غیرفعال')` on a dropdown fails even though the option exists in the DOM. Assert the select's presence, not its hidden options

### Available assertions

- `assertSee($text)` — text visible on page
- `assertDontSee($text)` — text not visible
- `assertSeeInElement($selector, $text)` — text in specific element
- `assertNoJavascriptErrors()` — no JS errors in console
- `assertNoConsoleLogs()` — no console.log output
- `assertScreenshotMatches()` — visual regression

### Available interactions

- `click($selector)` — click element
- `type($selector, $text)` — type into input
- `press($text)` — click button/link by text
- `select($selector, $value)` — select dropdown option
- `check($selector)` — check checkbox
- `waitFor($selector)` — wait for element to appear
- `scrollTo($selector)` — scroll to element

## Running

```bash
# All browser tests
php artisan test --testsuite=Browser

# Single file
php artisan test tests/Browser/LoginTest.php

# Via composer
composer test:browser
```

## Pitfalls

- **`Target class [config] does not exist`** — the Pest HTTP server started an empty Laravel container. Fix: add `uses(Tests\TestCase::class)->in('Browser')` to `tests/Pest.php`.
- **`Call to a member function sendText() on null` / `ServerNotFoundException` in `beforeEach`** — you called `visit()`/`fill()` before the browser runtime booted. Call `bootBrowser()` first (see above).
- **`TypeError: ... must be of type Webpage, AwaitableWebpage returned`** — login helpers must declare `AwaitableWebpage`, not `Webpage`.
- **`SyntaxError: ... is not a valid selector` on `[wire:model...]`** — never target wire:model attributes; use `[placeholder]`, `#id`, or visible text.
- **`assertSee('option-text')` fails on a closed `<select>`** — hidden `<option>`s are not visible. Assert the select's presence instead.
- **Duplicate function declarations** — keep shared helpers in one central `tests/Browser/helpers.php` required once from `tests/Pest.php`; don't redefine `loginViaBrowser()` per test file.
- **Test database must exist and have migrations** — the server connects to the real DB, not in-memory. Create it and run migrations before first test run.
- **Browser tests are slower** than unit tests — use them for critical flows only, not every assertion.
- **Flaky tests** — add `waitFor()` before assertions on dynamically loaded content.
- **Persian/Arabic text** — use `assertSee()` with exact Unicode; do not normalize.
- **Headless Chrome gpu-process orphan hangs** — in some CI/envs a `chrome-headless-shell --type=gpu-process` spins at ~99% CPU and the run never exits; kill orphaned `playwright`/`chrome` processes before re-running, and don't leave full-suite runs to hang on it.