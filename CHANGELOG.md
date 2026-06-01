# Changelog

All notable changes to **dev-leader-plugin** are recorded here.

## 0.1.0 (2026-06-02) — Initial release

- Bundle three project-agnostic skills as one Claude Code plugin:
  - `leader` — software development lead persona (greeting "hi leader", standup, ticket orchestration)
  - `dev-workflow` — 6-stage flow for `type:story` / `type:epic` GitHub issues (file → refinement → AC gate → user approval → automated implementation → QA → close)
  - `dream` — memory consolidation + daily log update + skill self-improvement + skill health check (7-phase, manual invocation only)
- Cross-skill invocation supported via `Skill(skill="dev-workflow")` and `Skill(skill="dream")` from `leader` (and from any consumer skill).
- No sibling skill auto-dispatches `dream`; operators invoke it manually.
- MIT licensed.
