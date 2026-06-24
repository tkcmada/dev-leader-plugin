# dev-workflow — Self-improvement pipeline (generic)

Optional opt-in self-improvement pipeline. Split out of `SKILL.md` for context economy ([#686] R-SH1 body hygiene). Content is unchanged. Includes the quality bar, per-day cap, standup surfacing, issue body template, consumer-specific extension boundary, and the export-time name-leak lint. The parent `SKILL.md` carries a summary + link.

## TOC

- [Self-improvement pipeline (generic)](#self-improvement-pipeline-generic)
- [Quality bar (per-issue)](#quality-bar-per-issue)
- [Per-day cap](#per-day-cap)
- [Surface in standup](#surface-in-standup)
- [Issue body template](#issue-body-template)
- [Consumer-specific extensions](#consumer-specific-extensions)
- [Export-time verification](#export-time-verification)

## Self-improvement pipeline (generic)

This skill ships an optional self-improvement pipeline that lets the consumer's memory-consolidation skill (when one exists) propose improvements to the consumer's own skills / agents and surface them through the same 6-stage flow above. The pipeline is **opt-in** — a consumer that does not yet have a memory-consolidation skill simply omits these steps.

The pipeline distinguishes two suggestion origins so that limits and quality bars can be tuned separately:

| Origin | Description (generic) | Per-day cap |
|--------|-----------------------|-------------|
| Internal | Patterns observed inside the consumer project (daily notes, session logs, closed-issue history, recurring friction) | 1 issue / day |
| External | Patterns sourced from generic external sources — web search engines, technical community blogs, public newsletters, official SDK / framework documentation | 1 issue / day |

**Hard rule (source naming):** when documenting this pipeline inside any skill text (including this SKILL.md), refer to external sources **only by generic category** — for example "web search engine result", "technical community blog post", "public newsletter article", "official SDK documentation page". Do **not** name concrete sites, services, individual repositories, internal skills, persons, or notification channels. Concrete names belong only in the consumer's private reference files outside this skill.

### Quality bar (per-issue)

A proposed self-improvement issue is filed only when **all** of the following hold:

1. The target is identified to skill / library / file / line granularity.
2. There are **≥ 2 independent supporting sources**:
   - Internal: ≥ 2 distinct observations in daily notes, session logs, or closed issues.
   - External: ≥ 2 generic external sources, **or** 1 official documentation page plus 1 supporting community / newsletter article.
3. The issue body includes a `## 推奨対策案 (外部リサーチ)` (or equivalent locale) section summarizing the best-practice remediation. The section lists at least one generic external source category — never a concrete name.
4. A duplicate check against existing open issues, the consumer's persistent feedback notes, and the last 90 days of the consumer's retrospective adoption table returns **no match ≥ 0.80** on a normalized similarity score (token-set ratio preferred, sequence-matcher fallback). The duplicate-detection library is shared with the broader retrospective pipeline; no new cron or per-pipeline detector is added.
5. Implementation scope is estimated **≤ 1 week**.

### Per-day cap

A cap counter file (e.g. `cache/self_improvement_today_count.json`) tracks the number of internal and external issues filed today. Pre-flight reads the counter, skips extraction for any axis that has already reached the cap, and on a date mismatch resets both counters atomically. Update is done via temp-file rename so a crash mid-write does not corrupt the counter.

```json
{
  "date": "YYYY-MM-DD",
  "internal_count": 0,
  "external_count": 0,
  "filed_issue_numbers": []
}
```

### Surface in standup

Pending self-improvement issues are surfaced in the consumer's standup (or daily review) inside the **existing retrospective adoption loop** — no new IF is created. The standup section header is, conventionally, `## 自己改善提案` (or `## Self-improvement proposals` in English locales). Each pending row carries:

- The issue prefix (`R-Self-Internal-N` or `R-Self-External-N` — see prefix table in the consumer's memory-consolidation reference).
- The target area (skill / library) and one-line proposed change.
- The filed issue link.
- Adjudication interface (same as the consumer's other retrospective rows): accept / reject / defer, with 7-day defer resurfacing and 3-defer auto-reject escalation.

On accept, the issue routes through this skill's 6-stage flow exactly like any other `type:story` (with the optional Fast Lane when the change is small and the consumer supports a fast-lane label).

### Issue body template

```markdown
## Proposal (R-Self-<Internal|External>-<N>)

- **Origin axis**: internal | external
- **Target area**: <skill / lib / line — concrete>
- **Observed pattern (or external signal)**: <1–3 lines>
- **Proposed change**: <what / how>
- **Estimated impact**: <UX / latency / failure rate / dev velocity>
- **Estimated cost**: small / medium / large (≤ 1 week scope)

## 推奨対策案 (外部リサーチ)

- 推奨アプローチ: <1–3 lines summarizing best-practice>
- 根拠 source カテゴリ: <generic category — e.g. "official SDK documentation" / "technical community blog" / "public newsletter" / "web search engine">
- 補強 source 数: <N (≥ 2 required)>
- 代替案 (任意): <alternative approach, if any>
- 注意点: <trade-offs / conflicts with current implementation>

## Duplicate detection

- Checked against: open issues / persistent feedback notes / retrospective adoption rows (last 90 days)
- Similarity threshold: 0.80 (token-set ratio; sequence-matcher fallback)
- Result: no match above threshold (top match: `<title>` @ <score>)

## Source

- Generated: YYYY-MM-DD
- Evidence (internal): <≥ 2 internal references>
- Evidence (external): <≥ 2 generic source categories>

## Next action

Owner: consumer
Action: surfaced in standup `## 自己改善提案` section. Accept → enter this skill's 6-stage flow. Reject → close. Defer → `status:blocked` (resurfaces in 7 days).
```

### Consumer-specific extensions

Consumer-specific specifics — for example which concrete trend-watch source the consumer subscribes to, which notification channel surfaces the proposal, which paths hold the daily notes — belong in a consumer-side reference file (typically `reference/self-improvement-<consumer>.md`) and are **never** baked into this SKILL.md. This keeps the skill clone-and-reusable across projects with zero placeholder substitution.

### Export-time verification

Consumers that publish this skill upstream should run a name-leak lint over this file before pushing. A reference lint script (`scripts/lint_skill_export.sh`) is provided in the same repository; it `grep`s a fixed deny-list (concrete site names, internal skill names, repository owners, personal names, notification-channel IDs, environment-variable values) and fails with a non-zero exit code if any match is found. The lint is wired into the consumer's pre-commit hook so an unintended leak blocks the commit.
