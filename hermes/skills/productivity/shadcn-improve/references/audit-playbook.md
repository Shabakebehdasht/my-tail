# Audit Playbook

What to look for, per category. Each subagent (or direct audit pass) gets the relevant section plus the **Finding format** at the bottom.

A finding is only a finding with evidence.

---

## 1. Correctness / Bugs

- Error handling: swallowed exceptions, empty catch blocks, missing error states in UI code.
- Async hazards: unawaited promises, race conditions, missing cancellation/cleanup.
- Null/undefined flows: non-null assertions on nullable values, optional chaining hiding required values.
- Boundary conditions: off-by-one, empty-collection handling, timezone assumptions.
- State machines: impossible-state combinations, status enums with unhandled branches.
- Concurrency: check-then-act, missing transactions, idempotency of retried operations.
- Type escape hatches: `any` / `as` casts / `@ts-ignore` clusters.
- Resource leaks: unclosed handles, connections, subscriptions; missing `finally`.

## 2. Security

**Handling rule:** never copy a secret value into a finding or plan. Reference `file:line` and credential type only.

- Credential hygiene: hardcoded keys/tokens, credentials in committed `.env` files.
- Data crossing into interpreters: SQL/shell injection, XSS, path traversal.
- Access control: endpoints lacking server-side checks, IDOR, missing CSRF.
- Input contracts: API boundaries trusting request bodies without validation.
- Dependency posture: `npm audit`, `pip-audit`, `cargo audit` — critical/high only.
- Production configuration: overly broad CORS, missing security headers.
- Data minimization: PII in logs, stack traces to clients, internal errors in responses.

## 3. Performance

- N+1 patterns: query/fetch per item inside loops.
- Wrong complexity: nested scans, repeated find/filter in hot loops.
- Caching gaps: identical expensive computations repeated per request/render.
- Payload size: over-fetching, missing pagination, large JSON.
- Frontend: bundle composition, missing code-splitting, unoptimized assets.
- Backend: sync work in queues, missing indexes, connection-per-request.

## 4. Test Coverage

- Map critical paths (money, auth, data mutation) — check coverage.
- High churn + no tests = top refactor risk.
- Existing test quality: meaningless assertions, heavy mocking, flaky patterns.
- Missing test layers: unit-only with zero integration, or slow E2E for unit-catchable.
- Verification infrastructure: is there a one-command way to know the codebase works?

## 5. Tech Debt & Architecture

- Duplication: same logic in 3+ places, divergent copies.
- Layering violations: UI importing data layer internals, circular dependencies.
- Dead code: unexported-unused modules, feature flags fully rolled out.
- God objects: files an order of magnitude larger than median.
- Inconsistent patterns: three ways of doing the same thing.

## 6. Dependencies & Migrations

- Major-version lag on core framework/runtime.
- Deprecated APIs with announced removal timelines.
- Abandoned dependencies on critical paths.
- Duplicate dependencies solving the same problem.

## 7. DX & Tooling

- Missing or broken: typecheck, lint, formatter, pre-commit hooks.
- Slow feedback loops: dev-server/test startup in minutes.
- Onboarding friction: wrong README steps, undocumented env vars.
- Missing `AGENTS.md` — high-leverage for agent-executed repos.

## 8. Docs

- Public API surface without reference docs.
- Undocumented architectural decisions.
- Stale docs that are actively wrong.

## 9. Direction

- Unfinished intent: TODO/FIXME clusters, never-rolled-out feature flags.
- Stated-but-undelivered: README promises with no code.
- Surface asymmetries: one-directional pairs, CRUD minus one.
- The adjacent possible: capabilities the architecture makes cheap.
- Friction worth productizing: things users do by hand around the project.

---

## Finding format

```markdown
### [CATEGORY-NN] Short imperative title

- **Evidence**: `path/file.ts:123` — description.
- **Impact**: What goes wrong / cost.
- **Effort**: S / M / L — for the fix, including tests.
- **Risk**: What the fix could break; LOW/MED/HIGH.
- **Confidence**: HIGH / MED / LOW.
- **Fix sketch**: 1–3 sentences.
```

## Prioritization rubric

Order by leverage = impact ÷ effort, discounted by confidence and fix-risk.
1. Anything that unblocks other findings floats up.
2. HIGH-confidence security findings float above equivalent non-security.
3. Prefer findings whose fix has a clean verification story.
4. "Not worth doing" is a valid verdict with one line of reasoning.