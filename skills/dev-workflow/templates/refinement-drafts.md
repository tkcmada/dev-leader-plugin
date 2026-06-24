# Refinement draft template ([2a] single-gate document-completion flow)

This file ships the **single integrated draft boilerplate** that dev-workflow's [2a] Refinement single-gate document-completion flow consumes. Copy the whole block as the starting point — do not improvise the wording, because the wording is what enforces the mandatory TOC and the single-gate stop condition.

The single-gate flow itself, the input gathering / self-check / brainstorming procedure, the Issue body overwrite procedure, the feedback comment read logic, and the AskUserQuestion usage policy live in the parent SKILL.md (`../SKILL.md` → `[2a] Refinement single-gate document-completion flow`). This file is **boilerplate only**.

## TOC

- [How to use](#how-to-use)
- [Integrated draft boilerplate (EN)](#integrated-draft-boilerplate-en)
- [Integrated draft boilerplate (JA)](#integrated-draft-boilerplate-ja)
- [Mandatory vs optional sections](#mandatory-vs-optional-sections)
- [User Story slot guide](#user-story-slot-guide)
- [User Story Japanese examples](#user-story-japanese-examples)
- [Architecture options sub-block (optional detail form)](#architecture-options-sub-block-optional-detail-form)
- [Stop condition](#stop-condition)

---

## How to use

1. **Gather inputs from 3 sources** — combine **User / Codebase / Net** (see [Input sources](#input-sources-3-categories) below). Read Issue body + comments + memory `feedback_*.md` (User); Read related files + `Explore` + related Issues + git history (Codebase); `WebSearch` / `WebFetch` / `deep-research` (Net, only when vendor / API / best-practice info is genuinely needed). Pick the source(s) per the [combination judgment table](#combination-judgment-table) below. Do this proactively.
2. **Self-check** — run the **6-item insufficiency check** from SKILL.md ([2a] step 2). If **all 6 are YES**, skip to step 4. If **any is NO**, go to step 3 (brainstorming) or back to step 1 for more inputs.
3. **Brainstorming (only when inputs are insufficient)** — ask T via chat or `AskUserQuestion` to fill the missing inputs. Apply the 4 question-quality checks from SKILL.md `AskUserQuestion usage policy (brainstorming-only)`. Skip this step entirely when the 6-item self-check passes.
4. **Write the draft in one shot** — copy this boilerplate and fill **all 8 mandatory sections** (Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure scenario / Acceptance Criteria) plus any optional sections relevant to the ticket (関連 Issue / etc.). Overwrite the Issue body via `gh issue edit <N> --body-file <tmp>`. **Even simple tickets must fill Analysis / Architecture / Use case with at least 1 paragraph (or 3 steps) of real content** — `N/A` / placeholder is not acceptable (#582 AC13).
5. **Prompt for feedback** — tell the user in chat: "最新版を Issue body に反映しました。修正があればコメントで指摘してください。OK なら次へ進みます。"
6. **Iterate** — at the start of each iteration, run `gh issue view <N> --comments` and process new comments. Update the draft, overwrite the body again. Repeat 5–6 until step 7 fires.
7. **Single-gate approval** — the user explicitly says `OK / 進めて / 同意 / yes` in chat or as a comment. Then proceed to [3] AC Gate → [3.5] approval → [4] implementation.

**AskUserQuestion**: skip it for normal feedback-loop iteration (text + body update + comment is sufficient). Use it **only** in step 3 brainstorming — most commonly for an architecture option choice — and only after the 4 question-quality checks are satisfied. See SKILL.md `AskUserQuestion usage policy (brainstorming-only)`.

---

## Input sources (3 categories)

The skill combines **3 input source categories** at step 1. Pick which to use based on the question type (see [combination judgment table](#combination-judgment-table) below).

| # | Source | What it covers | Tools |
|---|--------|----------------|-------|
| **1** | **User input** | Issue body / 既存コメント + memory `[[feedback_*]]` / `[[project_*]]` + 過去 daily / decisions / notes + T's chat utterances | `gh issue view <N> --comments` / Read memory / chat |
| **2** | **Codebase** | Related files, related Issues / PRs, git history, test coverage, module structure | `Read` (file path 既知) / `Explore` agent (file path 未知) / `gh issue view <N>` / `gh pr view <N>` / `git log` |
| **3** | **Net (external)** | Official docs, API specs, vendor pages, best-practice articles; heavyweight broad → narrow → deep research | `WebSearch` / `WebFetch` / `Skill(skill="deep-research")` |

## Combination judgment table

Which source to consult for which question type (T amendment 2026-06-10, #582):

| Question type | Primary source | Secondary source |
|---------------|----------------|------------------|
| Existing implementation behavior / interface / file path | Codebase | User (past design decision) |
| T preference / family context / subjective judgment | User | — |
| External API spec / library doc / vendor specification | Net | Codebase (current import) |
| Best practice / industry trend / state of the art | Net (deep-research) | User (T's domain experience) |
| AC / acceptance-criteria concreteness | Codebase + User (T expectation) | Net (e.g. testing best practice) |
| Architecture option trade-off | Codebase (constraints) + Net (technology) + User (preference) | — |

**Proposal simplicity:** call only the source(s) you actually need — do not over-research. But do not hesitate to confirm spec / vendor / library when the answer is needed for AC.

---

## Integrated draft boilerplate (EN)

All 8 sections below are **mandatory** (#582 AC13). Simple tickets may use shorter content (Analysis ≥ 1 paragraph, Use case ≥ 3 steps, Architecture "1 adopted + rejected reasons") but `N/A` / placeholder is not allowed.

```markdown
## Goal

<1–3 sentence summary — what problem, for whom, observable outcome>

## Non-goal

- <explicitly out of scope this iteration — prevents scope creep>
- <…>

## Analysis

(Current-state understanding. Mandatory — at least 1 paragraph for simple tickets.)

### 既存実装
- `<path/to/file>:<symbol>` — <one-line summary of what it does today>
- `<path/to/file>:<symbol>` — <one-line summary>

### 制約
- `<memory-note>` — <rule that applies>
- `CLAUDE.md` <section> — <rule that applies>
- Prior T directive (YYYY-MM-DD) — <rule that applies>
- (For simple tickets with no special constraints, write "no special constraints apply" — placeholder / `N/A` is not allowed.)

### 関連 Issue
- Parent epic: #<N> <title>
- Sibling stories: #<N>, #<N>
- depends-on / supersedes: #<N>

### 過去 attempts
- #<N> (closed YYYY-MM-DD) — <why it closed / what was learned>
- (none found, if applicable)

### 未知
- <bullet 1 — something the read pass could not answer; will be resolved via T confirmation>
- <bullet 2>

## User Story

As <User>,
from <Where>,
when <When>,
I do <What>,
expecting <Expected>.

## Use case

(Concrete walkthrough — Golden path step 1-N + Alternate paths. Mandatory — simple tickets may use as few as 3 steps but `N/A` is not allowed.)

### Golden path
1. <User action 1>
2. <System response 1>
3. <User action 2>
4. <System response 2>
5. <Observable success outcome>

### Alternate paths
- <Alternate 1 — short label>: <when this branches off> → <expected outcome>
- <Alternate 2>: <when> → <outcome>

## Architecture

(Option enumeration + scope context + comparison axis. Mandatory — simple tickets may use "1 adopted option + rejected-option reasons" but `N/A` is not allowed. See [Architecture options sub-block](#architecture-options-sub-block-optional-detail-form) below for the per-option boilerplate.)

- **Adopted: Option A — <short label>**
  - What: <1 sentence — what this option does concretely>
  - Why adopted: <1–2 sentences — what we gain on the comparison axis>
- **Rejected: Option B — <short label>** (when applicable)
  - Why rejected: <1–2 sentences — what we give up>
- (For trivial tickets: "1 adopted option, no alternatives considered because <reason>" is acceptable.)

## Failure scenarios

- **<Failure 1 — short label>**: <when this happens> → <expected behavior / fallback>
- **<Failure 2>**: <when> → <fallback>
- **<Failure 3>** (optional): <when> → <fallback>

## Acceptance Criteria

- [ ] AC1 — <concrete done-condition with verify procedure>
- [ ] AC2 — <…>
- [ ] AC3 — <…>
```

---

## Integrated draft boilerplate (JA)

下記 8 section はすべて **mandatory** (#582 AC13)。 簡単な ticket では短く書いてよい (Analysis 最低 1 段落 / Use case 3 step / Architecture 「採用案 1 + 棄却理由」) が、 `N/A` / placeholder は不可。

```markdown
## ゴール

<1〜3 文で要約。誰の何の問題を、どんな観測可能な結果にするか>

## 含まないこと (Non-goal)

- <この iteration の scope 外 — scope creep 防止>
- <…>

## 現状理解・分析 (Analysis)

(現状理解。 必須 — 簡単な ticket でも最低 1 段落書く。)

### 既存実装
- `<path/to/file>:<symbol>` — <1 行で何をしているか>
- `<path/to/file>:<symbol>` — <1 行で>

### 制約
- `<memory-note>` — <適用ルール>
- `CLAUDE.md` <section> — <適用ルール>
- 過去 T 指示 (YYYY-MM-DD) — <適用ルール>
- (特殊な制約がない簡単な ticket では「現状特殊な制約なし」 と明記。 placeholder / `N/A` は不可。)

### 関連 Issue
- 親 epic: #<N> <title>
- sibling story: #<N>, #<N>
- depends-on / supersedes: #<N>

### 過去 attempts
- #<N> (YYYY-MM-DD close) — <なぜ close / 何が学べたか>
- (該当なし、の場合はそう書く)

### 未知
- <未知 1 — 現状理解で答えられないこと。T 確認に回す>
- <未知 2>

## User Story

As <User>,
from <Where>,
when <When>,
I do <What>,
expecting <Expected>.

## ユースケース (Use case)

(具体的 walkthrough — Golden path step 1-N + Alternate paths。 必須 — 簡単な ticket でも最低 3 step、 `N/A` は不可。)

### Golden path
1. <ユーザ操作 1>
2. <システム応答 1>
3. <ユーザ操作 2>
4. <システム応答 2>
5. <観測可能な成功結果>

### Alternate paths
- <代替 1 — ラベル>: <分岐条件> → <期待結果>
- <代替 2>: <分岐条件> → <期待結果>

## Architecture

(設計選択肢 option 列挙 + scope context + 比較軸。 必須 — 簡単な ticket では「採用案 1 つ + 他案棄却理由」 で OK、 `N/A` は不可。 per-option boilerplate は [Architecture options sub-block](#architecture-options-sub-block-optional-detail-form) 参照。)

- **採用: Option A — <ラベル>**
  - What: <1 文 — この選択肢が何をするか>
  - 採用理由: <1-2 文 — 比較軸でどう優位か>
- **棄却: Option B — <ラベル>** (該当時)
  - 棄却理由: <1-2 文 — 何が劣るか>
- (簡単な ticket では「採用案 1 つ、 他案検討せず (理由: <理由>)」 で可。)

## 失敗シナリオ

- **<失敗 1 — ラベル>**: <発生条件> → <期待挙動 / フォールバック>
- **<失敗 2>**: <発生条件> → <フォールバック>
- **<失敗 3>** (任意): <発生条件> → <フォールバック>

## Acceptance Criteria

- [ ] AC1 — <具体 done-condition + verify 手順>
- [ ] AC2 — <…>
- [ ] AC3 — <…>
```

---

## Mandatory vs optional sections

The 8 sections below are **mandatory** — the skill must fill all of them in every refinement document (#582 AC13, T directive 2026-06-10 final). Simple tickets may use shorter content for Analysis / Use case / Architecture (≥ 1 paragraph or ≥ 3 steps) but `N/A` / placeholder is not allowed.

| # | Section | Purpose | Simple-ticket minimum |
|---|---------|---------|----------------------|
| 1 | **Goal** | 1–3 sentence summary — what problem, for whom, observable outcome. | 1 sentence |
| 2 | **Non-goal** | Explicitly out of scope this iteration. Prevents scope creep. | ≥ 1 bullet |
| 3 | **Analysis** (current-state understanding) | Existing implementation / constraints / related Issue / past attempts / unknowns. | ≥ 1 paragraph (e.g. "no special constraints apply"); `N/A` not allowed |
| 4 | **User Story** | Abstract contract: `As <User>, from <Where>, when <When>, I do <What>, expecting <Expected>.` | 1 sentence (all 5 slots filled) |
| 5 | **Use case** (golden / alternate) | Concrete walkthrough — Golden path step 1-N + Alternate paths. | ≥ 3 steps; `N/A` not allowed |
| 6 | **Architecture** | Option enumeration + scope context + comparison axis. | "1 adopted option + rejected-option reasons" is OK; `N/A` not allowed |
| 7 | **Failure scenarios** | What can go wrong and the expected fallback. | ≥ 1 failure scenario |
| 8 | **Acceptance Criteria** | Concrete done-conditions with verify procedure. | ≥ 1 verifiable AC |

**Optional** (include only when relevant to the ticket):

| Section | Include when |
|---------|--------------|
| **関連 Issue** (as a separate top-level section) | Usually folded into Analysis › 関連 Issue subsection. Pull out only when the relation list is very long. |
| **etc.** | Anything else this ticket specifically needs. |

### Why Analysis / Use case / Architecture became mandatory (T directive 2026-06-10 final)

- **Analysis mandatory**: 現状理解なしに正しい AC は書けない。 制約 / 過去 attempt / 関連 Issue を踏まえる必要がある。
- **Use case mandatory**: flow 仕様の sequencing / 副作用 / alternate path を可視化しないと implementation 時に混乱しやすい。 簡単な ticket でも 3 step で済む。
- **Architecture mandatory**: 設計選択肢を文書化しないと implementation の合理性 / 代替案検討が後追いできない。 簡単な ticket でも「採用案 1 つ + 他案棄却理由」 で済む。

---

## User Story slot guide

The 5 slots are **mandatory**. Do not omit any slot, even if it feels obvious — the user must see and agree to each.

| Slot | What goes here |
|------|----------------|
| `<User>` | The role / actor (family member name, operator, BG agent, external caller, …). |
| `<Where>` | The entry point / surface (voice / web UI / iPhone Safari / cron / GitHub Issue comment / …). |
| `<When>` | The triggering condition (wake word detected / scheduled cron tick / user clicks button / …). |
| `<What>` | The action the user takes (or the system performs on their behalf). |
| `<Expected>` | The observable outcome that signals success. |

---

## User Story Japanese examples

### Example 1 — Voice intent

```
As an end user,
from a voice session (real-time speech API),
when 定型フレーズを発話したとき,
I do 対応する skill を起動して定型アクションを実行する,
expecting 結果を音声で読み上げ → ユーザーが「はい」と返したときのみ確定送信し、確定後に処理結果を音声報告する.
```

### Example 2 — Web dashboard

```
As an end user,
from モバイルブラウザで開いた Web Dashboard の「今日の概要」タブ,
when 朝の時間帯に最初に開いたとき,
I do 当日のスケジュール + 補助判定 + 天気 を 1 画面で確認する,
expecting 各データソース (カレンダー API / 判定 skill / weather skill) の結果が同じ画面に並んでいて、 タップせずに俯瞰できる.
```

### Example 3 — BG agent contract

```
As the leader skill (dev-workflow 経由で起動),
from BG general-purpose agent slot (normal),
when AC 全項目を実装し終わって commit + push を完了したとき,
I do Issue にコメント + completion marker を残して終了する,
expecting consumer 側の `<task-notification status=completed>` handler が完了検証ルーチンを呼び、 recommendation に従って `status:done` / `status:user_confirming` / `status:action_required` に auto-transition する.
```

---

## Architecture options sub-block (optional detail form)

The Architecture section itself is **mandatory** (#582 AC13) — even trivial tickets must document the adopted option + rejected-option reasons. This sub-block is the **optional detailed form**: use it when a design decision actively blocks AC and benefits from per-option Scope context / Comparison axis / Trade-off bullets. For simple tickets the inline "Adopted / Rejected" bullets in the mandatory Architecture section above are sufficient. The 4 question-quality checks from SKILL.md `AskUserQuestion usage policy (brainstorming-only)` apply: (1) inputs gathered, (2) trade-off per option, (3) scope context printed, (4) comparison axis explicit.

```markdown
### Architecture — Q<N>: <one-line question>

**Scope context**

<1 paragraph: current implementation (file path / function), the constraint that forces this question, related Issue numbers, what is already decided and out of scope. ~3–6 sentences.>

**Comparison axis**

<one phrase: e.g. "implementation cost vs. extensibility" / "speed vs. certainty" / "migration risk vs. clean rewrite">

**Options**

- **Option A — <short label>** (recommended)
  - What: <1 sentence — what this option does concretely>
  - Trade-off: <1–2 sentences — what we gain on the axis above and what we give up>

- **Option B — <short label>**
  - What: <1 sentence>
  - Trade-off: <1–2 sentences>

- **Option C — <short label>** (optional)
  - What: <1 sentence>
  - Trade-off: <1–2 sentences>
```

### Common axes (pick the one that fits)

| Axis | When to use |
|------|-------------|
| Speed vs. certainty | Realtime / latency-sensitive vs. correctness-sensitive choices |
| Implementation cost vs. extensibility | Quick fix vs. long-term framework |
| Migration risk vs. clean rewrite | Touching a running service with state vs. green-field replacement |
| Coupling vs. autonomy | Shared lib vs. per-consumer copy |
| Auto-execution vs. T confirmation | Confidence / blast-radius of an automated action |
| Single-source vs. duplication | Where the same rule appears in multiple skills |

### AskUserQuestion (brainstorming-only)

If you use `AskUserQuestion` to surface the option choice (recommended for a clean structured pick), apply the 4 quality checks from SKILL.md first. For the normal feedback-loop iteration on the 8 mandatory sections (Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure / AC), do **not** use `AskUserQuestion` — chat text + Issue body overwrite + user feedback comment is the standard loop.

---

## Stop condition

| Stop signal | Action |
|-------------|--------|
| User explicitly says `OK / 進めて / 同意 / yes / 合ってる` in chat or as an Issue comment | Single-gate approved. Proceed to [3] AC Gate → [3.5] approval → [4] implementation. |
| Silence / off-topic reply / counter-question | **Not approval.** Continue iterating: incorporate any feedback, overwrite the body, re-prompt. |
| User comments edits / additional requirements | Apply them, overwrite the body, re-prompt. |

Do not advance past the gate without the explicit approval signal.
