---
name: dream
description: Memory consolidation + daily log update + skill self-improvement + skill health check. 7-Phase process invoked manually by the operator (no auto-dispatch from sibling skills). Project-agnostic — signal sources are auto-discovered, GitHub repo / push branch are resolved at runtime.
---

# dream — Memory Consolidation + Daily Log Update + Self-Improvement + Skill Health Check

The **dream** skill is a 7-Phase memory-consolidation + self-improvement loop. The implementation is project-agnostic: all signal sources are **auto-discovered** by scanning a fixed set of repo-relative paths, and the GitHub repo / push branch are read at runtime.

Inspired by REM-sleep memory consolidation (Anthropic Dreaming, Code with Claude 2026). The original 4 Phase pattern is extended here with Phase 5 (Daily Log Update), Phase 6 (Self-Improvement), and Phase 7 (Skill Health Check).

## TOC

- [Trigger](#trigger)
- [Mode](#mode)
- [Auto-discovered signal sources](#auto-discovered-signal-sources)
- [Allowed Zone](#allowed-zone)
- [Protect List](#protect-list)
- [Startup safety checks](#startup-safety-checks)
- [Phase 1 Orient](#phase-1-orient)
- [Phase 2 Gather Signal](#phase-2-gather-signal)
- [Phase 3 Consolidate](#phase-3-consolidate)
- [Phase 4 Prune & Index](#phase-4-prune--index)
- [Phase 5 Daily Log Update](#phase-5-daily-log-update)
- [Phase 6 Self-Improvement](#phase-6-self-improvement)
- [Phase 7 Skill Health Check](#phase-7-skill-health-check)
- [Mode behaviour summary](#mode-behaviour-summary)
- [apply-mode post-processing](#apply-mode-post-processing)
- [Failure handling](#failure-handling)
- [Report format](#report-format)
- [R-N / R-D / R-S / R-SH relationship](reference/outcome.md)
- [Phase details](reference/phases.md)
- [Signal sources](reference/signal-sources.md)

## Trigger

**Manual invocation only.** Voice / text triggers: "dreaming", "consolidate memory", "dream", "auto dream" — run inline in the current session.

No sibling skill in this plugin auto-dispatches dream. If you want scheduled runs (e.g. once-per-day), wire it up at the consumer / wrapping-skill / cron layer outside this plugin. The operator stays in control of when consolidation happens.

## Mode

| Mode      | Description | Default |
|-----------|-------------|---------|
| `dry-run` | Only build a diff. No writes to memory or daily log. | **default** |
| `apply`   | Real Edit/Write + final git commit. | explicit opt-in |

Callers must pass `mode: dry-run` or `mode: apply`. Unknown → treat as **dry-run** (fail-safe).

**Phase 5 exception:** even in dry-run, if the day's daily file does not exist, create only the header (no auto-generated zone) so handwritten notes are not lost.

## Auto-discovered signal sources

dream **scans a fixed set of repo-relative paths** at startup. Each path that exists is used as a signal source; missing paths are silently skipped (treated as empty signal).

```
Standard signal-source paths (all repo-relative):

  .claude/projects/<project-slug>/*.jsonl           Text session logs (Claude Code)
  memory/users/*/daily/*.md                         Per-actor daily logs (last 3 days)
  memory/dev/daily/*.md                             Dev daily logs (last 3 days)
```

Consumers may add their own log sources by extending the auto-discovery list in their wrapping skill (e.g. a separate voice-session log directory or a project-specific turn-log file). The dream skill itself only knows the three sources above.

Implementation: at the start of Phase 2, run `ls`/`find` on each path and build a list of existing sources. If **all** paths are empty, log "no signal source available" and proceed to Phase 4 prune only (Phase 5/6/7 still run, scoped to whatever input exists).

The exact `<project-slug>` is auto-detected from `.claude/projects/*/` (Claude Code's per-project log directory uses the absolute project path with `/` → `-`).

Full source-by-source semantics → `reference/signal-sources.md`.

## Allowed Zone

dream may **read** anything under the project root, but **writes** are restricted to:

```
memory/auto/                                  ← auto-memory layer
  MEMORY.md (≤ 200 lines)
  <topic>.md files

memory/shared/notes/                          ← R-D / R-S / R-SH output files (create / append only)
  dream-suggestions-YYYY-MM-DD.md
  dream-skill-suggestions-YYYY-MM-DD.md

memory/users/<actor>/daily/YYYY-MM-DD.md      ← Phase 5: auto-generated marker zone only
memory/dev/daily/YYYY-MM-DD.md                ← Phase 5: auto-generated marker zone only

cache/dream_last_at                           ← apply-mode post-processing
```

If a consumer project does not have a `memory/users/` tree (no per-actor concept), Phase 5 falls back to `memory/dev/daily/YYYY-MM-DD.md` only.

## Protect List

```
memory/shared/decisions/      # finalized decisions (read-only)
memory/shared/notes/          # living notes (read-only EXCEPT dream-*-suggestions-*.md)
memory/users/*/profile.md     # per-actor profile (read-only)
memory/users/*/daily/         # daily logs (Phase 5 may rewrite only the auto-generated zone)
memory/dev/                   # dev memory (read-only; Phase 5 may rewrite only the auto-generated zone in daily/)
.git/                         # git metadata
scripts/                      # script bodies
.claude/skills/               # skill defs (Phase 6/7 emit R-S / R-SH only, never edits)
.claude/settings.json
.claude/settings.local.json
CLAUDE.md                     # project instructions

# Plus any local secrets file (e.g. ~/.secrets/*.env) — never written, never echoed.
```

**Exceptions:**
- `memory/shared/notes/dream-suggestions-YYYY-MM-DD.md` — create / append only (R-D output)
- `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` — create / append only (R-S / R-SH output)
- `memory/users/<actor>/daily/<today>.md` and `memory/dev/daily/<today>.md`, **only** between `<!-- auto-generated:start -->` and `<!-- auto-generated:end -->` markers — Phase 5 full re-render

## Startup safety checks

Before any real work:

1. **Protect-list existence sanity check** (warn-only, do not abort):
   - `memory/shared/decisions/index.md` (if memory/shared/ exists)
   - `.git/HEAD`

2. **Memory dir existence**: if `memory/auto/` does not exist, **abort** with a clear message.

3. **Cache dir**: `mkdir -p cache/` if missing.

4. **Baseline memory size** (for Phase 4 sudden-shrink detection):
   - record `MEMORY.md` line count
   - record total bytes of topic files
   - if Phase 4 produces > 50% shrink → auto abort

## Phase 1 Orient

**Goal:** snapshot the current memory.

**Steps:**

1. `Bash: ls -la memory/auto/` — list topic files
2. `Read: memory/auto/MEMORY.md`
3. Group topic file names into rough themes
4. `Bash: stat -c '%y %n'` for last-modified dates

## Phase 2 Gather Signal

**Goal:** extract "recurring themes / user corrections / explicit save instructions / important decisions / friction" from prior session logs. This phase's output also seeds Phase 5 and Phase 6.

**Important:** exhaustive read is forbidden. Use **narrow queries** only.

**Steps:**

1. Auto-discover available JSONL sources (see "Auto-discovered signal sources" above). Take the most recent N files per source:
   ```bash
   ls -t .claude/projects/*/*.jsonl 2>/dev/null | head -20
   # consumer-extended sources (if any) go here
   ```
2. Run narrow queries (one at a time, via Bash). Each is `grep -h <pattern> <files>`. Full pattern list and locale-specific variants → `reference/signal-sources.md`.
3. Read the most recent 3 daily files per actor (`memory/users/*/daily/*.md`) and per dev (`memory/dev/daily/*.md`) and extract decision / proposal sections.
4. Aggregate into 5–10 candidate topics (defer the rest).
5. Separately collect today's JSONL entries (all sources) — they feed Phase 5.

## Phase 3 Consolidate

**Goal:** merge new signal into existing topics, absolutize relative dates, resolve contradictions.

**Steps:**

1. For each Phase 2 candidate, classify as **merge** or **new** vs existing topic files.
2. Merge processing (apply only): Read existing file, absolutize relative dates against today's date, prefer new info over contradicting old info, drop stale references.
3. For new candidates, only materialize ones that were confirmed by repeat observation.
4. All changes recorded as a diff (dry-run keeps it in memory).

**apply-mode invariants:** every Edit is 1 file 1 operation. No bulk updates. Always present the diff in chat before Edit.

## Phase 4 Prune & Index

**Goal:** rebuild `MEMORY.md` ≤ 200 lines, drop obsolete pointers, re-rank by relevance.

**Steps:**

1. Read current `MEMORY.md`.
2. For each entry, `Bash: ls` to confirm the topic file still exists. Missing → delete candidate.
3. Re-rank by `last-modified` recency + Phase 2 signal strength.
4. Overflow > 200 lines: demote into topic-file body and keep only a summary line in the index.
5. **Sudden-shrink guard:** if the new line count is < 50% of the old, **auto abort**. apply-mode also skips the commit.

### Phase 4 trailer: R-D emission

Write cross-cutting behavioural proposals (those that don't fit any single topic) to
`memory/shared/notes/dream-suggestions-YYYY-MM-DD.md`:

```markdown
# Dream Suggestions YYYY-MM-DD

Cross-cutting patterns extracted in this dreaming session. The next standup surfaces them
for accept / reject / defer adjudication.

- R-D1: <pattern> / <evidence (which daily / JSONL)> / <proposed action>
- R-D2: <...>
```

**Rules:** `R-D` prefix is mandatory. Same-day file → append. Zero patterns → write `- N/A` only. Dry-run also writes this file.

## Phase 5 Daily Log Update

**Goal:** regenerate the `<!-- auto-generated:start -->` ~ `<!-- auto-generated:end -->` zone of today's daily file(s) into a 6-section summary built from the Phase 2 today-entries.

**Design notes:**
- **External APIs / `claude -p` are NOT used.** The summary is produced by this same agent (host subagent pool).
- Anything outside the markers (preceding header, handwritten notes) is preserved.

**Target daily files (auto-discovery):**

- If `memory/users/` exists: process each `memory/users/<actor>/daily/<today>.md` for actors that have today's JSONL entries.
- Always process `memory/dev/daily/<today>.md` if the file or directory exists.

**Steps:**

1. For each target daily file, pull today's JSONL entries (already collected in Phase 2). Zero entries → write "N/A" template only.
2. Render the 6 sections (see below) — 1–2 sentences each / dedup / **always emit all sections (no truncation)** / total ≤ 3500 tokens. Language follows the consumer's convention; detect from the existing file's content if unsure.
3. Replace the marker zone (or append at file end if no markers exist). If the file itself is missing, create it with a header first.

**Sections (in order):**

- `## Work Done` / equivalent in consumer locale
- `## Decisions`
- `## Next Actions`
- `## Highlights`
- `## Open questions`
- `## Retrospective proposals`

Full phase-by-phase detail → `reference/phases.md`.

### Phase 5 marker zone example

```
<!-- auto-generated:start -->
<!-- Last updated: YYYY-MM-DD HH:MM by dream skill (Phase 5) -->

## Work Done
- ...

## Decisions
- ...

## Next Actions
- ...

## Highlights
- ...

## Open questions
- ...

## Retrospective proposals
- ...
<!-- auto-generated:end -->
```

## Phase 6 Self-Improvement

**Goal:** from Phase 2 friction signals and repeated observations not captured as R-N / R-D, generate **R-S (Skill suggestions)** that target the skills / agents themselves.

**Hard rules:**
- **One R-S = one GitHub issue.**
- **No auto-PR, no auto-commit.** SKILL.md edits are performed by the main session after user approval.
- Phase 6 only observes → emits R-S → files issues.

**Steps:**

1. Pick 3–5 candidate frictions targeting a specific skill / agent.
2. For each, capture: observed pattern / proposed change / target skill name / estimated impact / estimated cost (small / medium / large) / target SKILL.md section.
3. Append to `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md`:

   ```markdown
   # Dream Skill Suggestions YYYY-MM-DD

   - R-S1: <skill> / <pattern> / <proposal> / effort=small|medium|large / #<issue>
   - R-S2: ...
   ```

4. File each R-S as a separate GitHub issue (`gh issue create --label "status:action_required,type:dev"`).
   - Title: `R-S<N> (YYYY-MM-DD): <skill> — <one-line change>`
   - Body: see "R-S issue body template" in `reference/phases.md`.

5. Back-write the filed issue number into the R-S list line.

**dry-run:** file the markdown only with `(dry-run)` marker. **Do not** file issues.
**apply:** file markdown + issues + back-write.

## Phase 7 Skill Health Check

**Goal:** scan `.claude/skills/*/SKILL.md` for line-count / structure anomalies and emit one **R-SH GitHub issue** per detection.

**Thresholds:**

| Threshold | Level | Source |
|-----------|-------|--------|
| SKILL.md > 500 lines | WARNING | Anthropic platform best-practices |
| SKILL.md > 1000 lines | CRITICAL | community observed accuracy drop |
| reference > 100 lines without TOC | WARNING | community convention |
| nested reference (`reference/x.md` → `reference/y.md` Read) | WARNING | depth = 1 convention |
| frontmatter description > 200 chars | WARNING | skill-listing budget |

**Steps:**

1. Read every SKILL.md. Compute line count + frontmatter length + reference link inventory.
2. For each `reference/*.md`, check for TOC (`## TOC` or equivalent within the first 30 lines).
3. Grep each `reference/*.md` for inter-reference links to detect nesting.
4. For each detection, generate a compression diff proposal.
5. File one R-SH issue per detection (labels `status:action_required`, `type:dev`).
6. User accepts → main session applies the diff (commit + push). Rejects → close issue. **No auto-apply.**

Full template + post-processing detail → `reference/phases.md`.

## Mode behaviour summary

| Operation | dry-run | apply |
|-----------|---------|-------|
| Phase 1–2 reads | run | run |
| Phase 3 diff build | in-memory | run + present |
| topic Edit | no | yes |
| MEMORY.md rewrite | no | yes |
| dream-suggestions-*.md (R-D) | yes | yes |
| Phase 5 marker-zone rewrite | no (header-only create allowed) | yes |
| dream-skill-suggestions-*.md (R-S) | yes (with dry-run marker) | yes |
| Phase 6 issue filing | no | yes |
| Phase 7 scan | run (detect-only) | run (file R-SH) |
| git commit | no | yes |
| `cache/dream_last_at` touch | no | yes |

## apply-mode post-processing

1. Write current epoch to `cache/dream_last_at`.
2. (Optional, project-specific) Post a summary comment to any reference issues tracked by the consumer.

## Failure handling

1. **Protect-list violation detected**: immediately abort Edit/Write and return an error.
2. **Memory shrink > 50%**: Phase 4 auto-abort. No commit in apply.
3. **Zero JSONL**: skip Phase 2 ("no signal"). Phase 4 prune + Phase 5 header create still run.
4. **GitHub API failure**: record the R-S / R-SH line with `#TBD` for later re-filing.
5. **Mid-run abort**: do not touch the cache file so the next run is safe to retry.

## Report format

```
Dream consolidation complete (mode: dry-run|apply, date: YYYY-MM-DD)

Phase 1 Orient: <summary>
Phase 2 Gather: <summary>
Phase 3 Consolidate: <summary>
Phase 4 Prune & Index: <summary>
Phase 5 Daily Log Update: <summary>
Phase 6 Self-Improvement: <summary>
Phase 7 Skill Health Check: <summary>

R-D suggestions: N -> memory/shared/notes/dream-suggestions-YYYY-MM-DD.md
R-S suggestions: N -> memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md
R-SH suggestions: N (same file)
Filed issues: #N1, #N2, ...

Changed files:
  Modified: memory/auto/MEMORY.md
  Modified: memory/auto/<topic>.md
  Modified: memory/users/<actor>/daily/YYYY-MM-DD.md (auto-generated zone)
  Modified: memory/dev/daily/YYYY-MM-DD.md (auto-generated zone)
  New: memory/shared/notes/dream-suggestions-YYYY-MM-DD.md
  New: memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md

git commit (apply only): <hash>
```

## Design notes

- Model weights are unchanged (this is structured note-taking, not re-training).
- Batch process (between sessions only).
- Aborts on insufficient signal.
- No new goals are set.
- **SKILL.md edits are never automated** (Phase 6/7 emit R-S / R-SH only; main session applies after user approval).

## References

- `reference/phases.md` — Phase 1–7 detail (steps, templates, post-processing)
- `reference/signal-sources.md` — Auto-discovered source semantics, grep patterns, locale variants
- `reference/outcome.md` — R-N / R-D / R-S / R-SH 3-layer outcome loop relationship
