# Architecture — how a consumer loads this skill

This skill is **standalone and project-agnostic**. A consumer skill (e.g. `leader`, `butler`) loads it on demand when a `type:story` / `type:epic` issue is processed.

## TOC

- [Layout](#layout)
- [How a consumer references this skill](#how-a-consumer-references-this-skill)
- [What this skill does NOT include](#what-this-skill-does-not-include)
- [Design rationale](#design-rationale)

## Layout

```
.claude/skills/dev-workflow/
├── SKILL.md                       Main entry — the 6-stage flow
├── reference/
│   ├── ac-perspectives.md         6 viewpoint AC checklist
│   ├── titling-convention.md      [code] / [code-Sn] prefix
│   └── architecture.md            (this file)
├── README.md                      Clone-and-use instructions
└── LICENSE
```

## How a consumer references this skill

A consumer skill (e.g. `leader` or `butler`) invokes:

```
Skill(skill="dev-workflow")
```

when the user / system encounters a `type:story` issue that needs the 6-stage flow.

The consumer may also load specific reference files inline (e.g. `reference/ac-perspectives.md`) when writing AC. References are loaded on demand — depth = 1, no nested reference traversal.

## What this skill does NOT include

The following are **out of scope** and live in the consumer:

- **Per-user concepts** (family members, individual operators, etc.) — consumer responsibility
- **Voice integration / realtime daemons** — consumer responsibility
- **Notifications (Pushover, Slack, Discord, ...)** — consumer responsibility
- **Repo-specific PAT / secrets** — loaded by the consumer's env / shell, never referenced here
- **Fast-lane / trivial-change types** (e.g. `type:dev`) — consumer declares them; this skill only knows `type:story` and `type:epic`
- **Bot comment markers** (e.g. `<!-- leader-bot -->`, `<!-- butler-bot -->`) — consumer responsibility
- **Multi-project context** (multiple code_dir / push_remote pairs) — consumer responsibility

## Design rationale

- **Standalone**: no placeholders, no values.yaml, no generator step. `git clone` and the skill works.
- **Runtime acquisition**: the consumer resolves repo slug / push branch / project root via `gh` / `git remote` / `git rev-parse`.
- **Depth-1 references**: every `reference/*.md` is loaded directly from SKILL.md, never from another reference file.
- **Loaded on demand**: a session that does not touch a `type:story` issue never reads this skill at all.
