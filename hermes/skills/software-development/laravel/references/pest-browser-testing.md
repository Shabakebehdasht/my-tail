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

### Basic page test

```php
it('loads the login page', function () {
    $page = visit('/login');

    $page->assertSee('ورود')
         ->assertSee('کد ملی')
         ->assertNoJavascriptErrors();
});
```

### Livewire component test

```php
it('searches users', function () {
    $page = visit('/users')
        ->actingAs(User::factory()->create(['n_code' => '1234567890']));

    $page->type('[wire:model="search"]', 'محمد')
         ->assertSee('محمد');
});
```

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
- **Test database must exist and have migrations** — the server connects to the real DB, not in-memory. Create it and run migrations before first test run.
- **`press('ورود')` finds nothing** — button text in Blade may differ from rendered text (e.g., `ورود به سیستم`). Check with `assertSee()` first.
- **Browser tests are slower** than unit tests — use them for critical flows only, not every assertion.
- **Flaky tests** — add `waitFor()` before assertions on dynamically loaded content.
- **Persian/Arabic text** — use `assertSee()` with exact Unicode; do not normalize.
- **Empty Livewire `press()` hangs** — if the button triggers a heavy Livewire update, the test times out. Guard computed properties before writing form submission tests.