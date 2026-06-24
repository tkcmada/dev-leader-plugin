---
name: dream
description: Memory consolidation + daily log update + skill self-improvement + skill health check. 7-Phase BG process triggered by a consumer skill on a 24h cache. Project-agnostic (#445).
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
- [Phase 8 Self-Improvement Synthesis](#phase-8-self-improvement-synthesis)
- [Mode behaviour summary](#mode-behaviour-summary)
- [apply-mode post-processing](#apply-mode-post-processing)
- [Failure handling](#failure-handling)
- [Report format](#report-format)
- [R-N / R-D / R-S / R-SH / R-Self relationship](reference/outcome.md)
- [Phase details — full steps, templates, grep patterns, lib contracts](reference/phases.md)

## Trigger

1. **Voice / text trigger**: "dreaming", "consolidate memory", "dream", "auto dream" — run inline in current session.
2. **Agent() dispatch (auto)**: the calling skill checks the time since the last dream run (`cache/dream_last_at`) and, if 24h has elapsed, dispatches dream with `Agent(subagent_type="general-purpose", run_in_background=True)`. The backend runs inside the host platform's subagent pool (no external API key, no `claude -p` subprocess).

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
  .claude-voice/projects/<project-slug>/*.jsonl     Voice session logs (Claude Code, if any)
  knowledge/voice-logs/*.jsonl                      Voice realtime turn logs (if any)
  memory/users/*/daily/*.md                         Per-actor daily logs (last 3 days)
  memory/dev/daily/*.md                             Dev daily logs (last 3 days)
```

Implementation: at the start of Phase 2, run `ls`/`find` on each path and build a list of existing sources. If **all** paths are empty, log "no signal source available" and proceed to Phase 4 prune only (Phase 5/6/7 still run, scoped to whatever input exists).

The exact `<project-slug>` is auto-detected from `.claude/projects/*/` (Claude Code's per-project log directory uses the absolute project path with `/` → `-`).

Full source-by-source semantics + locale-aware grep patterns → `reference/phases.md` (Phase 2).

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
cache/self_improvement_today_count.json       ← Phase 8 cap counter (atomic rewrite)
cache/retro_proposal_today.json               ← Phase 6/7/8 retrospective 1/day TOTAL T-required cap (#493 RET-S1, atomic rewrite)
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

# Plus any local secrets file (e.g. a local *.env outside the repo) — never written, never echoed.
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
   ls -t .claude-voice/projects/*/*.jsonl 2>/dev/null | head -10
   ls -t knowledge/voice-logs/*.jsonl 2>/dev/null | head -10
   ```
2. Run narrow queries (one at a time, via Bash). Each is `grep -h <pattern> <files>`. Full pattern list and locale-specific variants → `reference/phases.md` (Phase 2).
3. Read the most recent 3 daily files per actor (`memory/users/*/daily/*.md`) and per dev (`memory/dev/daily/*.md`) and extract decision / proposal sections.
4. Aggregate into 5–10 candidate topics (defer the rest).
5. Separately collect today's JSONL entries (all sources) — they feed Phase 5.
6. **Memory access counting (#711 MEM-S2 Stage 1).** While the JSONL files are already open, compute each `memory/auto/*.md` file's **access_count** over a 14-day rolling window by scanning `Read` tool_use `file_path` arguments. This is deterministic (no LLM, no atime — the host `/` is `noatime` so atime is unusable). Call the shared lib, no re-implementation:

   ```bash
   python3 -m scripts.lib.memory_metrics --window 14
   ```

   The lib (`scripts/lib/memory_metrics.py`) returns `{file: {access_count, last_accessed}}`. Phase 3 uses this to refresh front-matter `importance`; Phase 4 re-ranks the index by it. Corrupt JSONL lines are skipped per-line (scan continues). See `reference/phases.md` (Phase 2) for the full contract.

## Phase 3 Consolidate

**Goal:** merge new signal into existing topics, absolutize relative dates, resolve contradictions.

**Steps:**

1. For each Phase 2 candidate, classify as **merge** or **new** vs existing topic files.
2. Merge processing (apply only): Read existing file, absolutize relative dates against today's date, prefer new info over contradicting old info, drop stale references.
3. For new candidates, only materialize ones that were confirmed by repeat observation.
4. All changes recorded as a diff (dry-run keeps it in memory).

**apply-mode invariants:** every Edit is 1 file 1 operation. No bulk updates. Always present the diff in chat before Edit.

### Front-matter importance schema (#711 MEM-S2 Stage 1)

Phase 3 maintains an importance signal in each `memory/auto/*.md` front-matter (`metadata:` mapping): `severity` (1=LOW/3=MED/10=HIGH, default 3), `access_count` (14-day Read count), `last_accessed` (YYYY-MM-DD), `importance` (= severity × access_freq × decay), `tags` (Stage 2 fills). **decay = exp(−λ·days_since_last_access)** (Ebbinghaus, λ≈0.05 ≈ 14-day half-life — a tunable knob T confirms per `feedback_standard_change_needs_t`). Only `severity` is non-deterministic (LLM-assigned at creation; **lowering an existing value needs T approval**); access/decay/importance are pure computation in `scripts/lib/memory_metrics.py`. Migration is **local-only** (`memory/` is gitignored — `python3 -m scripts.lib.memory_metrics --migrate memory/auto --apply`, never commit). Full schema + contract → `reference/phases.md` (Phase 3).

## Phase 4 Prune & Index

**Goal:** rebuild `MEMORY.md` ≤ 200 lines, drop obsolete pointers, re-rank by **importance** (#711 MEM-S2 Stage 1).

> Stage 1 changes only the **ranking key** (relevance → importance). The line cap stays 200 and the sudden-shrink guard is unchanged; raising the cap to 300 + tag-cluster hierarchy is **Stage 3** (out of this dispatch's scope).

**Steps:**

1. Read current `MEMORY.md`.
2. For each entry, `Bash: ls` to confirm the topic file still exists. Missing → delete candidate.
3. **Re-rank by importance descending** (`severity × access_freq × decay` from each memory's front-matter, computed via `scripts/lib/memory_metrics.py`). Importance subsumes the old `last-modified recency + signal strength` heuristic: recency is captured by `decay` (days since last access) and signal strength by `access_count`. A memory with high recent access (e.g. `feedback_bg_agent_limit.md`) sorts above a rarely-read one of equal severity. Files lacking valid front-matter fall back to the default severity (3=MED) and are not dropped (fail-soft).
4. Overflow > 200 lines: demote the **lowest-importance** entries into topic-file body and keep only a summary line in the index.
5. **Sudden-shrink guard:** if the new line count is < 50% of the old, **auto abort**. apply-mode also skips the commit. (Unchanged by #711.)

### Retrieval scoring (recall-time, #711 MEM-S2 Stage 1 — helper only)

Recall-time surface order = **recency × importance × relevance** (recency = exp decay of access age; relevance = query-overlap proxy in [0,1], no extra LLM), via the deterministic helper `scripts.lib.memory_metrics.retrieval_score(...)`. Phase 4's importance re-rank bakes the *importance* ordering into the index. **Stage 1 ships the helper + unit tests only**; wiring it into the live recall path is Stage 2+ (out of scope).

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
- **Each R-S must include an external-research-derived recommendation section** (see step 3 below). This is mandatory for R-S only — Phase 7 R-SH is excluded from external research because it is a deterministic structural check.

**RET-S1 policy (#493, 2026-06-06 onward) — applied to R-S, R-Self-*, R-SH-T-required jointly:**
- **1/day TOTAL T-required cap** across Phase 6 (R-S) + Phase 7 (R-SH T-required) + Phase 8 (R-Self-Internal / R-Self-External). Per-axis cap (previously `R-S = 1/day per axis`) is REPLACED by this single total cap. Counter file: `cache/retro_proposal_today.json` (`{"date","t_required_count","filed_issue_numbers"}`). Atomic write (temp + os.replace). See `scripts/lib/retrospective_lint.py`.
- **Single chosen approach** — issue body must present exactly ONE solution. No `## Approach comparison`, no "Option A/B/C", no "Phase A + Phase B" split. Alternatives may appear as a 1–2 line footnote only.
- **fail-first (no fallback)** — `## Edge cases (fail-first)` section must propose `raise` / `log_err` / stop. No `fallback` / `graceful degradation` / `legacy mode` / `silent skip` patterns. Counter file corruption → raise (do NOT silently rewrite).

**Procedure (summary):** pre-flight cap check → pick 3–5 candidate frictions → **per-candidate external research** (generic sources only, ≥ 1 required) → pick the single highest-impact one (1/day cap) → append to `dream-skill-suggestions-YYYY-MM-DD.md` → pre-filing lint (`retrospective_lint.lint_body`) → file one GitHub issue (`status:action_required,type:story`, title `R-S<N> (YYYY-MM-DD): <skill> — <one-line change>`) → bump `cache/retro_proposal_today.json` → back-write issue number.

**dry-run:** file the markdown only with `(dry-run)` marker; do NOT file issues (external research may still run + record). **apply:** file markdown + issue (with `## 推奨対策案 (外部リサーチ)` section) + bump counter + back-write.

Full step-by-step + external-research quality bar + R-S issue body template → `reference/phases.md` (Phase 6).

## Phase 7 Skill Health Check

Phase 7 has **two parallel sub-phases**: the original **mechanical structural check** (Phase 7-Mech) and the new **content quality consolidation** (Phase 7-Content, #543 DREAM-S1). Both feed into the same R-SH adoption loop and share the unified `cache/retro_proposal_today.json` cap.

### Phase 7-Mech (existing)

**Goal:** scan `.claude/skills/*/SKILL.md` for structure anomalies and emit one **R-SH GitHub issue** per detection. Phase 7-Mech does **NOT** re-derive health by reasoning — it calls the shared engine `python3 -m scripts.lib.skill_health .claude/skills/<name>` (the SAME engine the creation-time pre-commit linter uses, single-source). Body sizing is **token-primary** (tiktoken local estimate or `len/4` fallback; warn ≈ 8000 tok, error ≈ 16000 tok). The engine's `check` value maps 1:1 to the auto-execute eligibility table in `reference/phases.md`.

**RET-S1 policy (#493, 2026-06-06 onward) — R-SH split into 2 buckets:**
- **auto-execute eligible** (TOC missing / frontmatter > 200 chars / SKILL.md body over warn/error token threshold): Butler may auto-fix in BG. **NOT counted** toward the `cache/retro_proposal_today.json` 1/day cap (zero T burden). Mark issue body with `auto_execute_eligible: true`.
- **T-required** (nested reference / guard / logic refactor / stray files / anything outside the auto-execute table): counted toward the 1/day TOTAL cap (shared with Phase 6 R-S and Phase 8 R-Self-*). Mark issue body with `auto_execute_eligible: false`.

**Procedure (summary):** run the `skill_health` engine over every skill → classify each finding auto vs T-required per the eligibility table → pre-flight cap check (T-required only; auto-eligible files regardless) → generate compression diff → pre-filing lint (`retrospective_lint.lint_body`) → file one R-SH issue per finding (`status:action_required,type:dev`) → bump counter for T-required only → for each `auto_execute_eligible: true` finding, enqueue into the consumer's auto-execute queue (`scripts.lib.auto_execute_queue enqueue`, category `frontmatter_compress` / `toc_missing` / `body_split`; polled by a Stop-hook retry checker on Stop events, dispatched into the consumer session when the normal slot is free). Maintainer-required accepts → main session applies; auto-eligible dispatch with no user gating.

Full engine call + eligibility table + step-by-step + enqueue detail + R-SH issue body template → `reference/phases.md` (Phase 7).

### Phase 7-Content (new #543 DREAM-S1)

**Goal:** detect content-quality decay (duplicates / hierarchy violations / bloat) in `<consumer-skill>/reference/principal.md` (consumer-specific, optional) + `memory/auto/feedback_*.md`, propose consolidation, and (when the safety gates pass) auto-merge / auto-move so the upper-level files (principal.md) stay principle-level while concrete examples live in the lower-level memory.

**Scope (T-approved 2026-06-08, #543):**

- IN scope: `<consumer-skill>/reference/principal.md` (consumer-specific, optional) + `memory/auto/feedback_*.md`
- OUT of scope: SKILL.md / reference/*.md / CLAUDE.md (those stay under Phase 7-Mech)

**Detection (deterministic, lib: `scripts/lib/content_quality_check.py`):**

| Pattern | Signal | Auto-execute gate |
|---------|--------|-------------------|
| **memory duplicate** | token-set ratio > 0.7 across `memory/auto/feedback_*.md` pairs | ratio > 0.9 AND LLM confidence > 0.85 AND consecutive 2-run high (`cache/content_quality_decisions.json`) |
| **principal.md hierarchy violation** | `**Examples**` / `**NG パターン**` block whose token-set ratio with an existing feedback memory body > 0.7 | LLM confidence > 0.85 AND consecutive 2-run high |
| **bloat warning** | memory body > 200 lines / principal.md > 250 lines | **never auto** — warning only, T 判断 |

**Safety gates (mandatory):**

1. **First-run forced dry-run** — `cache/content_quality_first_run.flag` が **無い間は全提案 dry-run 強制** (`apply_proposal` 内で `dry_run=False` 渡しても強制 dry-run になり `forced_dry_run: true` を返す)。 T が初回 dry-run 結果を review → flag 作成で本番 auto-execute 解禁。
2. **2-pass decision cache** — `cache/content_quality_decisions.json` に `{pair_key: [{timestamp, confidence}, ...]}` を保存。 連続 2 回 ≥ 0.85 を満たさない限り auto-execute しない。 corruption → raise (silent reset 禁止、 fail-first)。
3. **Diff persistence** — auto / dry-run 問わず `apply_proposal` は事前に `cache/content_quality_dryrun/YYYY-MM-DD/<proposal_id>.diff` を書く。 復元手段確保。
4. **Auto-execute boundary** — auto category = (a) memory merge: drop file 物理削除 + representative に append + MEMORY.md index 更新、 (b) principal.md hierarchy move: 該当 block を `**Label**: see <memory-note>` 1 行に圧縮 + 元 block 本文を candidate memory に append。 それ以外は user_judgment (maintainer accept/defer 判断)。

**Steps (Phase 7 execute 時 7-Mech の直後に走らせる):**

1. `from scripts.lib.content_quality_check import (detect_memory_dups, detect_principal_bloat, detect_bloat_warnings, propose_consolidation, apply_proposal)` を import
2. `files = sorted(Path("memory/auto").glob("feedback_*.md"))`
3. `dups = detect_memory_dups(files)`
4. `bloat = detect_principal_bloat(Path("<consumer-skill>/reference/principal.md"))`
5. `warnings = detect_bloat_warnings()` — 行数 threshold 超のみ (auto 対象外、 R-SH-Content-Warning として記録)
6. `proposals = propose_consolidation(dups, bloat)` — 内部で `cache/content_quality_decisions.json` を bump、 category を auto / t_judgment に仕分け
7. 各 proposal について `apply_proposal(p, dry_run=False)` を呼ぶ。 first-run flag 無ければ自動 dry-run に降格 (`forced_dry_run: true` 確認)
8. auto-applied 分は R-SH-Content adoption に commit (`memory/auto/MEMORY.md` の merged file 削除 entry も同時更新)
9. t_judgment 分は `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` に `R-SH-Content-<N>` 行として append、 unified cap `cache/retro_proposal_today.json` を 1 消費 (Phase 7-Mech T-required と同じ枠)
10. 初回 dry-run の結果 JSON を `cache/content_quality_dryrun/YYYY-MM-DD/initial.json` に保存、 Issue #543 にコメント (本番化承認待ち)

**Cap interaction:**
- auto category → unified counter 不消費 (Phase 7-Mech auto と同じ扱い、 T burden ゼロ)
- t_judgment category → unified counter を 1 消費 (Phase 6 R-S / Phase 7-Mech T-required / Phase 8 R-Self-* と共有)

**Dry-run vs apply:**
- **dry-run mode**: detect + propose + diff 保存のみ。 file 物理変更しない、 Issue 起票しない
- **apply mode**: auto category 実行 (file 変更 + MEMORY.md update)、 t_judgment は dream-skill-suggestions-*.md に書き出し
- **first-run flag 無い間は apply mode 渡しても dry-run 降格** (再掲)

**Pushover通知:** auto-executed 件数 > 0 で priority:0 通知 (T 把握用)。 t_judgment 件数 > 0 は dream-skill-suggestions の通常通知に乗る。

**Observability log line:** `[r-sh-content] detected_dups=N detected_bloat=M auto_executed=X proposed=Y deferred=Z first_run_flag=<bool>`

Lib API contract / threshold tunable は `scripts/lib/content_quality_check.py` の docstring 参照。

## Phase 8 Self-Improvement Synthesis

**Goal:** distill long-running self-improvement candidates that are **larger than any single Phase 6 R-S** by synthesizing across the entire Phase 2 signal (internal) and across generic external sources (external). Phase 8 candidates share the **1/day TOTAL T-required cap** with Phase 6 R-S and Phase 7 R-SH-T-required (#493 RET-S1, 2026-06-06 onward).

Phase 8 is **shared infrastructure** with Phase 6 (no new cron / no new lib); the two phases differ only in scope:

| Aspect | Phase 6 R-S | Phase 8 R-Self |
|--------|-------------|----------------|
| Source | single observed friction in Phase 2 | aggregated pattern across many sessions / sources |
| Count per day | shares 1/day TOTAL T-required cap | shares 1/day TOTAL T-required cap |
| Origin axis | internal (skill / agent friction) only | **internal** *and* **external (generic sources)** |
| External research | mandatory per-issue | mandatory per-issue (same generic-source rules) |
| Duplicate check | optional | **mandatory** (`scripts/lib/self_improvement_dedup.py`) |
| Quality bar | 1 source minimum | **≥ 2 sources** + dedup + concrete target |

**RET-S1 policy reminder (#493):** Phase 8 obeys the same 3 rules as Phase 6 — single chosen approach (no `## Approach comparison`), fail-first edge cases (no `fallback` / `graceful degradation` / `legacy mode`), and 1/day TOTAL across R-S + R-Self-* + R-SH-T-required via `cache/retro_proposal_today.json`. The legacy per-axis `internal_count` / `external_count` in `cache/self_improvement_today_count.json` remains for back-compat bookkeeping but the **hard cap is enforced via the unified counter**.

### Internal vs external (generic)

**R-Self-Internal** (origin: own session history):
- Repeated patterns in daily / activity logs
- Closed task / Issue history showing latency or judgement friction
- Commit history with revert / rework loops
- Retrospective entries deferred ≥ 3 cycles

**R-Self-External** (origin: outside the project):
- Trending items from a consumer-side trend-watch skill, **if one exists** (consumer-specific; phase reads any output the consumer surfaces under `memory/shared/notes/` and treats it as one generic source)
- Technical community blog / public newsletter trending items
- Official source (LLM provider / cloud provider / relevant SDK) updates that directly map to an existing implementation

**Hard rule (AC13/AC14):** when describing external sources in this skill's documentation, **never** name concrete sites, services, internal skills, repos, or persons — use generic category labels only ("technical community blog", "public newsletter", "official SDK documentation", "web search engine"). Concrete source names belong only in consumer-side reference files outside this skill.

### Procedure (summary)

Pre-flight unified cap check (`cache/retro_proposal_today.json`; `t_required_count >= 1` → skip + "明日に持越") → internal extraction (max 1, concretely localizable + ≥ 2 observations + non-duplicate) → external extraction (max 1, generic sources, maps directly to current implementation) → per-candidate external research → **duplicate detection** (`scripts/lib/self_improvement_dedup.py` vs open issues + `feedback_*.md` + 90-day `retrospective-adoptions.md`) → quality bar (≥ 2 sources / remediation non-`N/A` / target to skill·lib·line / scope ≤ 1 week) → pre-filing lint → **pick exactly one** (1/day TOTAL cap) → file issue (`status:action_required,type:story`, title `R-Self-Internal-N` / `R-Self-External-N (YYYY-MM-DD): <area> — <one-line change>`) → bump unified counter (+ legacy `cache/self_improvement_today_count.json` for back-compat reporting only) → append to `dream-skill-suggestions-YYYY-MM-DD.md`.

**dry-run:** all steps run (incl. external research + dedup) but no issue filed, cap cache not bumped, markdown `(dry-run)`. **apply:** issues filed + cap cache bumped + markdown annotated with filed issue numbers.

Full step-by-step + R-Self issue body template + dedup lib contract → `reference/phases.md` (Phase 8).

Full template + dedup library contract → `reference/phases.md`.

## Mode behaviour summary

| Operation | dry-run | apply |
|-----------|---------|-------|
| Phase 1–2 reads | run | run |
| Phase 3 diff build | in-memory | run + present |
| topic Edit | no | yes |
| MEMORY.md rewrite | no | yes |
| dream-suggestions-*.md (R-D) | yes | yes |
| Phase 5 marker-zone rewrite | no (header-only create allowed) | yes |
| dream-skill-suggestions-*.md (R-S / R-Self-*) | yes (with dry-run marker) | yes |
| Phase 6 issue filing | no | yes |
| Phase 7 scan | run (detect-only) | run (file R-SH) |
| Phase 8 extraction + dedup + research | run (detect-only) | run |
| Phase 8 issue filing (R-Self-Internal / R-Self-External) | no | yes |
| `cache/self_improvement_today_count.json` bump | no | yes |
| `cache/retro_proposal_today.json` bump (#493) | no | yes |
| Pre-filing lint (`retrospective_lint.lint_body`) | run (detect-only, no abort) | run (raise SystemExit(1) on violation) |
| git commit | no | yes |
| `cache/dream_last_at` touch | no | yes |

## apply-mode post-processing

1. Write current epoch to `cache/dream_last_at` — **only here, only after all phases (1–8) have completed successfully in apply mode**. This is the single touch site. Consumer skills (butler / leader / etc.) and orchestration scripts MUST treat `cache/dream_last_at` as **read-only** in their cache_check paths (no pre-emptive / race-protection touch). Touching the cache before completion would cause an unfinished dream to be misclassified as "done" and the next 24h gate to skip a needed run.
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
Phase 6 Self-Improvement: <summary (R-S: N filed | skipped — "明日に持越" if cap reached)>
Phase 7 Skill Health Check: <summary (R-SH auto-eligible: N filed, R-SH T-required: N filed | skipped)>
Phase 8 Self-Improvement Synthesis: <summary (R-Self filed: 0|1, dedup-blocked: M | skipped — cap reached)>

RET-S1 cap state (#493): t_required_count=<N>/1 (axis-skipped: <list>)
Pre-filing lint violations (aborted issues): <N>

R-D suggestions: N -> memory/shared/notes/dream-suggestions-YYYY-MM-DD.md
R-S suggestions: N -> memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md
R-SH suggestions: N (same file)
R-Self-Internal: N (same file) / R-Self-External: N (same file)
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

- `reference/phases.md` — Phase 1–8 detail (steps, templates, grep patterns, locale variants, lib contracts, post-processing)
- `reference/outcome.md` — R-N / R-D / R-S / R-SH 3-layer outcome loop relationship
