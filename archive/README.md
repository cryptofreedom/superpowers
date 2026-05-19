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

## upstream-skills/{executing-plans,writing-plans,using-git-worktrees,dispatching-parallel-agents,subagent-driven-development,requesting-code-review,finishing-a-development-branch}/

Seven upstream SP skills removed 2026-05-18 because Claude Code + Opus 4.7
already covers them natively — they added rhetorical pressure to do things
the model was going to do anyway, and burned bootstrap context.

- `executing-plans` / `writing-plans` — superseded by Claude Code's plan
  mode (`ExitPlanMode`) which writes a real plan file and gates execution
  on user approval.
- `using-git-worktrees` — superseded by Claude Code's native
  `EnterWorktree` / `ExitWorktree` tools and the Agent tool's
  `isolation: "worktree"` parameter.
- `dispatching-parallel-agents` — superseded by Claude Code's default
  parallel-call behavior ("make multiple independent tool calls in a
  single message").
- `subagent-driven-development` — overweight execution pattern;
  Claude Code's Agent tool handles the same dispatch on demand.
- `requesting-code-review` — superseded by Claude Code's `/ultrareview`
  slash command (multi-agent cloud review).
- `finishing-a-development-branch` — superseded by Claude Code's
  built-in git + `gh` CLI fluency and the model's own merge/PR judgment.

Captured from upstream tag at commit `f2cbfbe` (Release v5.1.0). Kept on
disk for reference if any specific rhetorical device or sub-doc turns out
to be worth grafting into the survivors later.

Concurrent cleanup applied to surviving files:
- `skills/brainstorming/SKILL.md` — terminal state changed from "invoke
  writing-plans" to "enter Claude Code's plan mode (`ExitPlanMode`)".
- `skills/using-superpowers/references/codex-tools.md` — two dangling
  cross-references to deleted skills generalized.
- `.codex-plugin/plugin.json` — keywords + longDescription updated.

## upstream-harness-support/ — multi-harness assets removed 2026-05-18

Repo fully committed to Claude Code only. Everything that served Codex /
Cursor / OpenCode / Gemini / Copilot CLI was removed (with content snapshots
kept here in case a specific pattern is worth grafting later):

- `codex-plugin/` — Codex CLI plugin manifest
- `cursor-plugin/` — Cursor plugin manifest
- `opencode/` — OpenCode plugin (INSTALL.md + JS plugin)
- `gemini-extension.json` + `GEMINI.md` — Gemini CLI bootstrap
- `hooks-cursor.json` — Cursor SessionStart hook config
- `codex-tools.md` / `copilot-tools.md` / `gemini-tools.md` — per-harness
  Skill→tool mapping references for the bootstrap
- `sync-to-codex-plugin.sh` — script that pushed to OpenAI's Codex plugins
  repo (we never publish there)

Also deleted (no content backup — git history is enough):
- `.github/` (FUNDING.yml + ISSUE_TEMPLATE/ + PULL_REQUEST_TEMPLATE.md) —
  upstream open-source community flow; this fork doesn't accept PRs
- `CODE_OF_CONDUCT.md` — community guideline
- `RELEASE-NOTES.md` (1180 lines) — upstream's version history, not ours
- `docs/` (17 files) — upstream's historical spec/plan/design archive
- `tests/` (52 files) — upstream test scripts referencing deleted skills /
  non-CC harnesses; all broken after the round-1 cleanup anyway
- `README.md` — upstream's multi-harness install marketing
- `.version-bump.json` + `scripts/bump-version.sh` — drove version sync
  across 6 manifests; only 2 manifests now (plugin.json + marketplace.json)
- `package.json` — only field was `"main": ".opencode/plugins/superpowers.js"`
  pointing at the deleted opencode plugin
- `AGENTS.md` (symlink → `CLAUDE.md`) — read by OpenAI Codex, not CC

## upstream-originals/ — files rewritten in place

Originals captured before rewrite so the new versions are diffable:

- `CLAUDE.md` — original 107-line upstream contributor guidelines with "94%
  PR rejection rate" rhetoric (CC auto-loads project CLAUDE.md every session
  → cost was high). Rewritten to a 4-paragraph personal-fork notice.
