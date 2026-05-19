# Inner Loop: Adversarial Tests

> **Before starting**: confirm the code under test is worth testing at all. If the function is a pure I/O wrapper (SDK call, HTTP client) with no branching or transformation logic, mocks will dwarf the code and the tests will verify the mocks rather than behavior — say so to the user and stop. If the code touches a middleware client, the test will be a behavior-scoped test backed by a real Testcontainers container — MySQL via `AbstractMySqlIT` (see `testcontainers-mysql-guide.md`); same pattern for Redis / broker / other middleware (build the `AbstractXxxIT` base if missing). **Never mock a middleware client to simulate that middleware's semantics.** See SKILL.md → "Project test architecture" and "Decision: is a test worth writing?" for the full decision flow.

## Core Distinction

❌ Sycophantic (wrong):
```
// Seeing the implementation return user.name, just write
expect(getDisplayName({name: 'Alice'})).toBe('Alice');
```

✅ Adversarial (correct):
```
// Think: what if name is empty / null / undefined?
expect(getDisplayName({name: null})).toBe('Anonymous');
expect(getDisplayName({})).toBe('Anonymous');
```

## Outer-Loop Compensation (must be performed before entering the inner loop)

Inner-loop tests focus on edge-case attacks, but if outer-loop tests are missing, the basic business logic has no test protection at all. Therefore, **before generating inner-loop tests, you must always run the outer-loop compensation check**.

### Detection Flow

```
1. Look for an existing test file in the test directory corresponding to the class under test.
2. If one exists, scan the existing test methods:
   - Do they cover the core happy path of the method under test (normal input → correct output)?
   - Do they cover the main business branches (each business path of if/else)?
3. Decision:
   - Core happy path and main business branches both covered → skip compensation, go straight to inner loop
   - Coverage missing → run outer-loop compensation
```

### Compensation Content

The outer-loop compensation tests live in the same test class, separated from inner-loop tests using `@Nested`:

```java
@Nested
@DisplayName("Business logic verification")   // ← outer-loop compensation
class BusinessLogicTests {
    @Test void normalInput_shouldReturnCorrectResult() { ... }
    @Test void branchAConditionMet_shouldTakePathA() { ... }
    @Test void branchBConditionMet_shouldTakePathB() { ... }
}

@Nested
@DisplayName("Adversarial edge tests")  // ← inner loop
class AdversarialTests {
    @Test void nullInput_shouldHandleDefensively() { ... }
    @Test void boundaryValues_shouldBeHandledCorrectly() { ... }
}
```

### Principles for Generating Compensation Cases

| Principle | Description |
|-----------|-------------|
| Start from method signature and business semantics | Ignore implementation details; focus on "what should this method do" |
| One case per business path | Read the code to identify if/switch/strategy branches; at least one happy-path case per path |
| Assert outputs, not process | Assert return values and key side effects, not internal call ordering |
| Follow the value-source distinction rule | Test data in compensation cases must also satisfy Distinct Test Values |

## Adversarial Prompt

```
You are a senior QA engineer whose KPI is the number of bugs found.

## Business context
{outer-loop tests}

## Code under test
{implementation code}

## Task
Generate tests. The goal is to **attack** the code, not validate it.

### Attack vectors that must be covered
1. Empty values: null, undefined, empty string, empty array
2. Boundaries: 0, -1, MAX_INT, very long strings
3. Types: pass in wrong types
4. State: uninitialized, already destroyed, concurrent calls
5. Format: special characters, Unicode, injection patterns
6. Data sources: does the code read from the correct variable/field? (see "Distinct Test Values" rule below)

### Forbidden
- Validating only the happy path
- Paraphrasing the implementation logic
- Assuming inputs are always valid
- Copying mock expected values directly from variables used in the implementation (see "Distinct Test Values" rule below)
```

## Review Checklist

Keep:
- Tests that uncover real bugs
- Tests covering important edge cases

Discard:
- Tests that merely paraphrase the implementation
- Tests that test private details
- Tests that are overly fragile

## Distinct Test Values Rule

