# Outer Loop — Greenfield Path: PRD → Acceptance Tests

## When to use

Only when **all** of the following hold:
- The feature is genuinely new (not extending existing logic)
- A written PRD or design doc with clear acceptance criteria exists
- The feature is high-stakes: payments, auth, compliance, external API contract, or a multi-team interface

For everything else — most modification work, extensions, bug fixes — use the [modification path](outer-loop-modification.md).

## Why this path is narrower than classical TDD

Classical TDD assumes every unit of code starts from a clear spec. In practice, most engineering work is iteration, not specification. Forcing "PRD → Gherkin → test" onto small or vague changes produces ceremonial tests that mirror the implementation rather than constraining it. The greenfield path keeps the discipline where it actually pays back: contract-heavy new features where a regression would be visible to users or partners.

## Flow

```
PRD → Extract scenarios → Convert to Gherkin → Test code → PM/QA review
```

## Step 1: Extract Scenarios

Prompt:
```
Analyze the following PRD and extract user scenarios and acceptance criteria.

PRD:
---
{PRD_CONTENT}
---

Output:
1. Core scenarios (by priority)
2. Acceptance criteria for each scenario
3. Edge cases and exception flows
4. Implicit business rules
```

## Step 2: Convert to Gherkin

Prompt:
```
Convert the following scenarios into Gherkin format.

Scenarios:
---
{SCENARIO}
---

Requirements: Given-When-Then structure; use Scenario Outline for data-driven cases.
```

## Step 3: Generate Test Code

Prompt:
```
Generate test code based on the following Gherkin.

Feature:
---
{FEATURE}
---

Use the {framework} framework, applying the Page Object pattern.
```

## Step 4: PM/QA Review

Checks:
- Are all PRD acceptance criteria covered?
- Do the expected results align with business intent?
- Are edge cases and exception flows included?

## Failure mode: "I have a PRD but it's vague"

If you cannot extract **at least three concrete acceptance criteria** from the PRD without inventing them yourself, the spec is not ready. Two options:

1. **Push back to product** to sharpen the PRD before writing tests.
2. **Drop to the modification path**: ship a thin first version, then characterize and extend.

Do not proceed with PRD → Gherkin against a vague spec — the resulting tests will encode your invented assumptions, not the contract, and you will end up rewriting them as the real spec emerges.

## FAQ

**Q: How fine-grained should outer-loop tests be?**
A: Focus on "user-perceivable feature units"; ignore internal implementation.
