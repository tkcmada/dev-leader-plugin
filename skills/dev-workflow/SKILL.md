---
name: dev-workflow
description: type:story / type:epic 6-stage flow (file → refinement → AC gate → approval → implement → QA → close), plus optional consumer Fast Lane. Loaded on demand by a consumer skill. See body for details.
---

# dev-workflow — development task standard flow

This skill defines how a `type:story` issue is processed end-to-end, including the brainstorming → AC → approval → implementation → QA → close pipeline. It is loaded on demand by a consumer skill when a `type:story` / `type:epic` issue is encountered. The description was compressed to <200 chars — the full trigger list (when this skill applies vs. when the consumer routes elsewhere) lives in [When to use this skill](#when-to-use-this-skill) below.

All paths in this doc are **repo-relative**. The GitHub repo, push branch, and project root are resolved by the consumer at runtime (`gh` / `git remote` / `git rev-parse --show-toplevel`), not hard-coded.

Issues labeled `type:story` are **always** processed by the 6-stage flow defined below.

## TOC

**In this file:**

- [Ticket hierarchy: epic / story](#ticket-hierarchy-epic--story)
- [When to use this skill](#when-to-use-this-skill)
- [6-stage flow](#6-stage-flow) (summary + link)
- [Good vs bad AC examples](#good-vs-bad-ac-examples)
- [Options principle (full)](#options-principle-full)
- [User Story — bad vs good example](#user-story--bad-vs-good-example)
- [Commit conventions (full table)](#commit-conventions-full-table)
- [Self-improvement pipeline (generic)](#self-improvement-pipeline-generic) (summary + link)

**Reference files (detail split out for context economy):**

- [`reference/six-stage-flow.md`](reference/six-stage-flow.md) — full stage-by-stage procedure: [2a] single-gate refinement, [2b] AC (6-viewpoint checklist), [3] AC Gate, [3.5] approval, [4] implementation, [5]/[5e] QA + E2E gate, [6] close
- [`reference/self-improvement.md`](reference/self-improvement.md) — self-improvement pipeline detail + export-time name-leak lint
- [`templates/refinement-drafts.md`](templates/refinement-drafts.md) — refinement draft boilerplate (EN + JA)

## Ticket hierarchy: epic / story

```
type:story (default: 1 ticket = multiple sub-stories aggregated) ──(6-stage flow [1]–[6])──▶ implement & close

type:epic (only when user explicitly requests epic) ──(epic to story)──▶ type:story × N ──(6-stage flow)──▶ ...
```

- **`type:story` is the only implementation-unit ticket.** Single small tasks are also filed as `type:story`, not as a separate "dev" type. (Some consumers may add a separate fast-lane type — see the consumer's own SKILL.md.)
- **Default policy: aggregate multiple sub-stories into a single `type:story` ticket** rather than splitting them into separate tickets. Each sub-story carries its own AC under a single ticket; AC Gate review and approval happen **once per ticket**. After approval, all sub-stories are dispatched together.
- **Splitting into multiple story tickets** (or creating an epic + child stories) is done **only when the user explicitly requests it** (e.g. "epic にして" / "split into stories" / "add epic ticket"). Do not auto-split.
- A story can exist under an epic (when explicitly created) or on its own (default).
- The epic itself is **not** an implementation unit and does not go through the 6-stage flow. Close the epic once all child stories are `status:done`.
- **"add epic ticket"** / **"epic to story"** are recognition phrases handled by the consumer skill **only when the user explicitly invokes them** (file `type:epic` / refine → spawn child stories). Each child story then goes through the single-gate refinement defined in [\[2a\] Refinement single-gate document-completion flow](reference/six-stage-flow.md#2a-refinement-single-gate-document-completion-flow) (8 mandatory sections: Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure scenario / AC).
- When child stories are explicitly requested, slice them as **user-facing vertical slices** (e.g. "add logging feature"), not by component (e.g. "module A only"). Link the epic number from the story body.

## When to use this skill

A GitHub Issue qualifies for this workflow if it does any of the following:

- Writing / modifying code
- Creating or significantly modifying skills (SKILL.md) or agent definitions
- Adding / changing external tool / service integration
- Modifying existing automation flows
- Investigating / fixing bugs

Issues that do not qualify (routine task management, ops notes, etc.) are handled by the normal consumer flow.

---

## 6-stage flow

```
[1] File a ticket
   ↓
[2] Refinement (decide AC)        ← [2a] brainstorm + [2b] write AC
   ↓
[3] AC Gate check                  ← if fails, back to [2]
   ↓ pass
[3.5] User approval                ← brainstormed tasks only; also grants commit/push permission
   ↓ approved
[4] Automated implementation (background agent)
   ↓
[5] QA (consumer verifies each AC) ← includes [5e] mandatory E2E gate
   ↓
[6] Close the ticket
```

The **full stage-by-stage procedure** lives in [`reference/six-stage-flow.md`](reference/six-stage-flow.md) and is the authoritative detail for every stage. It covers, in order:

- **[1] File a ticket** — `gh issue create` with `status:open` + `type:story`; required body sections; AC may be deferred to [2].
- **[2] Refinement** — runs the **[2a] single-gate document-completion flow** (8 mandatory sections: Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure / AC; 3 input sources User/Codebase/Net; 6-item self-check; brainstorm only on insufficiency; Issue body = latest draft, comments = feedback; AskUserQuestion brainstorming-only policy) then **[2b] write Acceptance Criteria** (6-viewpoint AC checklist — see `reference/six-stage-flow.md` §[2b]).
- **[3] AC Gate check** — all gate items must pass or return to [2].
- **[3.5] User approval** — brainstormed tasks only; approval doubles as commit/push permission. Slot-busy queue state = **`status:ac_approved`** (not `status:blocked`, which is reserved for true external dependencies, #505).
- **[4] Automated implementation** — background agent brief + BG code-slot cap (consumer-specific).
- **[5] QA** — consumer verifies each AC, plus the **[5e] mandatory E2E gate** for web / UI / pipeline / voice changes (real-path Playwright + screenshot evidence, deploy-before-E2E for live services, E2E passes before `status:user_confirming`).
- **[6] Close the ticket** — diff review, commit with `#N`, direct push, label → `status:done`, close.

Read `reference/six-stage-flow.md` before driving any stage.

---

## Good vs bad AC examples

**Bad:**
```
- [ ] Fix the bug
- [ ] Improve it
- [ ] Make performance better
```

**Good:**
```
- [ ] `ssh <host> hostname` returns `<host>` without prompting for a password
- [ ] `~/.ssh/config` contains a `Host <host>` entry with HostName/User/IdentityFile set
- [ ] `python3 test.py --duration 180` keeps register A=0 and register B=0 for 3 minutes
```

---

## Options principle (full)

Brainstorming questions must **always offer 2–4 concrete options**. Each option has a `label` (short headline) and a `description` (trade-off).

**Why:**
- Open-ended questions ("how do you want to implement this?") push the design space back to the user.
- Options make the consumer's design proposals visible — the user just accepts or rejects.
- If you have a recommendation, place it first and append `(recommended)` to its label.

**Option label convention:** Use **uppercase Latin letters** `A` / `B` / `C` / `D` / `E` (e.g. `Option A`). **Never use Greek letters** (`α` / `β` / `γ` / `δ` / `ε`) — they are hard to type and read aloud.

**Good example:**

> Q: What polling interval?
> - 10 ms (recommended) — good balance of latency and load
> - 5 ms — higher responsiveness, more CPU
> - 20 ms — lower load, slight peripheral delay
> - Event-driven — large refactor, out of scope this time

**Bad example:**

> Q: How many ms?
> (forces the user to pick a number; trade-offs are invisible)

**Exceptions:**
- For free-form numeric or date values where options are hard to enumerate, present 3–4 representative options plus a "free input" fallback.
- If the user explicitly says "you decide" or "your call", do not ask — the consumer picks the best option and proceeds.

---

## User Story — bad vs good example

The User Story slot in the refinement document (see `templates/refinement-drafts.md`) is mandatory. Pattern: `As <User>, from <Where>, when <When>, I do <What>, expecting <Expected>.`

Bad:
> "Save settings"
> (Who? What problem does it solve?)

Good:
> As an operator, from the CLI, when I change a device config, I do `device set-config`, expecting the new config to survive unintended resets without manual reconfiguration.

---

## Commit conventions (full table)

| Item | Policy |
|------|--------|
| Granularity | One issue = one (or a few) commits |
| Message | English, subject + body, includes `#N` |
| Push | Direct push to the current branch (no PR workflow) |
| Approval gate | **[3.5] user approval doubles as commit/push permission** |
| Agent diff review | The consumer always runs `git diff` before committing |
| On failure | Do not amend existing commits — create a new commit |
| Exception | If the user said "I will commit myself" at approval time, stop just before commit in [6] and present the diff |

## Commit message example

```
Fix #234: Persist settings in non-volatile storage

Settings currently live only in RAM and are wiped on unintended resets,
forcing manual reconfiguration mid-run. Persist them with a magic byte
for invalidation and reload on boot.

Verified: settings survive 5 forced resets.
```

---

## Self-improvement pipeline (generic)

This skill ships an **optional, opt-in** self-improvement pipeline: a consumer's memory-consolidation skill (when one exists) can propose improvements to the consumer's own skills / agents and surface them through the same 6-stage flow above. A consumer without such a skill simply omits these steps.

The pipeline distinguishes two suggestion origins (**Internal** = patterns observed inside the consumer project; **External** = generic external sources) so limits and quality bars can be tuned separately, each capped at 1 issue / day.

The full pipeline — source-naming hard rule, per-issue quality bar, per-day cap counter, standup surfacing, issue body template, consumer-specific extension boundary, and the **export-time name-leak lint** (`scripts/lint_skill_export.sh`, wired into pre-commit) — lives in [`reference/self-improvement.md`](reference/self-improvement.md). It is **not duplicated here** ([[single-source]]).

---

## References

- [`reference/six-stage-flow.md`](reference/six-stage-flow.md) — full 6-stage procedure (stages [1]–[6], including [2a] single-gate refinement, [2b] AC with the 6-viewpoint checklist, [3.5] approval, and the [5e] mandatory E2E gate)
- [`reference/self-improvement.md`](reference/self-improvement.md) — optional self-improvement pipeline + export-time name-leak lint
- [`templates/refinement-drafts.md`](templates/refinement-drafts.md) — refinement draft boilerplate
