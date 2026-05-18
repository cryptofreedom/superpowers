---
name: test-driven-development
description: |
  Dual-loop TDD framework for the AI era. Activate when the user asks to
  write/generate tests, do TDD, add unit tests for existing code, modify
  or extend an existing module, or convert a PRD into test cases. The
  unit under test is a use case / a whole feature path through one
  service method, not a class — when middleware is in the path the test
  runs against a real Testcontainers container, otherwise plain JUnit,
  with the same scoping rule either way. Never mock middleware clients
  to simulate their semantics.
---

# Dual-Loop TDD Workflow

## Core Philosophy

Three concrete workflows, not one rigid TDD ceremony:

- **Outer loop — modification path** *(default for day-to-day work)*: lock current behavior with characterization tests **before** changing it. Catches silent regressions on bug fixes, extensions, and refactors.
- **Outer loop — greenfield path** *(reserved for high-stakes new features)*: PRD → acceptance tests. Use only when a real PRD exists and the feature is contract-heavy (payments, auth, external APIs, multi-team interfaces).
- **Inner loop**: adversarial tests against already-implemented code — attack the code, not validate it.

The two outer-loop paths are not interchangeable. Modification work rarely comes with clean acceptance criteria — its risk is **silent regression**, not contract violation. Greenfield work has a contract from day one — its risk is **shipping the wrong thing**.

## Project test architecture

**What counts as a unit.** The unit is a use case / a whole feature path through one service method, not a class or method. A test that splits a behavior into "data-layer test + pure-logic test with a mocked data layer" duplicates seeding, leaves the seam unprotected, and tests the mock's shape rather than the SQL — it is the anti-pattern this whole skill exists to push back on. This applies regardless of whether middleware is in the path: when middleware is, the unit is exercised through a real Testcontainers container; when it isn't, plain JUnit — but **the boundary of what counts as one test does not change**.

Whenever the code under test touches a middleware client (DataSource / `BaseMapper`, `RedisTemplate` / `RedissonClient`, `KafkaTemplate` / broker client, ...), wire it to a real Dockerized container — MySQL via `AbstractMySqlIT`, Redis / broker via the matching `AbstractXxxIT` (build the base mirroring `AbstractMySqlIT` if missing). Mocking the middleware client to simulate its semantics — partial-update FieldStrategy, TTL math, ack flow, hand-rolled JOIN return shapes — almost always tests the mock's behavior instead of the real one. **Reason about the bug surface from the production code, not from what existing assertions happen to cover** — assertions are bounded by what mocks can expose.

**Where you point the integration test follows from the bug surface, not from a separate category.** The default for connector code is to point it at the service method: instantiate the service with `new`, inject the real mapper via reflection, mock only true external collaborators (Feign clients, `ICATPlatformService`, `IInputPlatformService`, `RedisTemplate`, brokers), seed real rows, call the method, assert on the return value plus DB state. When the SQL itself deserves a focused red/green independent of any caller — see "When does the SQL deserve its own integration test" below — point the test at the mapper directly. The same logic applies for Redis cache/lock behavior or broker delivery: scope the test to whichever level the bug surface lives at.

Don't pre-emptively split a service-scoped test into "mapper-scoped for SQL + service-scoped with a mocked mapper for the decision." Hand-rolled mocks of multi-column JOIN returns (JSON typeHandler, enum / bit, NOT NULL DEFAULT) are almost certainly the wrong shape, the split forces the same business path to be seeded twice, and the seam between the halves is unprotected. The container is JVM-shared and lazy-started — going through real SQL costs almost nothing per test.

If the behavior under test happens to need no middleware client, no container is needed — but the unit-as-behavior rule still applies. Don't carve out a sub-method to test in isolation just because it's pure; test the behavior its caller exposes, mocking only true external collaborators.

What's deliberately out of scope:

- Controller HTTP slices (URL routing / `@Validated`) — cost/value ratio is poor for a connector system; rely on type checking + manual smoke
- External CAT / Phrase / Lokalise API contracts — mock at the `ICATPlatformService` / `IInputPlatformService` boundary
- Cross-service E2E — too brittle in a system this connector-heavy

If you find yourself wanting another tier, push back: it usually means a unit-level seam is in the wrong place, not that the architecture is missing a layer.

Files live in the package of the code under test (reference: `TaskMapperPartialUpdateTest` at `domain/mapper/`, next to `TaskMapper`; the service-scoped counterpart sits next to its `*ServiceImpl`). **No separate `integration/` folder, no `*IT.java` suffix, no Failsafe profile** — `mvn test` runs everything through Surefire.

