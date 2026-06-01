# Phase details — Phase 1–7 step-by-step

This reference expands the Phase 1–7 detail beyond what SKILL.md contains. SKILL.md is the canonical entry; this file is loaded on demand when an operator wants the full step-by-step.

## TOC

- [Phase 1 Orient — full steps](#phase-1-orient--full-steps)
- [Phase 2 Gather Signal — full procedure](#phase-2-gather-signal--full-procedure)
- [Phase 3 Consolidate — merge invariants](#phase-3-consolidate--merge-invariants)
- [Phase 4 Prune & Index — pruning rules + R-D emission](#phase-4-prune--index--pruning-rules--r-d-emission)
- [Phase 5 Daily Log Update — section rendering](#phase-5-daily-log-update--section-rendering)
- [Phase 6 Self-Improvement — R-S issue body template](#phase-6-self-improvement--r-s-issue-body-template)
- [Phase 7 Skill Health Check — R-SH issue body template](#phase-7-skill-health-check--r-sh-issue-body-template)

## Phase 1 Orient — full steps

**Goal:** snapshot the current memory.

1. `Bash: ls -la memory/auto/` — list topic files
2. `Read: memory/auto/MEMORY.md`
3. Group topic file names into rough themes (e.g. "user feedback", "decisions", "project facts")
4. `Bash: stat -c '%y %n' memory/auto/*.md` — get last-modified dates per file
5. Record baseline line count of MEMORY.md for Phase 4 sudden-shrink check

## Phase 2 Gather Signal — full procedure

**Goal:** extract recurring themes / user corrections / explicit save instructions / important decisions / friction from prior session logs.

**Forbidden:** exhaustive read of all JSONL files. Always use narrow `grep` queries.

### Query patterns (locale-aware)

Adapt the locale-specific patterns based on the consumer project's primary language. Defaults below cover Japanese + English:

| Signal | Japanese pattern | English pattern |
|--------|------------------|-----------------|
| save-intent | `覚えて\|記憶して\|メモして\|忘れないで` | `remember\|save this\|note this\|don't forget` |
| correction | `違う\|間違い\|そうじゃない\|訂正` | `wrong\|that's not\|correction\|actually` |
| decision | `決定\|確定\|採用\|やめる` | `decide\|decided\|adopt\|drop\|reject` |
| recurrence | `毎週\|毎回\|いつも\|繰り返し` | `every week\|every time\|always\|recurring` |
| friction | `わかりにくい\|使いにくい\|失敗\|遅い\|不便` | `confusing\|hard to use\|fail\|slow\|inconvenient` |

### Steps

1. Auto-discover available JSONL sources:
   ```bash
   ls -t .claude/projects/*/*.jsonl 2>/dev/null | head -20
   # consumer-extended sources (if any) go here
   ```
2. Run each grep pattern as a separate `Bash` call.
3. Read the most recent 3 daily files per actor (`memory/users/*/daily/*.md`) and per dev (`memory/dev/daily/*.md`).
4. Aggregate into **5–10 candidate topics** (defer the rest).
5. Separately collect today's JSONL entries (all sources) — these feed Phase 5.

## Phase 3 Consolidate — merge invariants

**Goal:** merge new signal into existing topics, absolutize relative dates, resolve contradictions.

**Invariants (apply-mode):**

- Every Edit is **1 file = 1 operation**. No bulk updates.
- Always present the diff in chat **before** Edit.
- Absolutize relative dates against today's date (e.g. "last week" → `YYYY-MM-DD`).
- When new and old information contradict, prefer **new**, but record the change reason in a side comment.
- Drop stale references (links to deleted files, expired URLs).

**New-topic criteria:**

- Only materialize a new candidate that was confirmed by **repeat observation** (≥ 2 distinct sessions or daily logs).

## Phase 4 Prune & Index — pruning rules + R-D emission

**Goal:** rebuild `MEMORY.md` ≤ 200 lines, drop obsolete pointers, re-rank by relevance.

### Pruning steps

1. Read current `MEMORY.md`.
2. For each entry, `Bash: ls` to confirm the topic file still exists. Missing → delete candidate.
3. Re-rank by `last-modified` recency + Phase 2 signal strength.
4. Overflow > 200 lines: demote into topic-file body and keep only a summary line in the index.

### Sudden-shrink guard

- If the new line count is `< 50%` of the old, **auto abort**.
- In apply-mode also skip the commit.
- Log the abort reason and the old/new counts.

### R-D emission (Phase 4 trailer)

Write cross-cutting behavioural proposals that don't fit any single topic file to
`memory/shared/notes/dream-suggestions-YYYY-MM-DD.md`:

```markdown
# Dream Suggestions YYYY-MM-DD

Cross-cutting patterns extracted in this dreaming session. The next standup surfaces them
for accept / reject / defer adjudication.

- R-D1: <pattern> / <evidence (which daily / JSONL)> / <proposed action>
- R-D2: <...>
```

**Rules:**
- `R-D` prefix is mandatory.
- Same-day file → append (do not overwrite).
- Zero patterns → write `- N/A` only.
- Dry-run also writes this file.

## Phase 5 Daily Log Update — section rendering

**Goal:** regenerate the `<!-- auto-generated:start -->` ~ `<!-- auto-generated:end -->` zone of today's daily file(s).

### Target file selection

- If `memory/users/` exists: process each `memory/users/<actor>/daily/<today>.md` for actors that have today's JSONL entries.
- If `memory/dev/daily/` exists: process `memory/dev/daily/<today>.md`.

### Section template

Total ≤ 3500 tokens. 1–2 sentences per section. **Always emit all sections** (no truncation).

```
<!-- auto-generated:start -->
<!-- Last updated: YYYY-MM-DD HH:MM by dream skill (Phase 5) -->

## Work Done
- (1–2 sentences)

## Decisions
- (1–2 sentences)

## Next Actions
- (1–2 sentences)

## Highlights
- (1–2 sentences)

## Open questions
- (1–2 sentences)

## Retrospective proposals
- R-N1: ... (or `- N/A`)
<!-- auto-generated:end -->
```

### Locale detection

Detect language from the existing file's content. Japanese consumers use:
- `## 完了したこと` (Work Done) / `## 決定事項` (Decisions) / `## 次のアクション` (Next Actions) / `## 今日のハイライト` (Highlights) / `## 未解決の論点` (Open questions) / `## Retrospective 提案`

### Missing-file behaviour

If the daily file does not exist:
- **dry-run**: create header only (e.g. `# YYYY-MM-DD\n\n`). Do not insert the marker zone.
- **apply**: create header + marker zone.

## Phase 6 Self-Improvement — R-S issue body template

```markdown
## Proposal (Dream Phase 6 R-S<N>)

- **Target skill**: <skill> (`.claude/skills/<name>/SKILL.md`)
- **Section**: <SKILL.md heading>
- **Observed pattern**: <specific>
- **Proposed change**: <what / how>
- **Estimated impact**: <UX / latency / failure rate>
- **Estimated cost**: small / medium / large

## Source

- Generated: YYYY-MM-DD (Dream Phase 6)
- dream-skill-suggestions: [memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md](...)
- Evidence: <daily / JSONL excerpt>

## Next action

Owner: maintainer
Action: review the R-S; if accepted, ask the main session to edit the SKILL.md. If rejected, close the issue.
```

## Phase 7 Skill Health Check — R-SH issue body template

```markdown
## Proposal (Dream Phase 7 R-SH<N> Skill Health Check)

- **Target skill**: <skill> (`.claude/skills/<name>/SKILL.md`)
- **Detection**: SKILL.md > 500 lines WARNING / > 1000 lines CRITICAL / reference > 100 lines no TOC / nested reference / description > 200 chars
- **Current value**: e.g. SKILL.md 612 lines (recommended max 500 / 122%)
- **Compression diff** (Edit form):

  \`\`\`diff
  - ## Long section (XX lines)
  - ...
  + ## Long section (5 lines)
  + Details: see `reference/<topic>.md`
  \`\`\`

- **Reference target**: `.claude/skills/<name>/reference/<topic>.md`
- **Estimated impact**: SKILL.md X → Y lines / token Z% reduction

## Source

- Generated: YYYY-MM-DD (Dream Phase 7)
- Detection scan: `dream-skill-suggestions-YYYY-MM-DD.md` row R-SH<N>

## Next action

Owner: maintainer
Action: review the compression diff; accept → main session applies (commit + push). Reject → close.
```
