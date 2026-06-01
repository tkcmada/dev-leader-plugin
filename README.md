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
│   │   ├── SKILL.md             Main entry: persona + standup
│   │   └── reference/           standup.md / memory-format.md / github-issues.md / handoff-from-butler.md
│   ├── dev-workflow/
│   │   ├── SKILL.md             Main entry: 6-stage flow
│   │   └── reference/           ac-perspectives.md / titling-convention.md / architecture.md
│   └── dream/
│       ├── SKILL.md             Main entry: 7-phase flow
│       └── reference/           phases.md / signal-sources.md / outcome.md
├── README.md                    (this file)
├── CHANGELOG.md
├── LICENSE                      MIT
```

## License

[MIT](LICENSE).
