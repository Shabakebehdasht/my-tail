---
name: livewire-debugging
description: "Use when debugging Livewire performance or UI failures."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [livewire, laravel, performance, debugging, ui, silent-failure]
    category: software-development
    related_skills: [systematic-debugging]
---

# Livewire Debugging

## Overview

Livewire updates are round-trips: every user action triggers a full server-side re-render of the component. Expensive operations that run on every update — heavy queries, unbounded collections, eager-loaded relationships — cause silent failures: the server takes too long, the response is too large, or Livewire gives up. **The browser shows no console error.** The UI element simply doesn't appear, disappears mid-update, or feels sluggish.

## The Core Rule

**Any query or collection that runs on every Livewire update MUST be guarded.** A computed property, `with()` method, or `wire:model.live`-triggered listener that executes `Model::all()`, `Model::get()`, or an unfiltered query without a conditional guard will cause performance degradation and silent render failures.

## Diagnosis: Silent Livewire Failures

When the user reports:
- "Button doesn't open anything" (no console error)
- "Results disappear randomly"
- "Page feels slow after searching"
- "Modal/form doesn't show"

### Step 1: Identify the expensive operation

Find computed properties and `with()` methods in the component's inline PHP class (top of the Blade file):

```bash
# Find all computed properties and with() in Livewire blade files
search_files(pattern="function.*Property\(\)|function with\(\)", path="resources/views/livewire", file_glob="*.blade.php")
```

Check each one: does it run a query? Does the query have a guard condition?

### Step 2: Check for missing guards

Common unguarded patterns:

```php
// BAD: runs on every update regardless of context
public function getFilteredPersonsProperty()
{
    return Person::query()->get(); // loads ALL rows every update
}

// BAD: conditional on a property that changes on every keystroke
public function getFilteredPersonsProperty()
{
    return Person::query()
        ->when($this->search, fn($q) => $q->where(...))
        ->get(); // when search is empty, loads ALL rows
}
```

### Step 3: Apply the guard pattern

```php
// GOOD: guard on context + minimum input length + limit
public function getFilteredPersonsProperty(): array
{
    if (! $this->formOpen || mb_strlen($this->search) < 2) {
        return [];
    }

    return Person::query()
        ->where(...)
        ->limit(20)
        ->get();
}
```

Three-part guard:
1. **Context guard** — only run when the feature is active (e.g., `formOpen`, modal is visible)
2. **Input guard** — minimum character count prevents empty-table scans
3. **Result limit** — prevents unbounded payload even with a good filter

## The Five Livewire Performance Pitfalls

### 1. Computed properties without guards

Livewire re-runs every `#[Computed]` property and every method called from `with()` on every update. If the component is embedded in a table row, every row expansion re-runs it too.

**Fix:** Guard with `if (! $this->someFlag) return [];` and add `limit`.

### 2. `wire:model.live` on expensive reactive properties

`wire:model.live` fires a Livewire update on every keystroke. If the update triggers a computed property that runs a query, every keystroke runs a query.

**Fix:** Use `wire:model.live.debounce.300ms` on search inputs, and always guard the computed property.

### 3. `with()` loading unbounded relationships

The `with()` method runs on every render. Loading relationships with `->get()` inside `with()` creates N+1 query bursts on complex pages.

**Fix:** Use `withAggregate` or `withCount` for simple aggregations. Scope relationship loading with `->limit()`.

### 4. Component re-renders from parent state changes

When a parent component updates (e.g., a Livewire table re-paginates), all child/expanded components re-render. If an expanded row has a heavy `with()` method, every page change triggers it.

**Fix:** Use `#[Computed(except: ['search'])]` to skip re-computation when unrelated properties change.

### 5. Large Blade payloads

Even if the query is fast, the rendered HTML payload can be large. Livewire diffs the full HTML on the client side. Payloads over ~100KB cause perceptible lag.

**Fix:** Paginate inline data, use `wire:init` for deferred loading, or move heavy displays to separate Livewire components with `lazy` loading.

## Debugging Checklist

When a Livewire component misbehaves silently:

1. **Read the inline PHP class** at the top of the Blade file — find all computed properties, `with()` method, and event listeners
2. **Check each query** for a guard condition (is it context-aware? input-guarded?)
3. **Check for `wire:model.live`** — does the triggered update run a query?
4. **Measure the response** — add `ray()->measure()` or check Laravel Boost `browser-logs` for long Livewire round-trips
5. **Apply the three-part guard** and re-test

## Pitfalls

- **No console error ≠ no problem.** Livewire silent failures are server-side: the render times out or returns partial HTML, and the browser applies whatever it got. Check `browser-logs` via Laravel Boost MCP or `storage/logs/laravel.log`.
- **`Model::all()` in a `with()` method is almost always wrong.** It runs unconditionally on every render.
- **Don't just add `->limit(100)`.** Without a context guard, the query still runs on every update — it just returns fewer rows. Both are needed.
- **`mb_strlen` not `strlen`** for Persian/Arabic input — multibyte characters need proper length checking.
- **Persian normalization on search input** — apply `PersianNormalizer::normalizeForSearch()` (or equivalent) before LIKE queries to match ي/ك variants and normalize ZWNJ.