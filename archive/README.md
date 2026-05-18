# archive/

Snapshots of upstream Superpowers content we've replaced with our own versions.

**NOT loaded as skills.** Superpowers' skill discovery only walks `skills/` —
anything in this directory is inert reference material.

Kept on disk (not relied on via `git log`) so we can `Read` it quickly when
iterating on the replacements.

## upstream-skills/test-driven-development/

Original SP `test-driven-development` skill (single-loop Red-Green-Refactor +
Iron Law + rationalizations table + `testing-anti-patterns.md` sibling).

Replaced 2026-05-18 by our dual-loop TDD-SKILL (outer characterization +
inner adversarial, unit = use case, Testcontainers over mocked middleware,
`Contract:` block + business-language naming as executable spec).

Captured from upstream tag at commit `f2cbfbe` (Release v5.1.0).
