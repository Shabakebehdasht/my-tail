# Audit Playbook

What to look for, per category. Each subagent gets the relevant section plus the **Finding format**.

## 1. Correctness / Bugs
- Error handling: swallowed exceptions, empty catch blocks, missing error states.
- Async hazards: unawaited promises, race conditions, missing cleanup.
- Null/undefined flows: non-null assertions on nullable values, unchecked array indexing.
- Boundary conditions: off-by-one, empty-collection handling, timezone assumptions.
- State machines: impossible-state combinations, unhandled enum branches.
- Concurrency: check-then-act on shared resources, missing transactions.
- Type escape hatches: `any` / `as` casts / `@ts-ignore` clusters.
- Resource leaks: unclosed handles, connections, subscriptions; missing `finally`.

## 2. Security
- Credential hygiene: hardcoded keys/tokens/passwords, credentials in committed .env files.
- Data crossing into interpreters: SQL/shell injection, XSS, path traversal.
- Access control: missing server-side identity checks, IDOR, CSRF.
- Input contracts: unvalidated request bodies, file uploads without constraints.
- Dependency posture: run `npm audit`/`pip-audit` in read-only mode.
- Production config: overly broad CORS, missing security headers.
- Data minimization: PII in logs, stack traces to clients.

## 3. Performance
- N+1 patterns: query/fetch per item in loops.
- Wrong complexity: nested scans, repeated find/filter in hot loops.
- Caching gaps: identical expensive computations repeated per request.
- Payload size: over-fetching, missing pagination, large JSON.
- Frontend: bundle composition, missing code-splitting, unoptimized assets.
- Backend: sync work that belongs in a queue, missing indexes.

## 4. Test Coverage
- Map critical paths and check coverage.
- High-churn + no tests = top refactor risk.
- Existing test quality: meaningful assertions, no snapshot overuse.
- Missing test layers: unit vs integration vs E2E.

## 5. Tech Debt & Architecture
- Duplication: same logic in 3+ places.
- Layering violations: circular dependencies, junk-drawer utils.
- Dead code: unexported unused modules, stale feature flags.
- God objects/modules: files much larger than median.
- Inconsistent patterns: multiple ways of doing the same thing.

## 6. Dependencies & Migrations
- Major-version lag on core framework.
- Deprecated APIs in use.
- Abandoned dependencies on critical paths.
- Duplicate dependencies solving the same problem.

## 7. DX & Tooling
- Missing/broken typecheck, lint, formatter, pre-commit hooks.
- Slow feedback loops.
- Onboarding friction: wrong README steps, undocumented env vars.

## 8. Docs
- Public API surface without reference docs.
- Architectural decisions nobody can reconstruct.
- Stale docs that are actively wrong.

## 9. Direction — features & where to take this next
- Unfinished intent: TODO/FIXME clusters, unused feature flags.
- Stated-but-undelivered: README promises with no code.
- Surface asymmetries: CRUD minus one, one-directional pairs.
- The adjacent possible: capabilities the architecture makes cheap.
- Friction worth productizing: things users do by hand.

## Finding format
```markdown
### [CATEGORY-NN] Short imperative title
- **Evidence**: `path/file.ts:123` — description.
- **Impact**: Concrete cost.
- **Effort**: S / M / L
- **Risk**: LOW / MED / HIGH
- **Confidence**: HIGH / MED / LOW
- **Fix sketch**: 1–3 sentences.
```

## Prioritization
Order by leverage = impact / effort, discounted by confidence and fix-risk.