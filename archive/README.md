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
