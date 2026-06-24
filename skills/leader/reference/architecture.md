# architecture — leader / dev-workflow / dream relationship

This skill bundles three previously-separate skills into a single, **clone-and-use** distribution.

## File layout

```
.claude/skills/dev-leader-skill/
├── SKILL.md                  Main entry — the leader persona (responds to "hi leader")
├── reference/
│   ├── dev-workflow.md       6-stage flow for type:story / type:epic implementation
│   ├── dream.md              7-Phase memory consolidation + skill self-improvement
│   ├── outcome.md            3-layer record/improvement loop (issue → retro → dream)
│   └── architecture.md       (this file)
├── scripts/                  Optional helper scripts (e.g. dream cron wrappers)
├── README.md                 Clone-and-use instructions
└── LICENSE
```

The skill is loaded by name `leader` (see SKILL.md frontmatter). All references are loaded on demand by the leader at the appropriate phase — `reference/dev-workflow.md` when the user triggers a `type:story`, `reference/dream.md` when the consumer dispatches dream, etc.

## Relationship to the consumer project

The consumer project (Butler, tanecon, tottori, ...) does one of two things:

### Option A: Submodule

```bash
git submodule add https://github.com/tkcmada/dev-leader-skill.git .claude/skills/dev-leader-skill
git submodule update --init --recursive
```

Pros: tracked version, easy to `git pull` upstream improvements.
Cons: requires `git submodule update --init` on fresh clone.

### Option B: Plain clone (vendor)

```bash
git clone https://github.com/tkcmada/dev-leader-skill.git /tmp/dls && \
  rm -rf /tmp/dls/.git && \
  cp -r /tmp/dls .claude/skills/dev-leader-skill
```

Pros: no submodule machinery; works in any environment.
Cons: manual `git pull` discipline to stay in sync.

Either way the result is a `.claude/skills/dev-leader-skill/SKILL.md` that the consumer's harness picks up.

## What goes in the consumer project (NOT in this skill)

This skill is intentionally **project-agnostic and user-agnostic**. The following stays in the consumer's own skills:

- **Per-user concepts** (family members, individual operators, etc.) — handled by a consumer-side wrapping skill (e.g. Butler's `butler` skill).
- **Voice integration / chat realtime daemons** — orchestrated separately.
- **Notifications (Pushover, Slack, Discord, ...)** — wrapped in a consumer-side helper script; this skill does not call them.
- **Repo-specific PAT / secrets** — loaded by the consumer's env / shell, never referenced here.
- **Fast-lane / trivial-change types** (e.g. Butler's `type:dev`) — the consumer's wrapping skill declares them; this skill only knows `type:story` and `type:epic`.

## Why the placeholder ZERO design

Prior versions used Jinja2 placeholders + a `generate.py` + per-project `values.yaml`. That worked but had three friction points:

1. **Indirection**: reading the rendered SKILL.md and the template at the same time to understand actual behavior.
2. **Sync drift**: `additions/*.md` had to be re-merged after every regenerate, easy to forget.
3. **Clone-and-use friction**: a new project needed values.yaml authoring + render step before it could even try the skill.

The current design replaces all of this with:

1. **Hard-coded relative paths** (everything under the project root).
2. **Runtime acquisition** of the repo slug / push branch (`gh` / `git remote` / `git rev-parse`).
3. **Auto-discovery** of dream signal sources (scan known paths, skip missing).
4. **Bundled** dev-workflow + dream in one skill.
5. **No additions/ mechanism** — consumer-specific extensions belong in the consumer's wrapping skill, not in this skill's tree.

The result is a single `git clone` → working skill.