How to wire one up — see [references/testcontainers-mysql-guide.md](references/testcontainers-mysql-guide.md) (covers MySQL specifics; the same pattern — base class, container, client factory, per-test isolation hook — applies to Redis / broker / others).

## Tests as executable spec — for human and future-agent readers

Tests in this project are not throwaway dev artifacts. They are the primary documentation a future reader (human or AI agent) consults when changing the code under test. Write tests so a reader who has **not seen the implementation** can extract the business contract from the test alone. Why this matters more in 2026 than in 2020: every non-trivial refactor on this codebase will eventually be done by an AI agent reading the test suite. If `@DisplayName` reads as `"test case 1"` and there is no contract anchor, the agent has to reverse-engineer business intent from the implementation — exactly the failure mode this skill exists to prevent.

**Three rules** (apply to outer-loop, inner-loop, and middleware-backed tests alike):

1. **Class-level `Contract:` block**. Each non-trivial test class opens with a Javadoc block describing what the method-under-test must do, in business language (1–3 sentences). State the contract, not what the test does. Example:
   ```java
   /**
    * Contract: ConfigService.autoBind() must (a) look up the resource binding by workflow UID
    * + ERP resource ID, (b) overwrite the config row's smartlingName / memsourceName when a
    * binding is found, (c) leave them untouched when no binding exists. Soft-deleted bindings
    * (is_delete != 0) must not match.
    */
   class ConfigServiceImplAutoBindTest extends AbstractMySqlIT { ... }
   ```

2. **`@DisplayName` and test method names read as one spec line in business language, not test mechanics**. A reader scanning the test list should see a feature spec.
   - ✅ `@DisplayName("autoBind: existing binding for resource overwrites smartlingName")`
   - ✅ `void autoBind_existingBindingForResource_overwritesSmartlingName()`
   - ❌ `@DisplayName("test case 1")` / `void test_autoBind_branch_a()` / `void should_work()`

3. **Don't tie tests to ephemeral context**. Comments like `// added for #1234`, `// covers PR-567 bug`, `// the new behavior after the 2026-04 refactor` rot the moment the ticket / PR / refactor fades. Pin the test to the **business contract**, not the **engineering history** that produced it. Bug-history goes in commit messages and PR descriptions, not in tests.

If a test cannot be named in business language, that is a signal the unit boundary is wrong — re-check whether the test is scoped to a use case or to an internal method.

## Decision: is a test worth writing?

Most damage from naive AI-generated tests happens in connector / I/O-heavy code, where mocks dwarf the code under test, the mock shape silently drifts from real mapper output, and tests verify the mocks rather than behavior. Skip the test when it would not earn its keep.

| Signal | Action |
|---|---|
| Function body is mostly delegation to an SDK / HTTP client / message bus, no branching of its own | **Skip** — nothing to attack without testing the mock; coverage belongs to E2E (out of scope) |
| Mock setup in any reasonable test would be longer than the function itself | **Skip** — you'd be testing the mocks |
| Bug surface is "did the request reach the partner with the right shape" | **Skip** — external-API contract is out of scope; mock at `ICATPlatformService` / `IInputPlatformService` and stop |
| Function calls a mapper / repository whose return shape is non-trivial (JOIN, ≥15-column projection, JSON typeHandler, enum / bit, NOT NULL DEFAULT) and the function decides on that shape | **Integration test, scoped to the service method** — real mapper, real SQL, mock only true externals |
| Function calls a mapper but the call is a simple by-id select / update / delete with a flat DTO | **Plain JUnit test** with a mocked mapper is fine — still scoped to the behavior, not the inner method |
| Function has branching / transformation / validation / mapping logic with no DB | **Plain JUnit test** — still scoped to the behavior, not the inner method |
| Bug surface is real middleware behavior itself (SQL: partial update, JSON columns, joins, FieldStrategy, soft-delete; Redis: TTL, eviction, locks, Lua; broker: ack, redelivery, partition routing) and the behavior is worth pinning independently of any caller | **Integration test, scoped to the mapper / cache / broker** — but check first whether folding into a service-scoped test gives the same coverage with one test instead of two |

If nothing meaningful can be extracted **and** the middleware behavior itself isn't worth pinning, **say so to the user and stop**. Do not generate ceremonial tests.

