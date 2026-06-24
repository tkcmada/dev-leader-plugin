# Phase details — Phase 1–8 step-by-step

This reference expands the Phase 1–8 detail beyond what SKILL.md contains. SKILL.md is the canonical entry; this file is loaded on demand when an operator wants the full step-by-step.

## TOC

- [Phase 1 Orient — full steps](#phase-1-orient--full-steps)
- [Phase 2 Gather Signal — full procedure](#phase-2-gather-signal--full-procedure)
- [Phase 3 Consolidate — merge invariants](#phase-3-consolidate--merge-invariants)
- [Phase 4 Prune & Index — pruning rules + R-D emission](#phase-4-prune--index--pruning-rules--r-d-emission)
- [Phase 5 Daily Log Update — section rendering](#phase-5-daily-log-update--section-rendering)
- [Phase 6 Self-Improvement — R-S issue body template](#phase-6-self-improvement--r-s-issue-body-template)
- [Phase 7 Skill Health Check — R-SH issue body template](#phase-7-skill-health-check--r-sh-issue-body-template)
- [Phase 8 Self-Improvement Synthesis — R-Self body template + dedup lib contract](#phase-8-self-improvement-synthesis--r-self-body-template--dedup-lib-contract)

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
   ls -t .claude-voice/projects/*/*.jsonl 2>/dev/null | head -10
   ls -t knowledge/voice-logs/*.jsonl 2>/dev/null | head -10
   ```
2. Run each grep pattern as a separate `Bash` call.
3. Read the most recent 3 daily files per actor (`memory/users/*/daily/*.md`) and per dev (`memory/dev/daily/*.md`).
4. Aggregate into **5–10 candidate topics** (defer the rest).
5. Separately collect today's JSONL entries (all sources) — these feed Phase 5.

### Memory access counting (#711 MEM-S2 Stage 1)

The JSONL session logs are already open here, so Phase 2 also computes per-memory **access_count** for the importance signal. This is **deterministic and atime-free** — the host mounts `/` with `noatime` (verified `findmnt -no OPTIONS /`), so file atime is never updated on read and is unusable; the genuine retrieval signal is the `Read` tool_use `file_path` argument recorded in each JSONL line.

Call the shared lib (do NOT re-implement the parse):

```bash
python3 -m scripts.lib.memory_metrics --window 14
```

Contract (`scripts/lib/memory_metrics.py`):

- `scan_access_counts(jsonl_paths, *, window_days=14, now=None, access_tools={"Read"})` → `{memory_basename: AccessStats(count, last_accessed)}`.
- Each JSONL line has a top-level ISO-8601 `"timestamp"` and `message.content[].type == "tool_use"` with `input.file_path`. A line counts iff its timestamp is within `window_days` of `now` AND the tool is in `access_tools` (default `{"Read"}` — `Edit`/`Write` are mutations, not accesses, so excluded) AND `file_path` contains `memory/auto/`.
- **Fail-soft at the line level:** corrupt / unparseable / timestamp-less lines are skipped, the scan continues. A missing JSONL file is skipped (the scan does not abort).
- `last_accessed` = the most recent in-window access date (`YYYY-MM-DD`), `''` if never accessed in-window.
- CLI `--window N` / `--jsonl <file>` (repeatable) / `--now <iso>` (for reproducible output) / `--migrate <dir> [--apply]` (front-matter migration, local-only).

Empirical sanity check (14-day window ending 2026-06-20, live tree): only `feedback_bg_agent_limit.md` had an in-window `Read` (1, 2026-06-13). Over 90 days: `MEMORY.md` 3, `feedback_bg_agent_limit.md` 1, `feedback_dream_auto_execute.md` 1. Explicit memory `Read`s are sparse because memory is mostly auto-loaded into the system prompt — that is the honest deterministic signal; the access_freq floor keeps unaccessed memories orderable by severity.

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

### Importance front-matter maintenance (#711 MEM-S2 Stage 1)

Phase 3 refreshes the importance metrics in each `memory/auto/*.md` front-matter (`metadata:` mapping), using the Phase 2 access counts:

```yaml
metadata:
  severity: 1 | 3 | 10        # 1=LOW / 3=MED / 10=HIGH (default 3=MED)
  access_count: <int>         # 14-day-window Read count (Phase 2)
  last_accessed: YYYY-MM-DD    # most recent in-window Read ('' if never)
  importance: <float>          # severity × access_freq × decay
  tags: [<str>, ...]           # Stage 2 populates; [] allowed in Stage 1
```

- **importance = severity × access_freq × decay**; `access_freq = access_count / window_days` floored at ε (so unaccessed memories still order by severity); `decay = exp(−λ·days_since_last_access)`, λ≈0.05 (half-life ≈14d). All via `scripts.lib.memory_metrics.importance()`.
- **severity scale {1=LOW / 3=MED / 10=HIGH}** (discrete, #711 T 2026-06-20; numeric in front-matter so the scale can gain levels later). The LLM assigns severity at memory-creation / consolidation time. **Lowering an existing memory's severity requires T approval** (`feedback_standard_change_needs_t`) — Phase 3 may set the default (3) for a new / unset memory but must not autonomously demote an existing value. access/decay/importance are pure computation (no T gate).
- **Migration is local-only.** `memory/` is gitignored; migrate existing memories with `python3 -m scripts.lib.memory_metrics --migrate memory/auto --apply` and **never commit** the result. Migration is idempotent and preserves the body + any human-set severity/tags.

## Phase 4 Prune & Index — pruning rules + R-D emission

**Goal:** rebuild `MEMORY.md` ≤ 200 lines, drop obsolete pointers, re-rank by **importance** (#711 MEM-S2 Stage 1; line cap stays 200 in Stage 1 — Stage 3 raises it to 300 + adds hierarchy).

### Pruning steps

1. Read current `MEMORY.md`.
2. For each entry, `Bash: ls` to confirm the topic file still exists. Missing → delete candidate.
3. **Re-rank by importance descending** (`severity × access_freq × decay` from front-matter via `scripts/lib/memory_metrics.py`). This replaces the old `last-modified recency + signal strength` key: `decay` encodes recency (days since last access) and `access_count` encodes signal strength. Memories with no valid front-matter use the default severity (3=MED) and are NOT dropped (fail-soft).
4. Overflow > 200 lines: demote the **lowest-importance** entries into topic-file body and keep only a summary line in the index.

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

## Phase 6 Self-Improvement — procedure + R-S issue body template

### Procedure (full steps)

1. **Pre-flight cap check.** Read `cache/retro_proposal_today.json` via `scripts.lib.retrospective_lint.load_counter` (fail-loud on corruption). If `t_required_count >= 1`, skip Phase 6 entirely + record "(明日に持越)" in the Phase 6 report and the standup `## 自己改善提案` marker. Do NOT queue / accumulate.
2. Pick 3–5 candidate frictions targeting a specific skill / agent.
3. For each, capture: observed pattern / proposed change / target skill name / estimated impact / estimated cost (small / medium / large) / target SKILL.md section.
4. **External research (R-S only, mandatory):** for each candidate, consult **generic external sources** (e.g. web search engines, technical community blogs, official documentation of the relevant SDK / framework, public newsletters) to identify the best-practice remediation for the observed friction. **Do not name specific sites, services, or internal skills in the SKILL.md text** — refer to them only by generic category. Capture at least one independent source and a one-paragraph summary of the recommended approach. The summary becomes the `## 推奨対策案 (外部リサーチ)` section of the filed issue body (template below).
5. Pick the **single highest-impact candidate** for filing today (the 1/day cap means only one will be filed regardless). Discard the others (or defer naturally — they will resurface as friction next run if still relevant).
6. Append to `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md`:

   ```markdown
   # Dream Skill Suggestions YYYY-MM-DD

   - R-S1: <skill> / <pattern> / <proposal> / effort=small|medium|large / external_sources=N / #<issue>
   ```

7. **Pre-filing lint.** Before `gh issue create`, run `scripts.lib.retrospective_lint.lint_body(body)` on the rendered body. Forbidden substrings (`option a/b/c`, `fallback`, `graceful degradation`, `legacy mode`, `phase a + phase b`) or the `## Approach comparison` header → `raise SystemExit(1)` (do NOT file).
8. File the R-S as a single GitHub issue (`gh issue create --label "status:action_required,type:story"`).
   - Title: `R-S<N> (YYYY-MM-DD): <skill> — <one-line change>`
   - Body: see "R-S issue body template" below (single chosen + fail-first edge cases + 推奨対策案).
9. Bump counter (`scripts.lib.retrospective_lint.bump_counter`) with the filed issue number.
10. Back-write the filed issue number into the R-S list line.

**External-research quality bar (R-S only):**
- ≥ 1 generic external source (web search result, technical community article, official documentation, newsletter, etc.) consulted; ideally ≥ 2 for cross-confirmation.
- The recommended approach is captured as 1–3 bullet points in `## 推奨対策案 (外部リサーチ)` of the issue body.
- Source category is named generically (e.g. "official SDK docs", "technical community blog", "web search engine") — **no concrete site/service names** in skill text.
- If no external source yields a credible remediation, write `- N/A (no external source found)` and downgrade the candidate to `effort=large` for human review.

**dry-run:** file the markdown only with `(dry-run)` marker. **Do not** file issues. External research may still be performed and recorded in the dry-run markdown.
**apply:** file markdown + issues (with `## 推奨対策案 (外部リサーチ)` body section) + back-write.

### R-S issue body template

**RET-S1 policy (#493):** body must have exactly ONE solution in `## Chosen approach`. NO `## Approach comparison` table, NO Option A/B/C, NO Phase A + Phase B split. `## Edge cases (fail-first)` must propose raise / log_err / stop — NOT `fallback`, `graceful degradation`, or `legacy mode`. Pre-filing lint (`scripts.lib.retrospective_lint.lint_body`) enforces this; violations abort the filing.

```markdown
auto_execute_eligible: false

## Proposal (Dream Phase 6 R-S<N>)

- **Target skill**: <skill> (`.claude/skills/<name>/SKILL.md`)
- **Section**: <SKILL.md heading>
- **Observed pattern**: <specific>
- **Estimated impact**: <UX / latency / failure rate>
- **Estimated cost**: small / medium / large

## Chosen approach

<!--
Single solution only. No comparison table, no A/B/C, no Phase A + Phase B.
If a serious alternative exists, mention it in a 1-line footnote only.
-->

<1–3 paragraphs describing the single chosen change: what edit / where / how>

(任意脚注) 代替: <1 行で代替案、 採用しなかった理由を 1 行>

## Edge cases (fail-first)

<!--
fail-loud only. NO fallback / graceful degradation / legacy mode / silent skip.
On failure: raise, log_err, or halt the operation.
-->

- <ケース 1>: raise <Exception> / log_err + exit (no fallback)
- <ケース 2>: 失敗時は動作停止 (silent retry / cache 返却 / 旧 prompt 経路は禁止)

## 推奨対策案 (外部リサーチ)

<!--
R-S only (R-SH excluded). Populated from Phase 6 external research step.
Cite source categories generically (e.g. "official SDK documentation",
"technical community blog post", "web search engine result", "public newsletter").
Do not name concrete sites, services, internal skills, or repos.
-->

- 推奨アプローチ: <1–3 行で best-practice 要約>
- 根拠 source カテゴリ: <e.g. "official SDK documentation" / "technical community blog" / "web search engine" — generic only>
- 補強 source 数: <N (≥ 1 required, ≥ 2 recommended)>
- 注意点: <導入時の trade-off / 既存実装との不整合があれば 1 行>

## Source

- Generated: YYYY-MM-DD (Dream Phase 6)
- dream-skill-suggestions: [memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md](...)
- Evidence: <daily / JSONL excerpt>

## Next action

Owner: maintainer
Action: review the chosen approach + 推奨対策案; if accepted, ask the main session to edit the SKILL.md. If rejected, close the issue.
```

## Phase 7 Skill Health Check — R-SH issue body template

**RET-S1 policy (#493):** body must start with `auto_execute_eligible: true|false`. `true` rows skip the 1/day cap (Butler may BG-auto-fix); `false` rows count toward the shared 1/day TOTAL T-required cap with Phase 6/8. Pre-filing lint (`scripts.lib.retrospective_lint.lint_body`) checks the body for `option a/b/c`, `fallback`, `graceful degradation`, `legacy mode`, `## Approach comparison` — any hit aborts the filing.

### Detection scan — call the shared engine (single-source, #575)

Phase 7 does **NOT** re-derive structural health by reasoning. It calls the shared
detection engine `scripts/lib/skill_health.py` (the SAME engine the creation-time
pre-commit linter uses — single-source principle: one engine, no duplicate logic). For each
skill, run:

```bash
python3 -m scripts.lib.skill_health .claude/skills/<name>
```

The CLI prints a JSON array of findings (`{check, severity, message, current_value, limit}`)
and exits 1 when ≥1 ERROR. File one R-SH issue per finding using the table below to
set `auto_execute_eligible`. The `check` value returned by the engine maps 1:1 to the
rows here. Body sizing is **token-primary** (tiktoken `cl100k_base` local estimate, or
`len/4` fallback — NO API call); thresholds calibrated against current skills (warn
≈ 8000 tok, error ≈ 16000 tok; see the engine docstring for derivation). Skills with
no frontmatter, and externally-vendored skills that are their own nested git repo, are
skipped by the engine.

### Procedure (full steps, Phase 7-Mech)

1. Read every SKILL.md. Compute line count + frontmatter length + reference link inventory (the shared engine does this; see the `skill_health` call above).
2. For each `reference/*.md`, check for TOC (`## TOC` or equivalent within the first 30 lines).
3. Grep each `reference/*.md` for inter-reference links to detect nesting.
4. For each detection, classify as `auto_execute_eligible: true|false` per the eligibility table below.
5. **Pre-flight cap check for T-required.** Read `cache/retro_proposal_today.json` via `scripts.lib.retrospective_lint.load_counter` (fail-loud on corruption). If `t_required_count >= 1`, skip the **T-required** R-SH filings + mark "(明日に持越)" in report. Auto-eligible R-SH continue to file regardless of the counter.
6. For each detection that will be filed, generate a compression diff proposal.
7. **Pre-filing lint.** For each rendered body, run `scripts.lib.retrospective_lint.lint_body`. Violation → `raise SystemExit(1)`. The body must include the `auto_execute_eligible: true|false` marker at the top.
8. File one R-SH issue per detection (labels `status:action_required`, `type:dev`).
9. After each T-required filing, bump `cache/retro_proposal_today.json` via `scripts.lib.retrospective_lint.bump_counter`. Auto-eligible filings do NOT bump.
10. **For each `auto_execute_eligible: true` filing**, immediately enqueue into Butler's auto-execute queue (#494) — see "Auto-execute enqueue" below.
11. User accepts T-required → main session applies the diff (commit + push). Rejects → close issue. Auto-eligible items are dispatched automatically per step 10; no user gating.

### Auto-execute eligibility table (check name = engine `check` value, 1:1)

| `check` (engine) | severity | Detection | auto_execute_eligible | auto-fix category / counter |
|------------------|----------|-----------|----------------------|------------------------------|
| `body_too_large` | ERROR | SKILL.md body > error token threshold (~16000 tok) | `true` | `body_split` — NOT counted (Butler auto-fix) |
| `body_large` | WARNING | SKILL.md body > warn token threshold (~8000 tok) | `true` | `body_split` — NOT counted |
| `reference_no_toc` | WARNING | reference/*.md > 100 lines without TOC | `true` | `toc_missing` — NOT counted |
| `frontmatter_description_hard_limit` | ERROR | frontmatter `description` > 1024 chars (official hard limit) | `true` | `frontmatter_compress` — NOT counted (auto-compress) |
| `frontmatter_description_soft_limit` | WARNING | frontmatter `description` > 200 chars (Butler soft convention) | `true` | `frontmatter_compress` — NOT counted |
| `stray_root_md` | WARNING | non-standard root `.md` (move under reference/) | `false` | counted toward 1/day TOTAL |
| `frontmatter_name_too_long` | ERROR | frontmatter `name` > 64 chars (official hard limit) | `false` | T-required — counted toward 1/day TOTAL |
| `stray_file` | ERROR | non-allowlisted root file (non-`.md`, e.g. `notes.txt`) | `false` | T-required — counted toward 1/day TOTAL |
| `stray_dir` | ERROR | non-allowlisted subdir (e.g. `references/` typo) | `false` | T-required — counted toward 1/day TOTAL |
| `reference_nested` | ERROR | nested reference (depth > 1 under reference/) | `false` | T-required — counted toward 1/day TOTAL |
| (guard / logic refactor / anything else) | — | reasoning-based, outside the engine | `false` | counted toward 1/day TOTAL |

### Template

```markdown
auto_execute_eligible: <true|false>

## Proposal (Dream Phase 7 R-SH<N> Skill Health Check)

- **Target skill**: <skill> (`.claude/skills/<name>/SKILL.md`)
- **Detection**: <engine `check` value — e.g. body_too_large | body_large | reference_no_toc | frontmatter_description_hard_limit | frontmatter_description_soft_limit | frontmatter_name_too_long | stray_file | stray_dir | stray_root_md | reference_nested | guard / logic>
- **Current value**: e.g. SKILL.md body ~9061 tok (warn 8000 / error 16000) — taken from the engine finding's `current_value` / `limit`
- **Estimated impact**: SKILL.md X → Y lines / token Z% reduction
- **Reference target**: `.claude/skills/<name>/reference/<topic>.md`

## Chosen approach

<!-- Single solution only. Compression diff for auto-eligible; concrete edit plan for T-required. -->

<for auto-eligible structural fixes:>

  \`\`\`diff
  - ## Long section (XX lines)
  - ...
  + ## Long section (5 lines)
  + Details: see `reference/<topic>.md`
  \`\`\`

<for T-required logic/guard refactors:>

<1–3 paragraphs describing the single chosen change.>

## Edge cases (fail-first)

<!-- raise / log_err / halt. No fallback / graceful degradation / legacy mode. -->

- <ケース 1>: 例 — diff 適用後 TOC 検証 fail → raise + skip commit
- <ケース 2>: 例 — reference Read 失敗 → log_err + 即時中断 (旧本文に戻さない)

## Source

- Generated: YYYY-MM-DD (Dream Phase 7)
- Detection scan: `dream-skill-suggestions-YYYY-MM-DD.md` row R-SH<N>

## Next action

Owner: maintainer (auto-eligible: Butler BG may dispatch automatically per feedback_dream_auto_execute)
Action: review the chosen approach; accept → apply (commit + push). Reject → close.
```

### Auto-execute enqueue (#494, applies to `auto_execute_eligible: true`)

After filing each auto-eligible R-SH issue, dream Phase 7 immediately enqueues it into Butler's auto-execute queue (`cache/auto_execute_queue.json`):

```bash
python3 -m scripts.lib.auto_execute_queue enqueue <issue_number> <category> "<full_agent_prompt>" "<issue_title>"
```

`<category>` ∈ `frontmatter_compress` / `toc_missing` / `body_split` (matches the auto-eligibility table).

Enqueue failures are silent log (stderr only, fail-first retrospective policy — no fallback). A Stop-hook retry checker (event-driven; e.g. a Claude Code Stop hook) polls the queue, checks BG slot status for normal-cap free capacity, and injects `[auto-execute] issue=<N> ...` into the consumer session via the inject scheduler when a slot is available. The consumer skill picks up the inject, acquires the slot, dispatches the Agent, and marks dispatched/completed in the queue.

The old trigger ("scan on next `hi butler`") was removed by #494 so auto-eligible fixes proceed without waiting for T to wake the session.

## Phase 8 Self-Improvement Synthesis — procedure + R-Self body template + dedup lib contract

### Procedure (full steps)

1. **Pre-flight cap check (unified, #493).** Read `cache/retro_proposal_today.json` via `scripts.lib.retrospective_lint.load_counter` (fail-loud on corruption). If `t_required_count >= 1`, log "Phase 8 cap reached (shared with Phase 6/7); skipping" + mark "(明日に持越)" and return. Otherwise, also read `cache/self_improvement_today_count.json` for back-compat per-axis bookkeeping (legacy).
2. **Internal extraction (max 1):** scan the aggregated Phase 2 signal (daily logs / JSONL / closed-issue history within the protect-list-respecting read scope). Pick the single highest-impact pattern that:
   - is concretely localizable (a specific skill / lib / line / behaviour),
   - has ≥ 2 independent supporting observations,
   - does not duplicate an existing open issue / feedback memory entry / a row in `retrospective-adoptions.md`.
3. **External extraction (max 1):** consult generic external sources (web search engine / technical community blog / public newsletter / official SDK documentation, etc.) for an improvement that maps directly to the consumer project's current implementation. Same dedup constraints as step 2.
4. **External research per candidate.** Each surviving candidate has at least one external source consulted and the best-practice remediation written into the issue body's `## 推奨対策案 (外部リサーチ)` section (same template as Phase 6 R-S).
5. **Duplicate detection.** Run `scripts/lib/self_improvement_dedup.py` (or the project's equivalent) against open issues, `memory/auto/feedback_*.md`, and the last 90 days of `retrospective-adoptions.md` rows. Reject any candidate above the similarity threshold; log the matched item.
6. **Quality bar.** Each candidate must satisfy:
   - ≥ 2 independent sources (≥ 2 internal observations for R-Self-Internal; ≥ 2 generic external sources OR ≥ 1 official source + 1 supporting article for R-Self-External),
   - external-research remediation present and non-`N/A`,
   - target identified to skill / lib / line granularity,
   - implementation scope estimated ≤ 1 week.
7. **Pre-filing lint (#493).** For each rendered body, run `scripts.lib.retrospective_lint.lint_body`. Violations (`option a/b/c`, `fallback`, `graceful degradation`, `legacy mode`, `phase a + phase b`, `## Approach comparison` header) → `raise SystemExit(1)`.
8. **Pick one (#493 1/day TOTAL cap).** If both an internal and an external candidate survive dedup + quality bar, file only the **single highest-impact one** (because the unified cap is 1/day total). Discard the other or defer naturally.
9. **File issue.**
   - Title: `R-Self-Internal-N (YYYY-MM-DD): <skill or lib> — <one-line change>` or `R-Self-External-N (YYYY-MM-DD): <area> — <one-line change>`
   - Labels: `status:action_required,type:story`
   - Body: see "R-Self issue body template" below (single chosen + fail-first edge cases + 推奨対策案).
10. **Update cap caches.** Bump the unified `cache/retro_proposal_today.json` via `scripts.lib.retrospective_lint.bump_counter` (atomic). Also bump the legacy `internal_count` / `external_count` in `cache/self_improvement_today_count.json` (for back-compat reporting only — the hard cap is the unified counter).
11. **Append to suggestions file.** Add lines to `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` with the `R-Self-Internal-N` / `R-Self-External-N` prefix.

**Dry-run vs apply:** dry-run runs all steps (including external research and dedup) but files no issue and does not bump the cap cache (markdown annotated `(dry-run)`); apply files issues + bumps cap cache + annotates markdown with the filed issue numbers.

### R-Self issue body template

**RET-S1 policy (#493):** body must have exactly ONE solution in `## Chosen approach`. NO `## Approach comparison`, NO Option A/B/C, NO Phase A + Phase B split. `## Edge cases (fail-first)` must propose raise / log_err / stop — NOT fallback / graceful degradation / legacy mode. Counted toward shared 1/day TOTAL cap (`cache/retro_proposal_today.json`).

```markdown
auto_execute_eligible: false

## Proposal (Dream Phase 8 R-Self-<Internal|External>-<N>)

- **Origin axis**: internal | external
- **Target area**: <skill / lib / line — concrete>
- **Observed pattern (or external signal)**: <1–3 lines>
- **Estimated impact**: <UX / latency / failure rate / dev velocity>
- **Estimated cost**: small / medium / large (≤ 1 week implementation scope)

## Chosen approach

<!--
Single solution only. No comparison table, no A/B/C, no Phase A + Phase B.
If a serious alternative exists, mention it in a 1-line footnote only.
-->

<1–3 paragraphs describing the single chosen change: what edit / where / how>

(任意脚注) 代替: <1 行で代替案、 採用しなかった理由を 1 行>

## Edge cases (fail-first)

<!--
fail-loud only. NO fallback / graceful degradation / legacy mode / silent skip.
On failure: raise, log_err, or halt the operation.
-->

- <ケース 1>: raise <Exception> / log_err + exit (no fallback)
- <ケース 2>: 失敗時は動作停止 (silent retry / cache 返却 / 旧 prompt 経路は禁止)

## 推奨対策案 (外部リサーチ)

<!--
Mandatory for both R-Self-Internal and R-Self-External.
Cite source categories generically — no concrete site/service/skill/repo/person names.
-->

- 推奨アプローチ: <1–3 行で best-practice 要約>
- 根拠 source カテゴリ: <e.g. "official SDK documentation" / "technical community blog" / "public newsletter" / "web search engine">
- 補強 source 数: <N (≥ 2 required for Phase 8)>
- 注意点: <導入時の trade-off / 既存実装との不整合があれば 1 行>

## Duplicate detection

- Checked against: open issues / `memory/auto/feedback_*.md` / `retrospective-adoptions.md` (last 90 days)
- Similarity threshold: 0.80 (rapidfuzz token_set_ratio, or difflib SequenceMatcher ratio ≥ 0.80)
- Result: no match above threshold (top match: `<title>` @ <score>)

## Source

- Generated: YYYY-MM-DD (Dream Phase 8)
- dream-skill-suggestions: [memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md](...)
- Evidence (internal): <≥ 2 daily / JSONL / closed-issue references>
- Evidence (external): <≥ 2 generic source categories>

## Next action

Owner: maintainer
Action: surfaced in standup `## 自己改善提案 (R-Self-*)`. Accept → start work (label flips). Reject → close. Defer → `status:blocked` (resurfaces in 7 days, same loop as R-N / R-D / R-S).
```

### Dedup lib contract (`scripts/lib/self_improvement_dedup.py`)

```python
def check_duplicate(
    candidate_title: str,
    candidate_body: str,
    *,
    open_issue_titles: list[str],
    feedback_memory_lines: list[str],
    retrospective_adoptions_rows: list[str],
    threshold: float = 0.80,
) -> dict:
    """
    Return shape:
      {
        "is_duplicate": bool,
        "best_match": str | None,
        "best_score": float,
        "matched_against": "open_issue" | "feedback_memory" | "retrospective_adoptions" | None,
      }
    Algorithm:
      - prefer rapidfuzz.fuzz.token_set_ratio if available (returns 0–100, normalize /100);
      - fallback to difflib.SequenceMatcher(a, b).ratio() (0.0–1.0).
      - Compare candidate_title against each haystack entry; also compare a
        normalized candidate_body excerpt (first 300 chars) against entries
        whose title hit ≥ threshold-0.10 (cheap second pass).
    """
```

### Cap counter file (`cache/self_improvement_today_count.json`) — legacy per-axis bookkeeping

```json
{
  "date": "YYYY-MM-DD",
  "internal_count": 0,
  "external_count": 0,
  "filed_issue_numbers": []
}
```

- `date` is JST today (`date +%F` in JST).
- On a date mismatch, reset both counters to 0 before pre-flight (Phase 8 step 1).
- This file is retained for back-compat reporting only. Cap enforcement is now the unified counter below (#493 RET-S1).
- Atomic write: write to `<file>.tmp` then `os.replace()`.

### Cap counter file (`cache/retro_proposal_today.json`) — unified Phase 6 / 7 / 8 cap (#493 RET-S1)

```json
{
  "date": "YYYY-MM-DD",
  "t_required_count": 0,
  "filed_issue_numbers": []
}
```

- `date` is JST today (`scripts.lib.retrospective_lint` uses `datetime.now(timezone(timedelta(hours=9)))`).
- On date mismatch, the counter returns a fresh state for today without writing to disk (`bump_counter` writes on the next file).
- **Cap**: `t_required_count >= 1` blocks ALL further T-required filings across Phase 6 R-S, Phase 7 R-SH-T-required (`auto_execute_eligible: false`), and Phase 8 R-Self-Internal / R-Self-External. Phase 7 R-SH `auto_execute_eligible: true` filings do NOT count toward this cap.
- Atomic write: temp file + `os.replace()`.
- **fail-loud**: JSON parse failure / non-dict payload / invalid schema → raise `CounterCorruptError`. No silent reset. Inspect manually and either repair or `rm` the file (which yields a fresh initial state).
- Library: `scripts/lib/retrospective_lint.py` — `load_counter()`, `can_file()`, `bump_counter()`, `lint_body()`.

### Pre-filing lint contract (`scripts.lib.retrospective_lint.lint_body`)

Called by Phase 6, Phase 7 (T-required filings), and Phase 8 BEFORE `gh issue create`.

Forbidden substrings (case-insensitive): `option a`, `option b`, `option c`, `fallback`, `graceful degradation`, `legacy mode`, `phase a + phase b`.
Forbidden section header: `## Approach comparison`.

On any match → print details to stderr and `raise SystemExit(1)`. The issue is NOT filed. dream Phase reports the violation in its summary so the operator can repair the template.

