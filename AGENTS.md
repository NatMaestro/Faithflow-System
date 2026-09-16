# AGENTS.md — Engineering Guardrails

**Read this file in full before your first tool call in a session.**
If a session instruction conflicts with a rule here, the human wins — but say out loud which rule you are overriding and why.

Every rule here is **checkable**: verifiable from a diff or by running a command. If a rule can't be checked, it doesn't belong in this file.

Sections 1–16 apply to every system regardless of size or architecture. Section 17 holds optional modules — keep the ones that apply, delete the rest.

---

## 0. Project config — fill this in, keep it current

| Key | Value |
|---|---|
| System shape | 2 services: Django REST API + React SPA. Separate repos, wired here as git submodules. |
| Language / runtime | Python 3.11.9 (`faithflow-backend/runtime.txt`) · TypeScript 5.8 on Node 22 |
| Package manager | pip (`requirements.txt`) · npm (`package-lock.json`) — **see ⚠️ below** |
| Run tests | Backend: `python manage.py test tests.test_tenant_binding tests.test_subscription_webhook tests.test_queryset_scoping tests.test_role_permissions` · Frontend: **none — see ⚠️ below** |
| Lint | Frontend: `npm run lint` · Backend: **not configured — see ⚠️ below** |
| Typecheck | Frontend: `npm run typecheck`, gated as `npm run ratchet:typecheck` · Backend: n/a |
| Build | Frontend: `npm run build` · Backend: n/a |
| Data stores | PostgreSQL (Neon), schema-per-tenant via `django-tenants`. Redis is declared in `requirements.txt` and `docker-compose.yml` but unused — the cache falls back to `LocMemCache` unless `REDIS_URL` is set. |
| Protected branches | `main` |
| Max files per change | `<FILL>` |
| Deploy mechanism | Backend: Render, `autoDeploy` from `main` (`faithflow-backend/render.yaml`) · Frontend: GitHub Actions → S3 + CloudFront invalidation (`.github/workflows/deploy.yml`) |
| Rollback mechanism | `<FILL>` |
| Who to escalate to | `<FILL>` |

**Active modules from §17:** multi-tenancy · financial `<confirm>` · regulated-data `<confirm>`

Anything still marked `<FILL>` is undecided. **Do not decide it yourself — ask.**

### ⚠️ Gaps found while filling this in

These are stated, not fixed — §3 says log unrelated findings rather than acting on them.

1. **The frontend has no test runner.** No `test` script, and no vitest/jest/testing-library
   in `package.json`. §9 ("every behavior change ships with a test") is currently
   unsatisfiable on the frontend — every frontend change violates it by construction.
2. **Two lockfiles.** `bun.lockb` and `package-lock.json` both exist; CI runs `npm ci`, so
   the bun lockfile is stale and misleading. Pick one.
3. **Backend lint is declared but not runnable.** `black`, `flake8` and `isort` are in
   `requirements.txt` with no `setup.cfg`, `.flake8`, `pyproject.toml` or CI step.
4. **§13 references "the interactive budget in §0"**, but no such row exists in this table.
   Either add the budget or drop the reference.

### Verified as already compliant

- §5 monetary values: `DecimalField(max_digits=10, decimal_places=2)` throughout
  `apps/payments/models.py` and `apps/churches/models_subscription_payment.py`. No floats.

---

## 1. Prime directives

1. **Read before you write.** Cite real file paths and real symbols. Never invent an API, config key, field, or endpoint. If you haven't opened the file, you don't know what's in it.
2. **Smallest change that solves the problem.** No opportunistic refactors.
3. **Never leave the tree broken.** Tests, lint, and typecheck pass before you declare done.
4. **If you are guessing, stop and ask.** A wrong confident answer costs more than a question.
5. **No silent scope expansion.** New problem discovered → note it, don't fix it.
6. **Match the existing pattern.** Consistency beats your preferred style. If the pattern is wrong, say so and ask before deviating.
7. **Truth over agreement.** "This approach won't work because X" is a required output, not an optional one.
8. **Report failures loudly.** Never describe partial work as complete. Never hide a failing test or a command you didn't run.