**Final durability check**: will this test still be true in 6 months, or is it pinning a coincidence (a particular literal value, a current data shape that's expected to evolve)? If coincidence, don't write it.

### When does the SQL deserve its own integration test, separate from the calling service?

A mapper-scoped integration test is justified only when the SQL behavior deserves a focused red/green independent of any caller. Don't conflate "no service-scoped test covers this yet" with "this is an independent anchor" — those are orthogonal. The state-transition logic:

| Mapper method has a caller in main code? | The calling service has an integration test that already exercises this SQL path? | The SQL behavior is on the independent-anchor list?<sup>*</sup> | Action |
|---|---|---|---|
| ❌ No | — | — | **Dead code.** Delete the mapper method, its XML, and the mapper-scoped test together |
| ✅ Yes | ✅ Yes | ❌ No | **Fold into the service-scoped test.** Lift the assertions over (migrate → verify red → delete the mapper-scoped test); see [testcontainers-mysql-guide.md → "When to delete or rewrite"](references/testcontainers-mysql-guide.md#when-to-delete-or-rewrite) for the migration flow |
| ✅ Yes | ❌ No | ❌ No | **Write the missing service-scoped test.** Don't keep a mapper-scoped test as cover for a missing service-scoped test — that's the failure mode this section exists to call out |
| ✅ Yes | — | ✅ Yes | **Keep both.** The mapper-scoped test pins the SQL boundary; the service-scoped test pins the business path |

<sup>*</sup> Independent-anchor list: partial-update FieldStrategy, JSON_SET / JSON_EXTRACT depth + COALESCE edges, FIND_IN_SET CSV semantics, cross-DB / SP invocation, multi-table UPDATE row/table isolation, dynamic-SQL EXISTS branches, nested-subquery pagination, correlated-subquery / GROUP-MAX shapes. **Plain JSON typeHandler round-trip, simple deserialization, single-column LIKE, basic CRUD do not qualify** — they belong to the third row when they have a caller, or the first row when they don't.

The same state machine applies when deciding whether a Redis / broker integration test deserves a scope independent of its caller. Substitute the corresponding independent-anchor list (Lua script semantics, lock fencing tokens, ack / redelivery flow, partition-routing edges, ...).

## Picking the path

Once you've decided the test is worth writing, pick the workflow that matches the situation:

| Situation | Path |
|---|---|
| Modifying / extending / fixing existing behavior | → [Outer loop — modification](references/outer-loop-modification.md) |
| New, large, contract-heavy feature with a real PRD | → [Outer loop — greenfield](references/outer-loop-greenfield.md) |
| Code already implemented, needs adversarial coverage | → [Inner loop](references/inner-loop-guide.md) |
| Greenfield + need adversarial coverage | greenfield first, then inner loop |

**Default for ambiguous cases**: if you cannot state a concrete acceptance criterion from the PRD in one sentence, treat the work as a modification (characterization-first), not greenfield (PRD → Gherkin). Greenfield is the exception.

**Before entering the inner loop, run the outer-loop compensation check** (see "Outer-loop compensation" in [inner-loop-guide.md](references/inner-loop-guide.md)): if the method under test has no tests covering its core business logic, generate business-logic cases first, then adversarial cases.

## Higher-layer quality gates

### Mutation testing (PIT)

From repo root (PowerShell): `git-bash.exe scripts/pit-diff.sh` — diffs vs `origin/main`,
mutates only changed Java files, writes `target/pit-reports/diff-filtered.md`.

### Property-based testing (PBT) — piloting

Scoped to pure utilities. Methodology TBD.

## Core Principles

1. The outer loop's job is **constraining behavior** (preventing silent regression or contract violation), not validating that code works.
2. The inner loop's job is **attacking code** (finding bugs at the edges), not validating it.
3. No sycophantic tests — no tests that merely paraphrase the implementation's happy path.
4. **Distinguish value sources**: every data source the method under test can reach (parameters, injected fields, query return values) must be assigned mutually distinct values in the test. Mock expected values must be chosen from the contract's intent, not copied from variables used in the implementation. `""` / `null` / `0` / seed-default values are NOT distinct from each other — they are all "unset" synonyms; an assertion comparing two of them locks nothing. See "Distinct Test Values rule" in [inner-loop-guide.md](references/inner-loop-guide.md).
5. The inner loop is disposable — regenerate it after architectural changes. Outer-loop tests are durable — touch them only when intent changes.
6. **When unsure, ask first.** If you are about to write an assertion and you are *guessing* the expected behavior rather than knowing it, OR the behavior has 2+ defensible interpretations, OR a newly written test fails and you cannot tell whether the test or the production code is wrong — stop and surface the question to the user. Tools (mysql MCP / grep / git blame) sharpen the question; they do not replace asking.
