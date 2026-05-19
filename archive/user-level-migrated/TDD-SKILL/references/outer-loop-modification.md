# Outer Loop — Modification Path: Lock Behavior Before Changing It

## When to use

Default path for most day-to-day work:
- Bug fixes
- Adding a new branch / parameter / case to existing logic
- Refactoring with behavior preservation
- Extending an existing feature

If the change has a clear, written acceptance criterion you would be willing to send to QA, use the [greenfield path](outer-loop-greenfield.md) instead.

## Why characterization tests, not PRD → Gherkin

Modification work rarely comes with clean acceptance criteria — it comes with "make this faster", "fix the bug where X", "also handle Y". The risk in this work is **silent regression**, not contract violation. Characterization tests address that directly: pin the current observable behavior before touching the code, so any unintended change shows up as a red test instead of an incident in the test environment.

## Flow

```
1. Identify the call surface that will change (method, endpoint, class)
2. Check existing tests:
   - Cover the behavior you are about to modify? → keep them, run them
   - Don't cover it?                            → write 1–3 characterization tests
3. Run the suite — must be green before you touch the implementation
4. Make the change:
   - Tests that should still pass → still passing? good
   - Tests that should now fail   → fail with the expected new output? update assertion
   - Tests that turn red unexpectedly → STOP, that's a regression
5. Add new tests for new behavior introduced by the change
6. Run the inner loop on the modified code for adversarial coverage
```

## Writing characterization tests

The goal is **behavior pinning**, not specification. You are not asking "what *should* this do?" — you are asking "what *does* it do today, and which of those behaviors must I not accidentally break?"

```java
// Goal: pin current behavior of OrderService.calculateDiscount before refactor
@Test
void calculateDiscount_5ItemsOver100_currentlyReturns10Percent() {
    Order order = orderWith(5, new BigDecimal("120.00"));
    assertThat(service.calculateDiscount(order)).isEqualByComparingTo("12.00");
}

@Test
void calculateDiscount_emptyOrder_currentlyReturnsZero() {
    assertThat(service.calculateDiscount(emptyOrder())).isEqualByComparingTo("0.00");
}
```

Rules:
1. **Assert observed output, not intended output.** If the current behavior is weird, pin the weirdness. Refactoring should preserve it; fixing it is a separate intentional change with its own test update.
2. **Two to four tests is usually enough.** Cover the happy path and one or two important branches you are *not* changing. Do not aim for exhaustive coverage at this step — that is the inner loop's job after the change.
3. **No mocks of internal collaborators.** Characterization tests target observable behavior. If the surface is hard to test without mocking everything, that is a refactor signal, not a testing problem.
4. **Name the tests after the observation, not the spec** (e.g. `currentlyReturns10Percent`, not `shouldReturn10Percent`). Reminds future you that these tests describe *what is*, not *what should be*.
5. **If the surface is middleware-backed** (real SQL, Redis cache / lock, broker delivery), pin behavior with a real Testcontainers container — not by mocking the middleware client. See SKILL.md → "Project test architecture".

## Failure modes after the change

After modifying the code, every test ends up in one of four states. Each state tells you what to do:

| State | Meaning | Action |
|-------|---------|--------|
| Was green, still green | Behavior preserved | Keep test as-is |
| Was green, now red — expected | Behavior intentionally changed | Update assertion to new expected output |
| Was green, now red — unexpected | Silent regression | Stop, investigate, fix |
| New test for new behavior | New functionality | Add and assert against the spec |

Catching the third row is the whole point of this path. Without characterization tests in place, an unexpected regression is invisible until it reaches the test environment.

## When to skip this path entirely

- Throwaway scripts
- Pure formatting / rename-only changes no test could catch
- Generated code
- Trivial one-liners with obvious no-behavior-change semantics (e.g. extracting a constant)

If you are tempted to skip it for a "quick fix" in the code under test, that is exactly the case where you should not skip it.
