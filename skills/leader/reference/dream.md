# dream — Memory Consolidation + Daily Log Update + Self-Improvement + Skill Health Check

This reference describes the **dream** skill — a 7-Phase memory-consolidation + self-improvement loop. The implementation is project-agnostic: all signal sources are **auto-discovered** by scanning a fixed set of repo-relative paths, and the GitHub repo / push branch are read at runtime.

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
- [R-N / R-D / R-S / R-SH relationship](#r-n--r-d--r-s--r-sh-relationship)

## Trigger

1. **Voice / text trigger**: "dreaming", "consolidate memory", "dream", "auto dream" — run inline in current session.
2. **Agent() dispatch (auto)**: the calling skill checks the time since the last dream run (`cache/dream_last_at`) and, if 24h has elapsed, dispatches dream with `Agent(subagent_type="general-purpose", run_in_background=True)`. The backend runs inside the Max-Plan subagent pool (no `claude -p` subprocess, no external API key).

## Mode

| Mode      | Description | Default |
|-----------|-------------|---------|
| `dry-run` | Only build a git diff. No writes to memory or daily log. | **default** |
| `apply`   | Real Edit/Write + final git commit. | explicit opt-in |

Callers must pass `mode: dry-run` or `mode: apply`. Unknown → treat as **dry-run** (fail-safe).

**Phase 5 exception:** even in dry-run, if the day's daily file does not exist, create only the header (no auto-generated zone) so handwritten notes are not lost.

## Auto-discovered signal sources

dream **scans a fixed set of repo-relative paths** at startup. Each path that exists is used as a signal source; missing paths are silently skipped (treated as empty signal).

```
Standard signal-source paths (all repo-relative):

  .claude/projects/-home-pi-<project>/*.jsonl       Text session logs (Claude Code)
  .claude-voice/projects/-home-pi-<project>/*.jsonl Voice session logs (Claude Code)
  knowledge/voice-logs/*.jsonl                      Voice realtime turn logs (if any)
  memory/users/*/daily/*.md                         Per-user daily logs (last 3 days)
  memory/dev/daily/*.md                             Dev daily logs (last 3 days)
```

Implementation: at the start of Phase 2, run `ls`/`find` on each path and build a list of existing sources. If **all** paths are empty, log "no signal source available" and proceed to Phase 4 prune only (Phase 5/6/7 still run, scoped to whatever input exists).

The exact project-name component (`-home-pi-<project>`) is auto-detected from `.claude/projects/*/` (Claude Code's per-project log directory uses the absolute path with `/` → `-`).

## Allowed Zone

dream may **read** anything under the project root, but **writes** are restricted to:

```
memory/auto/                                  ← auto-memory layer
  MEMORY.md (≤ 200 lines)
  <topic>.md files

memory/shared/notes/                          ← R-D / R-S / R-SH output files (create / append only)
  dream-suggestions-YYYY-MM-DD.md
  dream-skill-suggestions-YYYY-MM-DD.md

memory/users/<user>/daily/YYYY-MM-DD.md       ← Phase 5: auto-generated marker zone only
memory/dev/daily/YYYY-MM-DD.md                ← Phase 5: auto-generated marker zone only

cache/dream_last_at                           ← apply-mode post-processing
```

If a consumer project does not have a `memory/users/` tree (no per-user concept), Phase 5 falls back to `memory/dev/daily/YYYY-MM-DD.md` only.

## Protect List

```
memory/shared/decisions/      # finalized decisions (read-only)
memory/shared/notes/          # living notes (read-only EXCEPT dream-*-suggestions-*.md)
memory/users/*/profile.md     # per-user profile (read-only)
memory/users/*/daily/         # daily logs (Phase 5 may rewrite only the auto-generated zone)
memory/dev/                   # leader memory (read-only; Phase 5 may rewrite only the auto-generated zone in daily/)
.git/                         # git metadata
scripts/                      # script bodies
.claude/skills/               # skill defs (Phase 6/7 emit R-S only, never edits)
.claude/settings.json
.claude/settings.local.json
CLAUDE.md                     # project instructions

# Plus any local secrets file (e.g. a local *.env outside the repo) — never written, never echoed.
```

**Exceptions:**
- `memory/shared/notes/dream-suggestions-YYYY-MM-DD.md` — create / append only (R-D output)
- `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` — create / append only (R-S / R-SH output)
- `memory/users/<user>/daily/<today>.md` and `memory/dev/daily/<today>.md`, **only** between `<!-- auto-generated:start -->` and `<!-- auto-generated:end -->` markers — Phase 5 full re-render

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
   ls -t .claude-voice/projects/*/*.jsonl 2>/dev/null | head -10
   ls -t knowledge/voice-logs/*.jsonl 2>/dev/null | head -10
   ```
2. Run narrow queries (one at a time, via Bash). Each is `grep -h <pattern> <files>`:
   - save-intent (Japanese): `覚えて\|記憶して\|メモして\|忘れないで`
   - save-intent (English): `remember\|save this\|note this\|don't forget`
   - correction (Japanese): `違う\|間違い\|そうじゃない\|訂正`
   - correction (English): `wrong\|that's not\|correction\|actually`
   - decision (Japanese): `決定\|確定\|採用\|やめる`
   - decision (English): `decide\|decided\|adopt\|drop\|reject`
   - recurrence (Japanese): `毎週\|毎回\|いつも\|繰り返し`
   - recurrence (English): `every week\|every time\|always\|recurring`
   - friction (Japanese): `わかりにくい\|使いにくい\|失敗\|遅い\|不便`
   - friction (English): `confusing\|hard to use\|fail\|slow\|inconvenient`
3. Read the most recent 3 daily files per user (`memory/users/*/daily/*.md`) and per dev (`memory/dev/daily/*.md`) and extract decision / proposal sections.
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
- **Anthropic SDK / `claude -p` are NOT used.** The summary is produced by this same agent (Max-Plan subagent pool).
- Anything outside the markers (preceding header, handwritten notes) is preserved.

**Target daily files (auto-discovery):**

- If `memory/users/` exists: process each `memory/users/<user>/daily/<today>.md` for users that have today's JSONL entries.
- Always process `memory/dev/daily/<today>.md` (leader's own daily log).

**Steps:**

1. For each target daily file, pull today's JSONL entries (already collected in Phase 2). Zero entries → write "N/A" template only.
2. Render the 6 sections (see below) — 1–2 sentences each / dedup / **always emit all sections (no truncation)** / total ≤ 3500 tokens. Language follows the consumer's convention (e.g. Japanese for family-facing logs, English for dev logs); detect from the existing file's content if unsure.
3. Replace the marker zone (or append at file end if no markers exist). If the file itself is missing, create it with a header first (e.g. `# YYYY-MM-DD\n\n` for dev, `# YYYY年MM月DD日\n\n## 今日の予定\n\n` for family-facing).

**Sections (in order):**

- `## 完了したこと` / `## Work Done`
- `## 決定事項` / `## Decisions`
- `## 次のアクション` / `## Next Actions`
- `## 今日のハイライト` / `## Highlights`
- `## 未解決の論点` / `## Open questions`
- `## Retrospective 提案` / `## Retrospective proposals`

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

### Phase 5 post-hook — memory hybrid search index (#643 MEM-S3, consumer-specific)

After Phase 5 finishes rewriting the daily marker zone(s), consumer projects that ship the Butler hybrid-search index (`scripts/walk_memory.py` + `scripts/lib/memory_search.py`) should run an incremental walk so the BM25 + embedding index stays in sync with the freshly updated memory tree:

```bash
python3 scripts/walk_memory.py --quiet  # incremental, ~seconds for unchanged trees
```

This is a no-op (and silent) for projects that do not have `walk_memory.py`. Run it after Phase 5's marker zone write, before Phase 6 starts.

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
   - Body: see "R-S issue body template" below.

5. Back-write the filed issue number into the R-S list line.

**dry-run:** file the markdown only with `(dry-run)` marker. **Do not** file issues.
**apply:** file markdown + issues + back-write.

### R-S issue body template

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

### Voice/Service Error Pattern Scan (#615 OBS-S1)

Phase 6 末尾の sub-step として、 **service journal log の error burst** を自動 scan し、 閾値超過時に **R-Self proposal** を 1 件 1 Issue として起票する。 これは error-pattern 定義モジュールが単一 source (single-source principle)。

**Scope:** consumer が指定する service (例: 常駐 voice/realtime service) を default、 config に追加することで他の常駐 service も同 framework で扱える (AC7)。サービス名は consumer config から読む (ここでは generic placeholder で表記)。

**Detected patterns (= `ERROR_PATTERNS` 定義済):**

| pattern | category | 24h thresh | consecutive thresh | service |
|---------|----------|-----------:|-------------------:|---------|
| `response_empty` | voice_llm_empty | 3 | 2 | <voice-service> |
| `rate_limit_exceeded` | voice_rate_limit | 2 | 2 | <voice-service> |
| `stt_hallucination_burst` | voice_stt_quality | 10 | (disabled) | <voice-service> |
| `idle_timeout_burst` | voice_session_quality | 20 | (disabled) | <voice-service> |
| `response_failed` | voice_llm_failed | 2 | 2 | <voice-service> |

**Steps (apply mode):**

1. `from scripts.lib.voice_error_patterns import scan_journal, detect_threshold_breach, format_r_self_proposal`
2. `scan = scan_journal(since_iso="24 hours ago")` — journalctl --user -u <service> --since "24 hours ago" を全 pattern について walk。
3. `breaches = detect_threshold_breach(scan)` — 24h count >= threshold OR consecutive >= threshold で Breach 抽出。
4. `for b in breaches:` 各 breach について:
   - `proposal = format_r_self_proposal(b)` で `{"title", "body", "labels": ["type:dev","status:action_required","user:<owner>"]}` を得る。
   - 既存 R-Self cap (retrospective policy / 1/day) と dedup (proposal-simplicity) を共通 lib (`self_improvement_cap` / `self_improvement_dedup`) 経由で適用。
   - cap 枠内 + 同一 category の open Issue が無ければ `gh issue create --title <title> --body <body> --label "type:dev,status:action_required,user:<owner>"` で起票。
5. R-Self list (= 既存の `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md`) に 1 行追記:
   `- R-Self<N>: voice_error / <pattern_name> / 24h=<count> / #<issue>`

**CLI 確認 (= dry-run 用):**

```bash
python3 -m scripts.lib.voice_error_patterns --since "24 hours ago"           # human readable
python3 -m scripts.lib.voice_error_patterns --since "24 hours ago" --json    # 自動化用
```

**dry-run mode:** breach detection は実行するが Issue 起票しない (= scan + proposal の console preview のみ)。
**apply mode:** breach 検出 → cap/dedup 経由で 1 Issue 1 breach 起票。 既存 Phase 6 R-S と同じ cap を共有 (retrospective policy)。

**多 service 拡張 (AC7):**

新 service の error を扱うには `ERROR_PATTERNS` に dict を追加するだけ:

```python
{
    "name": "<pattern_name>",
    "regex": r"<error log regex>",
    "threshold_24h": <N>,
    "threshold_consecutive": <N or 0>,
    "category": "<category_label>",  # 例: reservation_quality / toku_session_drop
    "service": "<consumer>-<svc>.service",
    "fix_hint": "<推奨 fix 1-3 行>",
}
```

regex は **fail-first compile** で起動時に検証される (retrospective policy)。

**Secrets handling:** journal sample line は `_mask_secrets()` で `org-*` / `sk-*` / `ghp_*` / `Bearer *` を mask してから body に貼る (secret-handling / api-key policy)。

## Phase 7 Skill Health Check

**Goal:** scan `.claude/skills/*/SKILL.md` for line-count / structure anomalies and emit one **R-SH GitHub issue** per detection.

**Thresholds:**

| Threshold | Level | Source |
|-----------|-------|--------|
| SKILL.md > 500 lines | WARNING | Anthropic platform best-practices |
| SKILL.md > 1000 lines | CRITICAL | community observed accuracy drop |
| reference > 100 lines without TOC | WARNING | superpowers convention |
| nested reference (`reference/x.md` → `reference/y.md` Read) | WARNING | official "reference depth = 1" |
| frontmatter description > 200 chars | WARNING | skill-listing budget |

**Steps:**

1. Read every SKILL.md. Compute line count + frontmatter length + reference link inventory.
2. For each `reference/*.md`, check for TOC (`## TOC` or equivalent within the first 30 lines).
3. Grep each `reference/*.md` for inter-reference links to detect nesting.
4. For each detection, generate a compression diff proposal (Edit form: which section moves to which reference, with line-savings estimate).
5. File one R-SH issue per detection (labels `status:action_required`, `type:dev`).
   - Title: `R-SH<N> (YYYY-MM-DD): <skill> SKILL.md hygiene — <kind>`
6. User accepts → main session applies the diff (commit + push). Rejects → close issue. **No auto-apply.**

### R-SH issue body template

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
  Modified: memory/users/<user>/daily/YYYY-MM-DD.md (auto-generated zone)
  Modified: memory/dev/daily/YYYY-MM-DD.md (auto-generated zone)
  New: memory/shared/notes/dream-suggestions-YYYY-MM-DD.md
  New: memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md

git commit (apply only): <hash>
```

## R-N / R-D / R-S / R-SH relationship

| Prefix | Source | Output file | Standup surface | adoption row | Auto-file issue |
|--------|--------|-------------|-----------------|--------------|-----------------|
| R-N | daily log (Phase 5) | `memory/users/<u>/daily/YYYY-MM-DD.md` `## Retrospective` | yes | accept/reject/defer | accept only |
| R-D | dream consolidate (Phase 4 trailer) | `memory/shared/notes/dream-suggestions-YYYY-MM-DD.md` | yes | accept/reject/defer | accept only |
| R-S | dream self-improvement (Phase 6) | `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` | yes | accept/reject/defer | **filed at generation** (`status:action_required` + `type:dev`) |
| R-SH | dream skill health check (Phase 7) | same file (append) | yes | accept/reject/defer | **filed at generation** (`status:action_required` + `type:dev`) |

R-S / R-SH are pre-filed at generation; the maintainer's adjudication becomes "accept (= start work) / reject (= close issue) / defer (= keep `status:blocked`)".

## Design notes

- Model weights are unchanged (this is structured note-taking, not re-training).
- Batch process (between sessions only).
- Aborts on insufficient signal.
- No new goals are set.
- **SKILL.md edits are never automated** (Phase 6/7 emit R-S / R-SH only; main session applies after user approval).