---

## 2. HARD STOPS — never without explicit human approval in-session

Stop and ask. Do not proceed on inference, urgency, or approval given in a previous session.

- Destructive schema or data operations: drop, truncate, type narrowing, irreversible backfill.
- Anything touching production: data, credentials, DNS, config, live infrastructure.
- Reading, writing, printing, or committing secrets, keys, tokens, credential files.
- Deleting, skipping, or weakening an existing test.
- Adding a new third-party dependency.
- Changes to authentication, authorization, or session handling.
- Changes to a published interface other code depends on — API, event schema, shared library signature.
- Changes to CI config, build pipelines, container definitions, or infrastructure-as-code.
- Applying infrastructure changes to any shared or remote environment.
- Changes spanning more than the max-files limit in §0.
- Force pushes, history rewrites, direct commits to a protected branch.
- Real customer data or PII in fixtures, logs, tests, or prompts.
- Anything flagged by an active module in §17.

---

## 3. Scope discipline

- One task per branch. One logical change per commit.
- Any change likely to exceed ~50 lines: post a plan and wait for a go-ahead.
- Found an unrelated bug? Log it. Do not fix it here.
- Do not reformat, reorder imports, or rename things you weren't asked to touch — churn hides real changes in review.
- Do not add abstraction for a second use case that does not yet exist.
- Do not introduce a new pattern, library, or architectural concept to solve a local problem.

---

## 4. Boundaries and interfaces

The same discipline applies whether a boundary is a module import or a network call. Only the cost of getting it wrong changes.

- Respect the declared dependency direction. No cycles between modules or services.
- Cross a boundary only through its published interface. Never reach into another component's internals, tables, or storage.
- Business logic lives behind a service/domain layer. Controllers, handlers, jobs, CLI commands, and event consumers are thin adapters — parse, delegate, format.
- The same logic must be reachable from every entry point. No behavior that exists only in the HTTP path or only in the async path.
- A new or changed cross-boundary interface is documented in the same change.
- If a change requires touching two components at once, that is a design signal. Say so before proceeding.

---

## 5. Data and state

- Ownership is exclusive: one component owns each piece of data and is the only writer. Everyone else reads through its interface.
- All input is validated and normalized at the system boundary. Trust nothing from a client — including your own.
- Timestamps stored in UTC, timezone-aware. Never a naive local clock.
- Monetary and precision-sensitive values: fixed-point or integer minor units. **Never floating point.**
- Identifiers are opaque and non-guessable where they are externally visible.
- Every externally-triggered write path is idempotent and keyed.
- No raw query strings outside the designated data-access layer. When unavoidable: parameterized only, plus a comment saying why the abstraction wasn't enough.

---

## 6. Contracts and compatibility

- Assume an old client, an old consumer, and an old message are in flight right now. They usually are.
- Additive changes only by default. Removing or renaming a field, or narrowing a type, is a breaking change and a hard stop.
- Every payload that crosses a boundary carries a version.
- Breaking changes ship as expand → migrate → contract, across at least two deploys, never one.
- Deprecate before deleting: announce, measure remaining usage, then remove.
- Consumers tolerate unknown fields. Producers never rely on strict-mode consumers.

---

## 7. Failure and distribution

Applies as soon as anything crosses a process boundary — a network call, a queue, a job runner, a third-party API.

- Every remote call has an explicit timeout. No unbounded waits, ever.
- Every retry is on an idempotent operation, uses backoff with jitter, and has a bounded attempt count.
- Delivery is at-least-once. Every consumer must be replay-safe and handle duplicates.
- Handle partial failure explicitly: state what happens when step 3 of 5 fails. "It won't" is not an answer.
- No distributed transactions. Use an outbox, saga, or compensating action, and name which one you chose.
- Ordering across independent producers is not guaranteed. Do not assume it.
- Every dependency failure has a defined degradation: fail closed, fail open, or queue. Choose deliberately and document it.

---

## 8. Security