- `using-superpowers-SKILL.md` — original 118-line bootstrap that branched
  to Copilot CLI / Gemini CLI / other-environment sections, mentioned
  nonexistent example skills (`frontend-design`, `mcp-builder`), used
  deprecated tool name `TodoWrite`, had a 12-row Red Flags table with
  several redundant entries. Rewritten to ~80 lines, CC-only, real example
  skill names, `TaskCreate`, 8-row Red Flags, plus an `Agent` tool line
  filling the semantic gap left by the deleted dispatch/parallel skills.
- `session-start` — original 57-line bash script with 3-way platform
  detection (CURSOR_PLUGIN_ROOT / CLAUDE_PLUGIN_ROOT / COPILOT_CLI) and
  3 different JSON output formats, plus a legacy `~/.config/superpowers/skills`
  migration warning. Rewritten to ~30 lines, CC-only JSON format.
- `claude-plugin.json` + `marketplace.json` — original upstream author /
  description / keywords. Renamed plugin `superpowers` → `jason-superpowers`
  (which required `superpowers:foo` → `jason-superpowers:foo` updates in
  `skills/systematic-debugging/SKILL.md`, `skills/writing-skills/SKILL.md`,
  and `skills/writing-skills/testing-skills-with-subagents.md`).

## upstream-skills/{brainstorming,verification-before-completion}/ — removed 2026-05-18 (round 3)

Two more skills cut after determining they are net-negative for jason's
actual workflow on Claude Code + Opus 4.7:

- `brainstorming/` — Forced design-doc ceremony before any creative work.
  Redundant when the user self-judges complexity: simple work → direct
  implement, complex work → CC's plan mode. brainstorming layered `spec →
  plan → execute` instead of the cleaner `plan → execute`. Includes the
  visual-companion browser server (`scripts/start-server.sh`, frame
  template, etc.) — full directory archived.
- `verification-before-completion/` — Forced re-run of validation
  commands before claiming "done". Useful for users who get burned by
  false-done claims; for jason, false-done is rare and the discipline
  cost outweighed the benefit.

Concurrent bootstrap simplification:
- `skills/using-superpowers/SKILL.md` — graphviz lost the
  `"About to EnterPlanMode?" → "Already brainstormed?" → "Invoke
  brainstorming"` branch (3 nodes + 4 edges removed); Skill Priority
  section rewritten to drop the brainstorming example and point
  "Add a feature" work at TDD + CC plan mode instead.
- `skills/systematic-debugging/SKILL.md` — dropped the
  `verification-before-completion` line from Related skills.
- `skills/writing-skills/SKILL.md` — replaced `verification-before-completion`
  with `systematic-debugging` in the Discipline-Enforcing skill examples.
- `skills/writing-skills/render-graphs.js` — example command pointed at
  `../brainstorming` (now deleted); switched to `../using-superpowers`.
- `.claude-plugin/plugin.json` + `marketplace.json` — description and
  keywords no longer mention brainstorming / verification.

Surviving 5 skills: `receiving-code-review`, `systematic-debugging`,
`test-driven-development`, `using-superpowers`, `writing-skills`.

## upstream-skills/systematic-debugging/ — removed 2026-05-19 (round 4)

Plugin skill cut after evaluating its content against CC + Opus 4.7 baseline.
Most of the 4-phase methodology (read errors, reproduce, check git, find
working examples, form hypothesis) is already strongly internalized by Opus
4.7. The genuine bite was ~20 lines: Iron Law, single-hypothesis discipline,
the 3-failed-fixes architectural gate, and a compressed rationalizations
table. Those 20 lines were migrated to `eci_log_debugger` (user-level skill)
as Pattern 7, Pattern 8, a top-of-file Iron Law block, and a Pitfalls
rationalizations table.

The 3 supporting reference files were all evaluated and dropped without
migration:
- `root-cause-tracing.md` — JS/TS technique (`console.error` + `new Error().stack`).
  Java has full stack traces by default, and `eci_log_debugger` Pattern 6
  already encodes the "trace to a logging-guaranteed boundary" principle in
  ECI-specific terms (Feign interface method name).
- `defense-in-depth.md` — Actually a fix-hardening / validation-design pattern,
  not a debugging methodology. Different concern. Spring already has the
  validation idioms it advocates (`@Valid`, ControllerAdvice, custom validators).
  If anything, this content might belong in `java-backend-dev` later — not in
  the debug stack.
