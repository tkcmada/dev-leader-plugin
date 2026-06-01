# Signal sources — auto-discovery + per-source semantics

This reference documents the standard signal sources that Phase 2 (Gather Signal) auto-discovers, and the per-source semantics that Phase 5 / Phase 6 rely on.

## TOC

- [Source catalog](#source-catalog)
- [Auto-discovery algorithm](#auto-discovery-algorithm)
- [Per-source semantics](#per-source-semantics)
- [Privacy / scope filters](#privacy--scope-filters)
- [Missing-source handling](#missing-source-handling)

## Source catalog

| Path glob | What it contains | Phase consumers |
|-----------|------------------|------------------|
| `.claude/projects/<slug>/*.jsonl` | Claude Code text session logs (per-turn) | Phase 2 (signal), Phase 5 (today) |
| `memory/users/<actor>/daily/*.md` | Per-actor handwritten daily logs | Phase 2 (decisions / proposals), Phase 5 (write target) |
| `memory/dev/daily/*.md` | Dev daily logs | Phase 2 (decisions / proposals), Phase 5 (write target) |

The `<slug>` for `.claude/projects/` is the project's absolute path with `/` replaced by `-` (e.g. `/home/user/myproject` → `-home-user-myproject`).

Consumers that maintain additional per-turn log sources (separate voice-session log directories, project-specific event logs, etc.) can extend the auto-discovery list inside their wrapping skill — the dream skill itself does not assume any source beyond the three above.

## Auto-discovery algorithm

At the start of Phase 2:

```bash
# 1. List all candidate sources
for pattern in \
  '.claude/projects/*/*.jsonl' \
  'memory/users/*/daily/*.md' \
  'memory/dev/daily/*.md'; do
  ls -t $pattern 2>/dev/null | head -N
done

# 2. If every glob returns zero files: log "no signal source available"
#    Phase 4 prune still runs. Phase 5 header-create still runs.
```

**Recency budget:**
- text JSONL: most recent 20 files (1 file ≈ 1 session)
- per-actor daily / dev daily: most recent 3 files per dir

## Per-source semantics

### `.claude/projects/<slug>/*.jsonl`

Each line is a Claude Code turn (assistant + tool calls or user message). Phase 2 runs grep patterns for save-intent / correction / decision / recurrence / friction. Phase 5 uses today's lines (timestamp filter) to render the auto-generated zone.

### `memory/users/<actor>/daily/*.md`

Read the last 3 days. Extract sections matching `## Decisions` / `## Decision` / `## 決定事項` / `## Retrospective proposals`. These feed Phase 2's "decisions" candidate pool.

### `memory/dev/daily/*.md`

Same as per-actor daily, but for dev work (typically maintained by a `leader` consumer).

## Privacy / scope filters

Consumers that surface personal or always-on voice content should apply additional filters at Phase 2 ingestion. The dream skill itself does NOT enforce these — they live in the **consumer**'s wrapping skill. Recommended filters:

- **Wake-word-only ingestion** for any always-on voice log source: only include turns that follow a wake-word event. Background ambient conversation is excluded.
- **Per-actor exclusion** for sensitive actors: the consumer marks the actor as `excluded` and dream skips their daily / JSONL.
- **Profile files are off-limits**: `memory/users/<actor>/profile.md` is in the Protect List — not even read.

## Missing-source handling

| Condition | Behaviour |
|-----------|-----------|
| One glob is empty | Silent skip; the remaining sources are used |
| All globs are empty | Log "no signal source available"; Phase 4 prune still runs; Phase 5 header-create still runs |
| `memory/auto/` missing | **Abort** dream (see SKILL.md startup checks) |
| `memory/users/` missing but `memory/dev/daily/` present | Phase 5 falls back to dev-only daily update |
| `memory/dev/daily/` missing but `memory/users/` present | Phase 5 processes per-actor daily only |
| Both daily dirs missing | Phase 5 logs "no daily targets" and exits cleanly |