- Secrets come from a secret manager or the environment at runtime. Never in code, config, comments, tests, or logs.
- Authorization is enforced server-side in the service layer, not only in the UI or the request serializer.
- Every service-to-service call is authenticated. Network position is not identity.
- No `eval`-class execution, no deserializing untrusted input, no shell invocation with interpolated user input.
- New endpoints are authenticated and rate-limited by default; opting out requires a comment stating why.
- Log identifiers, never payloads. No PII, tokens, credentials, or full request bodies in logs.
- Pin dependency versions. Flag known CVEs rather than silently upgrading.

---

## 9. Testing

- Every behavior change ships with a test in the same change. No test, no merge.
- Bug fix workflow: write the failing test first, then fix. Show red-to-green.
- Test names state behavior: `test_retry_does_not_double_charge`, not `test_payment_2`.
- Unit tests hit no network, no real clock, no real filesystem, no sleeping.
- Use factories or builders. Never large fixture dumps.
- Never mock the unit under test. Mock at boundaries you own.
- Every cross-boundary interface has a contract test that fails when either side drifts.
- Cover the edges: empty, null, zero, negative, duplicate, concurrent, out-of-order, unauthorized.
- Don't test the framework. Test your logic.

---

## 10. Schema and migrations

- Every migration is reversible, or documents why it cannot be.
- Schema changes and data changes are separate migrations.
- Never a breaking schema change in a single deploy. Expand, migrate, contract.
- No blocking operations on large tables. Nullable-then-backfill, concurrent index creation.
- Backfills run in bounded batches with a resume point. Never one unbounded update.
- Never edit a migration that has been applied outside your machine.
- Verify against realistic data volume before proposing.

---

## 11. Infrastructure and deploy

- Infrastructure changes go through code. No console clicks, no live edits.
- Always dry-run or plan and show the output before proposing to apply.
- Every deploy-affecting change states its rollback path.
- Deploys are independently revertible. A change requiring two components to ship simultaneously is a design failure — flag it.
- Workloads declare resource requests and limits. Nothing unbounded.
- Least privilege always: no wildcard permissions, no open ingress, no shared admin credentials.
- Health and readiness checks on every new service or worker.
- Feature-flag anything risky so rollback doesn't require a redeploy.

---

## 12. Observability

- Structured logs with stable keys. Never bare interpolated strings for machine-read events.
- Every request and job carries a correlation/trace ID, propagated across every boundary it crosses.
- Never swallow an exception. Handle it, log it with context, or let it propagate.
- New endpoint, consumer, or job → latency, throughput, and error-rate metrics in the same change.
- Instrument the failure paths, not just the happy path. Retries, timeouts, and fallbacks each emit a signal.
- An alert without a runbook link is not done.

---

## 13. Performance

- No repeated-query patterns in loops. Assert query or call counts in tests for list and batch paths.
- Every new filter, lookup, or sort has a supporting index, or a note explaining why not.
- All collection endpoints are paginated. No unbounded result sets reaching a serializer.
- Work that can exceed the interactive budget in §0 moves to a background job.
- Set explicit limits: batch sizes, payload sizes, concurrency, queue depth.
- Don't optimize without a measurement. Show before and after.

---

## 14. Definition of Done — paste this filled in with every change

```
- [ ] Requirement restated in my own words, and it matches what was asked
- [ ] Tests written and passing:            <command + result>
- [ ] Lint + typecheck clean:               <command + result>
- [ ] Interfaces changed:                   <none | listed, back-compat verified>
- [ ] Schema/migration reviewed:            <yes | n/a>
- [ ] Failure paths handled:                <timeouts, retries, partial failure | n/a>
- [ ] Security-sensitive paths touched:     <no | listed + flagged>
- [ ] §17 modules implicated:               <none | listed>
- [ ] Files changed:                        <count>
- [ ] Rollback plan:                        <one line>
- [ ] Out of scope, found along the way:    <list, or none>
- [ ] Assumptions I made:                   <list, or none>
```

An unchecked box is not "done with a caveat." It is not done.