**Core problem**: when AI reverse-engineers tests from the implementation, it tends to set mock expected values to the same values as variables used in the implementation. If the code reads from the wrong data source (uses field A instead of field B), but A and B happen to be equal in the test, the test will never catch the bug.

**Rule: every data source the method under test can reach must be assigned mutually distinct values in the test.**

Data sources include:
- Different fields in method parameters
- Configuration values injected into the object under test (`@Value`)
- Object fields read from the database/cache
- Data returned from external services

### Negative example (bug escapes)

```java
// Bug in the code under test: should use task.getClientId(), but uses bybitClientId instead
platformService.listProjectTemplateLanguages(templateUid, Integer.valueOf(bybitClientId));

// ❌ Test: bybitClientId and task.clientId happen to share the same value, indistinguishable
ReflectionTestUtils.setField(service, "bybitClientId", "88");
Task task = new Task();  // clientId not set, or also 88
when(platformService.listProjectTemplateLanguages(eq("tpl-001"), eq(88)))
    .thenReturn(languages);  // matches regardless of which variable the code reads → bug escapes
```

### Positive example (bug exposed)

```java
// ✅ Test: a distinct value per data source
private static final String BYBIT_CLIENT_ID = "88";      // @Value config
private static final Integer TASK_CLIENT_ID = 999;        // task's own field

ReflectionTestUtils.setField(service, "bybitClientId", BYBIT_CLIENT_ID);
Task task = new Task();
task.setClientId(TASK_CLIENT_ID);  // intentionally different from bybitClientId

// mock expectation uses the task's clientId (the source the contract requires)
when(platformService.listProjectTemplateLanguages(eq("tpl-001"), eq(TASK_CLIENT_ID)))
    .thenReturn(languages);
// If the code under test uses bybitClientId(88), the mock won't match → test fails → bug exposed
```

### Execution Checklist

When generating tests, complete the following checks:

| Step | Check |
|------|-------|
| 1 | List every data source the method under test can reach (parameters, fields, injected values, query results) |
| 2 | Identify pairs of data sources that share the same type but different semantics (e.g. two `Integer clientId`s) |
| 3 | Assign each pair distinct literal values |
| 4 | Choose mock expected values based on the **contract's intent**, not by copying variables from the implementation |

### Patterns to watch for in this project

Two real violations and one positive reference. Inspect the cited files before writing similar tests.

**Violation 1 — One literal reused across semantically-distinct fields.** `IdexxProjectCreatedHookTest` (lines ~565–586) sets `task.id=1`, `task.clientId=1`, `config.clientId=1`, `language.taskId=1` — four distinct data sources, all the literal `1`. If production code reads `task.id` where it should read `task.clientId`, or `language.taskId` where it should read `task.clientId`, every assertion still passes. **Fix**: distinct primes per field (e.g. `task.id=11`, `task.clientId=23`, `language.taskId=37`).

**Violation 2 — Test imports the production formula it's supposed to pin.** `AssignResourceHandlerTest.shouldCallFallbackWithCorrectDefaultProjectId()` computes `expectedDefault = ConnectorProject.getDefaultProjectName(SOURCE, CLIENT_ID)` (line 180) then asserts production code passes that value. If the formula breaks, expected breaks the same way — test stays green. **Fix**: hard-code the expected literal (`"Default-LOKALISE-10"`) so formula drift fails the test.

**Positive reference — Don't conflate namespace isolation with the Distinct Values rule.** `ConfigServiceImplAutoBindTest` does both correctly side by side: the `taskIdSeed = 2_000_000` line (`// distinct namespace from sibling ITs`) prevents PK clashes across test classes — that's parallel-test hygiene, **not** Distinct Values. The actual rule lives in the constants block above (`WORKFLOW_OKX_UID` vs `BINDING_ERP_RESOURCE_ID` vs `SEED_SMARTLING_NAME` — distinct values per data source). They're separate concerns; the comments in this file make the distinction crisply, copy that style.

## When to Regenerate

- Architectural refactor (function signature changes)
- New functionality added (after the outer loop expands)
- A bug surfaces in the test environment that wasn't covered
