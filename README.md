# dev-leader-plugin

A Claude Code **plugin** that bundles three project-agnostic skills used together for development work:

| Skill | Trigger | What it does |
|-------|---------|--------------|
| **leader** | "hi leader" / "hey leader" | Software development lead persona — runs a standup over open `type:story` / `type:epic` issues, walks pending retrospective proposals, recommends the next task. Does NOT write code; delegates to background agents. |
| **dev-workflow** | invoked by `leader` (or any consumer skill) for `type:story` / `type:epic` issues | Defines the 6-stage flow (file → refinement → AC gate → user approval → automated implementation → QA → close), AC-writing conventions, brainstorming template, commit/push conventions. |
| **dream** | manual: "dream" / "dreaming" / "consolidate memory" | Memory consolidation + daily log update + skill self-improvement + skill health check. 7-phase background-friendly flow. **Manual invocation only** — no sibling skill in this plugin auto-dispatches dream. |

Everything is **clone-and-use**: no template generator, no `values.yaml`, no placeholder substitution. Paths are repo-relative; the GitHub repo / push branch / project root are resolved at runtime via `gh` and `git`.

## Install

### Option A: Submodule (recommended for shared repos)

```bash
git submodule add https://github.com/tkcmada/dev-leader-plugin.git .claude/plugins/dev-leader-plugin
git submodule update --init --recursive
```

### Option B: Plain clone

```bash
mkdir -p .claude/plugins
git clone https://github.com/tkcmada/dev-leader-plugin.git .claude/plugins/dev-leader-plugin
```

After installing, **restart Claude Code** so the plugin loader picks up the three skills.

## Verify

After restart, the three skills should appear in the available-skills list:

- `leader`
- `dev-workflow`
- `dream`

In a Claude Code session, greet with **"hi leader"** to start a standup. Greet with **"dream"** when you want a memory consolidation pass.

## Cross-skill invocation

Inside this plugin's namespace, sibling skills are reachable by their bare names:

```text
Skill(skill="dev-workflow")    # leader loads this when a type:story issue is being processed
Skill(skill="dream")           # operator invokes this manually
```

A consumer skill living outside this plugin (e.g. a household-assistant `butler` skill, or a per-team dev wrapper) can invoke `leader` the same way:

```text
Skill(skill="leader")          # hand off a development request to the leader persona
```

## What this plugin assumes about your project

| Path | Purpose | Required? |
|------|---------|-----------|
| `memory/dev/daily/` | leader's daily notes | leader creates it on first run |
| `memory/dev/decisions/` | dev ADRs | leader creates it on first run |
| `memory/dev/notes/` | living reference notes | leader creates it on first run |
| `memory/dev/retrospectives/` | session retrospectives | leader creates it on first run |
| `memory/auto/` | dream's auto-memory index (`MEMORY.md` + topic files) | dream aborts if missing — create it before running dream |
| `memory/users/<actor>/daily/` | per-actor daily logs (optional) | only used if you split work per actor |
| `cache/dream_last_at` | dream invocation timestamp | dream creates it on first run |
| GitHub issues with `type:story` / `type:epic` / `type:dev` labels | the ticket vocabulary | the dev-workflow skill expects them |

The plugin is **actor-agnostic and project-agnostic** by design. Per-actor concepts (per-user wake words, per-user voice integrations, per-family-member memory) belong in a consumer-side wrapping skill that lives outside this plugin.

## Dream invocation

Dream is **manual only** inside this plugin. There is no 24-hour cache check, no auto-dispatch from `leader`, no scheduler. If you want scheduled runs (once-per-day consolidation, end-of-session goodbye trigger, etc.), wire that up at the consumer / wrapping-skill / cron layer — and keep the trigger logic outside this plugin so the manual `dream` invocation always behaves the same way.

To run dream manually, just say one of:

- `dream`
- `dreaming`
- `consolidate memory`
- `auto dream`

By default dream runs in **dry-run** mode (no Edit, no Write, no commit). To actually apply changes, say `dream apply` or pass `mode: apply`.

## Repository layout

```
dev-leader-plugin/
├── .claude-plugin/
│   └── plugin.json              Plugin metadata (name, version, skills list)
├── skills/
│   ├── leader/
│   │   ├── SKILL.md             Main entry: persona + trigger table + flow outline
│   │   ├── README.md            Skill overview
│   │   └── reference/           architecture.md / dev-workflow.md / dream.md / outcome.md
│   ├── dev-workflow/
│   │   ├── SKILL.md             Main entry: single-gate 6-stage flow
│   │   ├── reference/           six-stage-flow.md / self-improvement.md
│   │   └── templates/           refinement-drafts.md
│   └── dream/
│       ├── SKILL.md             Main entry: 7-phase flow
│       └── reference/           phases.md / outcome.md
├── docs/
│   └── split-skill-pattern.md   Split-skill pattern doc (used by the leader skill split demo)
├── README.md                    (this file)
├── CHANGELOG.md
├── LICENSE                      MIT
```

## Split-skill pattern — main SKILL.md + reference/*.md

Every skill in this plugin keeps a **slim main `SKILL.md`** (persona / trigger table / flow outline) and pushes step-by-step detail into on-demand `reference/*.md` (and, for `dev-workflow`, `templates/*.md`) files. The main entry links to each reference via a `References` table, so the model loads detail only when it actually enters that stage. This keeps the always-loaded context small while preserving the full procedure.

- `leader` → `reference/{architecture,dev-workflow,dream,outcome}.md`
- `dev-workflow` → `reference/{six-stage-flow,self-improvement}.md` + `templates/refinement-drafts.md`
- `dream` → `reference/{phases,outcome}.md`

The trigger / flow outline / cross-skill links in each `SKILL.md` are the externally observable contract — splitting only relocates detail, never changes behavior.

### How to apply the same split to your skill

A consumer that wants to split their own skill can follow the recipe in [`docs/split-skill-pattern.md`](docs/split-skill-pattern.md):

1. Inventory the current `SKILL.md` by section.
2. Classify each section as `keep` (persona / trigger / flow outline) or `move` (step-by-step / format / side-flow).
3. Cluster `move` sections into 3–5 cohesive reference files (one topic per file).
4. Add YAML `name:` + `purpose:` frontmatter and a `## TOC` to each reference.
5. Slim the main entry; replace each moved section with a 1–3 line summary + reference link.
6. Add a `References` table to the main entry.
7. Verify functional parity — externally observable behavior must be unchanged.

### Relation to `dream` Skill Health Check

The `dream` skill's **Skill Health Check** phase can auto-propose splits for `SKILL.md` files that exceed the recommended length. Each proposal surfaces as an `R-SH-N` retrospective item, walked through during the next standup for accept / reject / defer. Use `docs/split-skill-pattern.md` as the execution playbook when accepting one.

## Secret / PII safety

This repo is public. A committed pre-commit hook (`hooks/pre-commit`) rejects any commit
that contains secrets (PAT / AWS / Slack / PEM / Bearer), maintainer family PII
(whole-word romaji names + kanji + `user:<member>` labels — substrings like `emit` /
`emission` do not fire), absolute home paths, or other private tokens. Enable it once after
cloning with:

```sh
git config core.hooksPath hooks
```

The matched value is never printed (only file + line + rule name).

## License

[MIT](LICENSE).