---

## 15. Escalation protocol

**Stop and ask** when:
- The requirement is ambiguous, or two readings are equally plausible.
- Two valid designs exist and the choice has lasting consequences.
- The work touches anything in §2 or an active §17 module.
- A test fails for a reason you don't understand.
- The change would exceed the max-files limit.
- You've attempted the same fix twice and it still fails.
- The clean solution requires changing something you were told not to change.

**Format:**
```
What I was doing:
What I found (with file:line references):
Options: A) ... B) ...
My recommendation and why:
What I need from you:
```

Ask early. A question after ten minutes is cheap. A wrong assumption discovered after three hundred lines is not.

---

## 16. Auto-reject anti-patterns

- A test "fixed" by deleting or loosening an assertion.
- Catch-all exception handling that discards the error.
- `sleep()` used to paper over a race condition.
- Mocking the unit under test.
- Retry wrapped around a non-idempotent operation.
- A timeout left unset because "the default is fine."
- Commented-out code left in the diff.
- `TODO` without a ticket reference.
- Magic numbers instead of named constants.
- A third copy of the same block instead of extracting it.
- "Should work" / "probably fine" in a completion report. Verify, or state that you didn't.
- Claiming a command succeeded without showing its output.

---

## 17. Optional modules — keep what applies, delete the rest

### Multi-tenancy
- Every query touching tenant data is tenant-scoped. No exceptions — including admin tools, exports, and background jobs.
- Every new tenant-scoped entity ships with a test proving a cross-tenant read returns nothing.
- Tenant context is derived server-side from the authenticated principal. Never from a client-supplied parameter.
- Cross-tenant operations are a hard stop.

### Financial / ledger
- Double-entry postings balance to zero, enforced by the engine, not by convention.
- Posted entries are immutable. Corrections are reversal plus repost, never an update.
- Rounding is defined once, in one place, and imported everywhere.
- Rates and rules are versioned by effective date. Never edit history in place.
- Every calculation has a test with a worked example including the rounding edge case.
- All changes here are hard stops.

### Offline-first / intermittent clients
- The client clock is untrusted. The server assigns authoritative timestamps.
- Conflict resolution is deterministic and documented per entity. "Last write wins" is a decision to be written down, not a default to assume.
- Per-device ordering is preserved; cross-device ordering is not.
- Every payload carries a schema version. Old clients will exist in the field.

### Event-driven
- Events describe facts that happened, in the past tense. They are not commands.
- Event schemas are append-only and versioned.
- Consumers are independently deployable and independently failing. One slow consumer must not stall a producer.
- Every consumer has a dead-letter path and a documented replay procedure.

### Regulated or sensitive data
- Data classification is declared per field before storage.
- Access to sensitive fields is logged with actor, purpose, and timestamp.
- Retention and deletion rules are implemented, not just documented.
- Any change to what is collected, stored, or exported is a hard stop.

---

## How this file scales

The rules don't change with size; the enforcement does.

| | Small team / single service | Large org / many services |
|---|---|---|
| §4 Boundaries | Code review and import linting | Ownership registry, API gateway |
| §6 Contracts | Same-repo tests | Schema registry, consumer-driven contract tests |
| §7 Failure | Timeouts and retries | Circuit breakers, bulkheads, load shedding |
| §11 Deploy | Single pipeline, manual revert | Progressive rollout, automated rollback |
| §12 Observability | Structured logs | Distributed tracing, SLOs, error budgets |

If you are small, the rules still apply — they are cheap now and expensive to retrofit later. If you are large, this file is the shared floor; each component may add stricter local rules, never looser ones.

---

## Maintaining this file

This file mirrors what CI enforces — it does not replace it. Every rule should eventually have a lint rule, a test, or a review gate behind it. A rule CI never checks will be broken quietly.

When a rule is repeatedly violated, the rule is unclear or wrong. Fix the rule.
Keep this file under ~300 lines. Beyond that it gets skimmed instead of read — push detail into per-component files instead.

Last reviewed: `<date>` · Owner: `<name>`
