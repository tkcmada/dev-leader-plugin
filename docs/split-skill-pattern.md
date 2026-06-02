# Split skill pattern — main SKILL.md + reference/*.md

A reusable pattern for keeping skill main-entries short and discoverable while moving step-by-step detail into on-demand reference files. This document captures the conventions used by the `leader` skill in this plugin (v0.2.0) so that consumers can apply the same split to their own skills.

## When to split

Split a skill's `SKILL.md` into a main entry + `reference/*.md` when **any** of the following holds:

- `SKILL.md` exceeds **~250 lines** (Anthropic's authoring guidance recommends keeping the main entry well under 500 lines; ~150–200 lines is a comfortable target).
- The same large detail block (procedure, table, format spec) is consulted only occasionally, not on every invocation.
- The persona / trigger / top-level flow is buried under low-level reference material.
- A consumer skill needs to read **just one specific piece** (e.g. only the issue template) without re-reading the whole persona.

## Conventions

### Main entry (`SKILL.md`)

- **YAML frontmatter** is required: `name` + `description`. The description is what the harness routes on, so write it from the trigger-recognition angle.
- Keep **~150 lines or less** in the body.
- Sections to keep in the main entry:
  - Persona / role summary (1 short paragraph).
  - Trigger phrases / recognition table.
  - Top-level flow outline (numbered, one line per step, each step pointing at a reference where the detail lives).
  - "References" table (file → when to read).
- Sections to move out:
  - Step-by-step procedures (each "Standup step 1, step 2, ..." block).
  - Format specs (file templates, issue body templates, label vocabularies).
  - Side-flows triggered by less common phrases (wrap-up, retrospective, etc.).
  - Exhaustive routing / agent / orchestration tables.

### Reference files (`reference/<name>.md`)

- **YAML frontmatter**: `name: <ref-name>` + a one-line `purpose:` so a consumer can scan reference titles without opening each file.
- Start with a top-level `# Title` and a `## TOC` (links to the in-file headings). The TOC makes a 100-line reference scannable.
- Keep references **depth ≤ 1** — a reference may link to a **sibling** reference, but should not nest its own `reference/` directory.
- A reference should be **self-contained** for one topic, not a grab-bag.

### Cross-skill references

- Cross-skill links use `Skill(skill="<name>")` (e.g. `Skill(skill="dev-workflow")`). These work regardless of whether the sibling skill is split.
- Splitting a skill does **not** change cross-skill invocation — the public name in `.claude-plugin/plugin.json` and the `name:` in the main `SKILL.md` frontmatter remain stable.

## Before-after example — leader skill (v0.1.0 → v0.2.0)

The `leader` skill in this plugin was split in v0.2.0 as a working demonstration.

### Before (v0.1.0)

```
skills/leader/
├── SKILL.md                    275 lines
└── reference/
    ├── standup.md              107 lines
    ├── memory-format.md        129 lines
    ├── github-issues.md        102 lines
    └── handoff-from-butler.md   74 lines
```

`SKILL.md` mixed:

- Persona + trigger table (small, belongs in main).
- Full standup step-by-step (large, repeated in `reference/standup.md`).
- Wrap-up / good-bye / retrospective flows (medium, all in main).
- Orchestrator constraint table + agent routing table + memory-writes event table (medium, all in main).
- Show-task-list procedure + priority heuristics + recommendation format (medium, partially repeated in `reference/standup.md`).

### After (v0.2.0)

```
skills/leader/
├── SKILL.md                    109 lines   ← persona + trigger + flow outline + references table
└── reference/
    ├── standup.md              112 lines   ← already existed, frontmatter added
    ├── memory-format.md        134 lines   ← already existed, frontmatter added
    ├── github-issues.md        107 lines   ← already existed, frontmatter added
    ├── handoff-from-butler.md   79 lines   ← already existed, frontmatter added
    ├── orchestration.md         83 lines   ← NEW: persona + orchestrator constraint + agent routing + memory writes
    ├── wrap-up.md               84 lines   ← NEW: wrap-up + good-bye + retrospective
    └── task-list.md             57 lines   ← NEW: show task list + priority heuristics + recommendation format
```

Result:

- Main entry shrank from **275 → 109 lines** (60% reduction).
- Total content roughly unchanged (a small amount of duplication was de-duplicated by linking).
- Each reference is focused on one topic and is independently readable.
- The trigger table in the main entry now has a `Reference` column pointing at the file with the detail.

## Steps to split your own skill

1. **Inventory** — list every section in the current `SKILL.md` with its line count.
2. **Classify** each section as `keep` (persona / trigger / flow outline) or `move` (step-by-step / format / side-flow / exhaustive table).
3. **Cluster** the `move` sections into 3–5 cohesive reference files (one topic per file).
4. **Write the references first** — copy the content out, add YAML frontmatter + TOC.
5. **Slim the main entry** — replace each moved section with a 1–3 line summary + a link to the new reference.
6. **Add a References table** — give the user a single index of every reference and when to read it.
7. **Verify functional parity** — the skill's externally observable behavior must be unchanged. Trigger phrases, cross-skill invocations, file paths, and output formats stay identical.
8. **Commit in 1–2 commits** — one "split refactor" commit, optionally one "doc polish" follow-up.

## Relation to the dream skill's health check

The `dream` skill (also in this plugin) includes a **Skill Health Check** phase that can suggest a split when it detects a `SKILL.md` exceeding the recommended length. The output is an `R-SH-N` proposal — "skill health" — recorded in the retrospective and surfaced in the next standup for accept / reject / defer. Consumers who want this safety net wired automatically can let `dream` propose splits and use this document as the playbook for executing them.