- `condition-based-waiting.md` — Jest/Vitest test-writing pattern
  (`waitFor` vs `setTimeout`). Java equivalent is Awaitility, well-known.
  TDD-SKILL's unit=use-case + Testcontainers transaction-commit semantics
  naturally avoids the multi-second sleep antipatterns this addresses.

Concurrent cleanup:
- `skills/using-superpowers/SKILL.md` — Skill Priority section rewritten
  to route "Fix this bug" at log/trace skills first (instead of
  systematic-debugging); Skill Types Rigid list dropped systematic-debugging.
- `skills/writing-skills/SKILL.md` — Discipline-Enforcing examples reduced
  to `TDD, designing-before-coding` (systematic-debugging removed).
- `eci_log_debugger/SKILL.md` (user-level) — Iron Law inserted after frontmatter;
  Patterns 7 & 8 appended to Reasoning Patterns; rationalizations table inserted
  at top of Pitfalls; frontmatter description extended to mention general debugging
  discipline so the skill auto-routes for any debugging task (not just log keywords).

Surviving 4 skills: `receiving-code-review`, `test-driven-development`,
`using-superpowers`, `writing-skills`.

## user-level-migrated/TDD-SKILL/ — removed 2026-05-19 (round 5)

The user-level `~/.claude/skills/TDD-SKILL/` was the original source we cloned
into plugin's `skills/test-driven-development/` during round 0. After the SP
bite grafts (Iron Laws, Common Rationalizations, Red Flags, Verification
Checklist, Final Rule, MANDATORY-in-Picking-the-path), the plugin version
became a strict superset of TDD-SKILL — verified by `diff`:
- 4 references files: 100% byte-identical
- SKILL.md: plugin has 5 extra H2 sections, no content lost
- Only "deletion" in the migration was the frontmatter `name:` field and the
  table rows that got enhanced with inline MANDATORY suffix

Kept as `archive/user-level-migrated/TDD-SKILL/` for recovery if needed.
After this round, the plugin's `test-driven-development` is the single
authoritative TDD skill — no duplicate routing in Opus's skill list.

Concurrent integration: `skills/using-superpowers/SKILL.md` Skill Priority
section was rewritten into a fuller **Skill Map + Intent Routing** that
explicitly lists all 7 post-install skills (4 plugin + 3 user-level —
`eci_log_debugger`, `local-debug`, `java-backend-dev`) and maps user-intents
to skill chains. Skill Types section's Rigid list extended to mention
`eci_log_debugger`'s Iron Law + Patterns 7/8.

User-level skills retained (after this round): `eci_log_debugger`,
`local-debug`, `java-backend-dev` (3 user-level; TDD-SKILL removed as above).

## user-level slimming 2026-05-19 (round 6)

Two follow-up edits on user-level skills, no content backup (user owns these
and can git-track separately; the changes are slim-not-delete):

### `~/.claude/skills/local-debug/SKILL.md` — Phase 4 slimmed (~25 lines removed)

The original Phase 4 ("Debug Loop Protocol") carried a self-contained 8-step
methodology cycle (understand → hypothesize → change → restart → test →
analyze → verify → loop) plus 6 substeps. After round 4 moved general
debugging methodology into `eci_log_debugger` (Iron Law + Pattern 7 + Pattern 8),
local-debug's 8-step cycle became redundant — two skills competing to own
the "how to debug" narrative.

Phase 4 renamed to "**In-Loop Operations (project-specific)**" and reduced
to only the project-specific bits: reload strategy decision (Java-only vs
pom/config), reproduce + capture trace_id, MySQL MCP verification, report
format. Opens with explicit pointer: "For methodology, invoke
`eci_log_debugger`". One-skill-one-responsibility now holds.

### `~/.claude/skills/java-backend-dev/SKILL.md` — dead `TDD-SKILL` reference removed

The section "## TDD-SKILL (MANDATORY — invoke before AND after editing)"
(3 lines) pointed at a skill that no longer exists (TDD-SKILL was deleted
round 5; its content migrated to plugin's `test-driven-development`).

Section deleted in full. TDD invocation for Java work is now handled by
the bootstrap's Intent Routing table — every code-change chain has
`test-driven-development` in it. The "before AND after" framing was
already redundant with `test-driven-development`'s own dual-loop
structure (outer-loop = before; inner-loop = after).
