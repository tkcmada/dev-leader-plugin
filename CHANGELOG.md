# Changelog

All notable changes to **dev-leader-plugin** are recorded here.

## 0.2.0 (2026-06-02) — leader split demonstration

- Split `skills/leader/SKILL.md` from a single 275-line file into a slim 109-line main entry plus seven on-demand reference files (60% reduction).
- New references: `reference/orchestration.md` (persona + agent routing + memory writes), `reference/wrap-up.md` (wrap-up + good-bye + retrospective), `reference/task-list.md` (task list display + priority heuristics + recommendation format).
- Added YAML `name:` + `purpose:` frontmatter to all reference files (existing and new) for scannability.
- New `docs/split-skill-pattern.md` documenting the split-skill convention (when to split, main / reference responsibilities, frontmatter / TOC rules, before-after example, 7-step recipe).
- Updated `README.md` with a "Split demonstration" section (before-after line counts, link to the docs).
- Functional parity preserved: trigger phrases, cross-skill invocations (`Skill(skill="dev-workflow")`, `Skill(skill="dream")`), file paths, and output formats are unchanged.

## 0.1.0 (2026-06-02) — Initial release

- Bundle three project-agnostic skills as one Claude Code plugin:
  - `leader` — software development lead persona (greeting "hi leader", standup, ticket orchestration)
  - `dev-workflow` — 6-stage flow for `type:story` / `type:epic` GitHub issues (file → refinement → AC gate → user approval → automated implementation → QA → close)
  - `dream` — memory consolidation + daily log update + skill self-improvement + skill health check (7-phase, manual invocation only)
- Cross-skill invocation supported via `Skill(skill="dev-workflow")` and `Skill(skill="dream")` from `leader` (and from any consumer skill).
- No sibling skill auto-dispatches `dream`; operators invoke it manually.
- MIT licensed.
