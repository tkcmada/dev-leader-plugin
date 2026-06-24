# dev-leader-skill

Clone-and-use **leader** skill for any Claude Code project: dev standup, dev-workflow 6-stage flow, dream memory-consolidation. No generator, no `values.yaml`, no placeholder substitution. Just clone or submodule and the skill is live.

## What it is

A single Claude Code skill (`name: leader`) that bundles:

- **leader** persona — dev standup, retrospectives, agent orchestration, GitHub issue management (`SKILL.md`).
- **dev-workflow** — 6-stage flow for `type:story` / `type:epic` (`reference/dev-workflow.md`).
- **dream** — 7-Phase memory-consolidation + skill self-improvement (`reference/dream.md`).
- **outcome** — 3-layer record/improvement loop documentation (`reference/outcome.md`).

All paths are **repo-relative**. The GitHub repo and push branch are read at runtime via `gh` and `git remote`. The skill is **user-agnostic** — any per-user concept (family members, individual operators, etc.) belongs to the consumer's own wrapping skill.

## Install — submodule (recommended)

```bash
cd <your-project>
git submodule add https://github.com/tkcmada/dev-leader-skill.git .claude/skills/dev-leader-skill
git submodule update --init --recursive
```

## Install — plain clone (vendor)

```bash
cd <your-project>
git clone https://github.com/tkcmada/dev-leader-skill.git /tmp/dls
rm -rf /tmp/dls/.git
mkdir -p .claude/skills
cp -r /tmp/dls .claude/skills/dev-leader-skill
```

Either way, Claude Code's harness picks up `.claude/skills/dev-leader-skill/SKILL.md` automatically and the skill becomes available as **leader** (trigger phrase: "hi leader" / "hey leader").

## Requirements

- `gh` CLI authenticated against the project's GitHub repo (the leader uses `gh issue list` / `gh issue create` / etc.).
- A `memory/dev/` tree (will be auto-created at first standup).
- For dream: optionally a `memory/auto/`, `memory/shared/`, `memory/users/` tree if you want the memory consolidation phases to do useful work. dream scans for these and skips gracefully if missing.

## Repository layout

```
SKILL.md                Main entry (loaded by Claude Code; defines the leader persona)
reference/
  dev-workflow.md       6-stage flow definition
  dream.md              7-Phase memory consolidation
  outcome.md            3-layer record/improvement loop
  architecture.md       leader / dev-workflow / dream relationship
scripts/                Optional helper scripts
README.md               This file
LICENSE
```

## What this skill does NOT include

By design, this skill does NOT provide:

- per-user concepts (consumers wrap leader in a per-user skill if needed)
- notifications (Pushover/Slack/Discord — add via your consumer-side helper script)
- voice / chat realtime daemons (orchestrate separately)
- a fast-lane `type:dev` flow (declare in your consumer skill if you want one)
- a `values.yaml` / `generate.py` toolchain (those were removed)

## Customization

If you need project-specific behavior on top of the leader persona — repo conventions, fast-lane labels, notification helpers, family-aware semantics, etc. — wrap this skill in a **second skill** in your project and let your consumer skill be the trigger:

```
.claude/skills/
  dev-leader-skill/         ← this skill (clone-and-use, do not edit)
  butler/                   ← your consumer skill that wraps leader for family/dev integration
```

Edits to this skill should be sent upstream as PRs against `tkcmada/dev-leader-skill`; consumer-specific tweaks stay in the consumer's wrapping skill.

## License

Not specified. Treat as personal/internal until a license is added.
