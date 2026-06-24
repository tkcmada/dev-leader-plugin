# dev-workflow — 6-stage flow (detail)

Detailed stage-by-stage procedure for the `type:story` 6-stage flow. Split out of `SKILL.md` for context economy ([#686] R-SH1 body hygiene). Content is unchanged — this is the authoritative detail for stages [1]–[6], including [2a] single-gate refinement, [3.5] approval, and the [5e] E2E gate. The parent `SKILL.md` carries a summary + link.

## TOC

- [6-stage flow](#6-stage-flow)
- [\[2a\] Refinement single-gate document-completion flow](#2a-refinement-single-gate-document-completion-flow)
- [\[3.5\] User approval](#35-user-approval-brainstorming-tasks-only)
- [\[5\] QA + \[5e\] Mandatory E2E gate](#5-qa-ac-verification)
- [\[6\] Close the ticket](#6-close-the-ticket)

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
[5] QA (consumer verifies each AC)
   ↓
[6] Close the ticket
```

### [1] File a ticket

Create a new GitHub Issue.

- **Command**: `gh issue create --title "..." --body "..." --label "status:open,type:story"`
- **Labels**: `status:open` + `type:story`.
- **Title**: short, verb-led summary.
- **Required body sections**: `## Overview` (1–2 sentences) / `## Background` (why, related issues) / `## Next Action` (Owner: consumer / agent / user; Action: 1–2 lines).

AC may be undefined at this stage (set in [2]).

### [2] Refinement (single-gate document-completion → AC)

Runs the [2a] single-gate document-completion flow, then [2b] AC finalization.

**Aggregation default**: when refinement surfaces multiple related sub-stories, **keep them in the single existing `type:story` ticket** (list each sub-story with its own AC block under one issue body). Do **not** spawn new story tickets per sub-story. Splitting into separate stories (or promoting to an epic + child stories) is done **only when the user explicitly requests it** at refinement time. AC Gate and [3.5] approval happen once per ticket; all sub-stories are dispatched together at [4].

#### [2a] Refinement single-gate document-completion flow

**Required when**: creating a new skill/agent, effort ≥ 30 min, spans multiple files, introduces a new capability, impacts a running service, changes hardware behavior.
**Can skip when**: trivial bug fix in one function, doc/comment edits, pattern-matching refactor, renames/typos, dependency bumps, tests/logs only.

##### Goal of refinement

The goal of [2a] is for the skill to **actively write up the specified-TOC document** and obtain explicit user approval. Refinement is **not** a Q&A loop — it is a document the skill drafts proactively from the inputs it gathers, and only asks the user when those inputs are insufficient. T directive 2026-06-10 (#582, supersedes 2-gate flow #581): **single gate + proactive document completion + brainstorming only on insufficiency**. Analysis remains as one section of the body but is not its own gate.

**Mandatory sections** — 8 in total (T directive 2026-06-10 final, #582). The skill MUST fill all 8 in every refinement document:

1. **Goal** — 1–3 sentence summary: what problem, for whom, observable outcome.
2. **Non-goal** — explicitly out of scope this iteration (prevents scope creep).
3. **Analysis** — current-state understanding: existing implementation / constraints / related Issue / past attempts / unknowns.
4. **User Story** — abstract contract (1 sentence): `As <User>, from <Where>, when <When>, I do <What>, expecting <Expected>.`
5. **Use case** — concrete walkthrough (multi-step): Golden path + Alternate paths. Simple tickets may use as few as 3 steps.
6. **Architecture** — option enumeration + scope context + comparison axis. Simple tickets may use "1 adopted option + rejected-option reasons".
7. **Failure scenario / edge case** — what can go wrong and the expected fallback.
8. **Acceptance Criteria** — concrete done-conditions with verify procedure.

**Important — even simple tickets must have substance** (#582 AC13): Analysis / Architecture / Use case each need **at least 1 paragraph (or 3 steps)**. `N/A` or placeholder-only is **not allowed**. Writing "no special constraints apply" / "adopted 1 option, no alternatives considered" is OK as long as it is a real assessment, not a placeholder.

**User Story vs Use case — difference table** (T directive 2026-06-10):

| Axis | User Story | Use case |
|------|------------|----------|
| Granularity | Abstract contract (1 sentence) | Concrete walkthrough (multi-step) |
| Form | `As <U>, from <W>, when <W>, I do <W>, expecting <E>` | step 1 (user action) → step 2 (system response) → … + alternate paths |
| Focus | WHO + WHAT (expected outcome) | HOW (time order / causal chain) |
| Role | Goal definition | Flow specification |

Both are required so reviewers can check the **contract (WHAT/WHY)** and the **flow specification (HOW)** independently.

**Optional sections** (include only when not already covered by Analysis):

- **関連 Issue** (parent epic / sibling stories / depends-on / supersedes) — usually folded into Analysis
- **etc.** as the ticket needs

The full integrated boilerplate (EN + JA) lives in `templates/refinement-drafts.md`. Copy it at the start of [2a] rather than improvising the wording.

##### Input sources (3 categories) — combine actively

The skill gathers from **3 input sources** and combines them. Do not rely on a single source.

| Source | What it covers | Tools |
|--------|----------------|-------|
| **User input** | Issue body + existing comments + memory `feedback_*.md` + past daily / decisions / notes + T's chat utterances | `gh issue view` / Read memory / chat |
| **Codebase** | Related files, related Issues / PRs, git history, test coverage, module structure | `Read` / `Explore` agent / `gh issue view <N>` / `git log` |
| **Net research** | Official docs, API specs, vendor pages, best-practice articles (heavyweight = deep-research skill) | `WebSearch` / `WebFetch` / `Skill(skill="deep-research")` |

**Combination judgment table** (which source to consult for which question type):

| Question type | Primary source | Secondary source |
|---------------|----------------|------------------|
| Existing implementation behavior / interface / file path | Codebase | User (past design decision) |
| T preference / family context / subjective judgment | User | — |
| External API spec / library doc / vendor specification | Net | Codebase (current import) |
| Best practice / industry trend / state of the art | Net (deep-research) | User (T's domain experience) |
| AC / acceptance-criteria concreteness | Codebase + User (T expectation) | Net |
| Architecture option trade-off | Codebase (constraints) + Net (technology) + User (preference) | — |

**Proposal simplicity:** call only the sources you actually need — do not over-research. But do not hesitate to confirm spec / vendor / library when the answer is needed for AC.

##### Process (single gate, 7 steps)

1. **Gather inputs** — combine the **3 input sources** (User / Codebase / Net) per the table above. Read Issue body + comments + memory `feedback_*.md` (User); Read related files + Explore + related Issues + git history (Codebase); WebSearch / WebFetch / deep-research as needed (Net). The skill does this proactively without waiting for instructions.
2. **Self-check for insufficiency** — the skill verifies it has enough to draft. Run the **6-item check** (see below). If any item is `NO`, go to step 3 (brainstorming); if all `YES`, go straight to step 4 (draft).
3. **Brainstorming (only when inputs are insufficient)** — ask T via chat or `AskUserQuestion` to fill the missing inputs. Question quality bar applies (per-option trade-off 1–2 sentences, 1-paragraph scope context, explicit comparison axis). See [AskUserQuestion usage policy](#askuserquestion-usage-policy-brainstorming-only).
4. **Write the document in one shot** — fill all 8 mandatory sections + relevant optional sections, then overwrite the Issue body via `gh issue edit <N> --body-file <tmp>` (see [Issue body overwrite update + feedback comment read](#issue-body-overwrite-update--feedback-comment-read)).
5. **Prompt for feedback** — tell the user in chat: "最新版を Issue body に反映しました。修正があればコメントで指摘してください。OK なら次へ進みます。"
6. **Feedback loop** — at the start of each iteration, run `gh issue view <N> --comments` and process any comment newer than the last body update. Update the draft and overwrite the body again. Repeat 5–6 until step 7 fires.
7. **Single-gate approval** — the user explicitly says "OK / 進めて / 同意 / yes" in chat or as a comment. Then proceed to [3] AC Gate → [3.5] approval → [4] implementation.

##### Self-check: 6-item insufficiency criterion

After step 1, the skill decides whether to brainstorm (step 3) or draft (step 4) by running this **6-item check** (T directive 2026-06-10 final, #582). If **any** item is `NO`, brainstorm; if **all** are `YES`, draft.

| # | Check | YES if … |
|---|-------|----------|
| (a) | **Goal clarity** | The Goal can be summarized in 1–3 sentences from the Issue body + related material — no fundamental ambiguity about "what problem, for whom, observable outcome". |
| (b) | **User identifiable** | The `<User>` slot in the User Story can be filled (T / family member / dev / BG agent / external caller / etc.). |
| (c) | **Main use case** | At least 1–2 main use cases (golden path or core action) are evident from the gathered inputs. |
| (d) | **AC draftable** | The Acceptance Criteria can be drafted as concrete verifiable items — not just placeholders. |
| (e) | **Architecture options enumerable** | At least 1 adopted option (and ideally 1–2 rejected options with reasons) can be listed. If the ticket is trivial, "1 adopted option, no alternatives considered" is acceptable. |
| (f) | **Source-type cross-check** | For every gap identified in (a)–(e), the right input source type (User / Codebase / Net) has been consulted per the combination judgment table above. |

##### Key principles

- **Single gate only** — Analysis is one of the 8 mandatory body sections, not a separate stop condition. The user reviews the whole document and approves once. (#581 2-gate flow is retracted.)
- **Proactive document completion** — write all 8 mandatory sections in one pass instead of asking the user section-by-section.
- **Brainstorm only when inputs are insufficient** — if the 6-item self-check passes, go straight to draft; do not invent questions.
- **Issue body = latest draft** — the skill owns the body and overwrites it each iteration.
- **Comments = feedback history** — the user writes feedback in comments, leaving a chronological audit trail.
- **AskUserQuestion is for brainstorming only** — the default feedback loop is chat text + body overwrite + comment read. Use `AskUserQuestion` only when step 3 brainstorming benefits from a structured pick (e.g. architecture option choice).
- **Simple tickets still need substance** — Analysis / Architecture / Use case each need ≥ 1 paragraph (or 3 steps). `N/A` / placeholder is not acceptable.

##### AskUserQuestion usage policy (brainstorming-only)

`AskUserQuestion` is **not** the default mechanism for [2a]. The default is chat text + Issue body overwrite + comment feedback loop. The user reads the latest body, comments edits, and the skill iterates. No `AskUserQuestion` call is needed for the normal feedback loop on the 8 mandatory sections (Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure / AC) iteration.

`AskUserQuestion` **is recommended** only when step 3 brainstorming surfaces a structured choice the user benefits from picking explicitly — most commonly an architecture option choice. When you do use it, the question MUST satisfy these four checks (the question-quality bar):

1. **Inputs already gathered?** Has step 1 (gather inputs) actually been run? If not, gather first — many "missing inputs" are answered by Read / Explore / memory before asking T.
2. **Trade-off per option?** Does every option carry a 1–2 sentence trade-off? If not, add it.
3. **Scope context printed?** Has a 1-paragraph scope context (current implementation / constraints / related Issue numbers) been printed in chat or in the draft **before** the question? If not, print it first, then ask.
4. **Comparison axis explicit?** Are the options differentiated along a stated axis (speed vs. certainty / cost vs. extensibility / migration risk vs. cleanness / etc.)? If not, rewrite the option labels so the axis is visible at a glance.

These 4 checks are the **single source of truth** for brainstorming question quality in [2a]. Other skills loading dev-workflow inherit them automatically.

##### Issue body overwrite update + feedback comment read

The single-gate flow uses the Issue body as the single source of truth for the latest draft. The skill overwrites the body at every iteration; the user provides feedback in **comments** (not in body edits). This minimizes T judgment count and keeps a clear iteration history.

**Body overwrite procedure (every iteration)**:

```bash
# Write the latest draft (full mandatory + relevant optional sections,
# per templates/refinement-drafts.md) to a temp file and overwrite the body.
TMP=$(mktemp --suffix=.md)
cat > "$TMP" <<'EOF'
<paste the filled-in boilerplate from templates/refinement-drafts.md here>
EOF
gh issue edit <N> --body-file "$TMP"
rm -f "$TMP"
```

The full boilerplate (8 mandatory sections: Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure scenarios / AC; optional: 関連 Issue as a separate top-level section / etc.) lives in `templates/refinement-drafts.md` — do not duplicate it here ([[single-source]]).

**Feedback comment read (at iteration start)**:

```bash
gh issue view <N> --comments
# (JSON form for programmatic processing:)
gh issue view <N> --json comments --jq '.comments[] | {author: .author.login, createdAt, body}'
```

Apply the user's comment-feedback to the draft, then loop back to "Body overwrite procedure". Continue until the user posts an explicit "OK / 進めて / 同意 / yes" comment (or says it in chat) — that is the single-gate approval signal. Do NOT advance past the gate without that explicit signal.

**Why comments not body edits**: the skill owns the body (overwrites it), the user owns the feedback (comments). This split prevents merge conflicts and gives a chronological audit trail of how the draft evolved.

#### [2b] Write Acceptance Criteria

Take the "AC candidates" from brainstorming and detail them to the level of verification commands. Cover the 6 AC viewpoints: **Functional / Behavior / Error / Observability / Verification / Documentation**.

```markdown
## Acceptance Criteria

- [ ] (concrete done-condition 1)
  - Verify: (command or procedure)
- [ ] (concrete done-condition 2)
  - Verify: (...)
```

### [3] AC Gate check

Self-check the AC. **All items must pass — otherwise return to [2].**

| Gate item | Check |
|-----------|-------|
| Refinement document complete | All 8 [2a] mandatory sections (Goal / Non-goal / Analysis / User Story / Use case / Architecture / Failure scenario / AC) are filled in the Issue body — including Analysis / Architecture / Use case at ≥ 1 paragraph (or 3 steps) of real content, not placeholders |
| Specificity | No vague phrasing like "works", "supports", "improves" |
| Verifiability | Each item is objectively checkable by a human or a command |
| Coverage | No gaps versus the stated goal |
| Verification means | Test / command / procedure is included for each item |
| Impact scope | Service restart / regression check is covered (if existing service modified) |
| Failure paths | Edge cases / error behavior are addressed (when relevant) |
| Hardware verification | If firmware / hardware changes, post-flash verification is included |

If the gate passes:
- **Task with brainstorming** → [3.5] User approval
- **Task that skipped brainstorming** → directly [4] Automated implementation; set `status:in-progress` and post a start marker

### [3.5] User approval (brainstorming tasks only)

For tasks where AC was set via interactive brainstorming, **always obtain explicit user approval before starting implementation**.

**Important**: The project's "do not auto-commit unless explicitly told" rule is **overridden** by this [3.5] approval. The approval grants "permission to start implementation **and** to commit/push at [6]". Unless explicitly opted out at approval time, the workflow may proceed to commit & push at [6] automatically.

Procedure:

1. Present a summary of the AC and implementation plan.
2. Ask for approval with options (state explicitly that approval covers commit/push):
   - "Approve — implement & auto commit/push at [6]" (recommended)
   - "Approve — implement, but I will review the commit manually"
   - "Modify AC" (return to [2b]) / "Restart brainstorming" (return to [2a]) / "Hold / later"
3. On "Approve":
   - If a BG code-agent **normal slot is free**, flip the label to `status:in-progress`, post a start marker, proceed to [4].
   - If **all normal slots are busy** (consumer's slot manager returns full), set the label to **`status:ac_approved`** (#505 LABEL-S3) and put the issue in the dispatch queue. The consumer's `<task-notification status=completed>` handler then promotes the oldest `status:ac_approved` issue to `status:in-progress` + [4] as soon as a slot frees up. Do **not** use `status:blocked` for this queue state — `blocked` is reserved for true external dependencies (external API lock, 3rd-party decision, vendor outage). See consumer SKILL.md (`butler` / `leader`) for the slot-management contract.
4. **Do not start implementation without approval.**

**Exception**: If the user has previously said "no need to re-approve" or "go ahead automatically", you may skip approval (log that fact in the issue).

#### Label semantics: `status:ac_approved` vs `status:blocked` (#505)

| Label | Meaning | Owner / next action |
|-------|---------|---------------------|
| `status:ac_approved` | [3.5] approved, waiting for a normal BG slot to free up. Normal queue state. | Consumer's slot manager — dispatch next when slot frees. |
| `status:blocked` | Truly blocked on something **outside** the dev-workflow: external API hard-block (Akamai BMP), 3rd-party / vendor outage, family / external-decision dependency, etc. | External — wait until the dependency resolves. |

Do not collapse the two: putting slot-queue items under `blocked` inflates T's perceived blocker queue and hides real external dependencies.

### [4] Automated implementation

Launch a background agent (`general-purpose` etc.). Include in the agent brief:

- Issue number (`#NNN`)
- All AC items
- Constraints (do not push, do not touch unrelated files, do not break existing tests, etc.)
- Comment format on completion

The agent must post a result comment on the issue when it finishes (ending with the consumer's bot marker, e.g. `<!-- <consumer>-bot -->`).

**BG code agent slot cap (Butler-specific consumer rule, #479 + #494 + #572):** Before dispatching, consumers MUST acquire a slot via `scripts/lib/bg_code_slot.py acquire <normal|express|research> <issue> <agent_id>` and release on completion. Cap is `normal 2 + express 1 = 3` for code (#494, 2026-06-06 normal 1→2) plus `research 2` for heavyweight research/investigation agents (#572); standup/ticket-status burst fan-outs are exempt. Express slot is only allowed when the user explicitly used a trigger phrase (detected by `scripts/lib/detect_express_trigger.py`). See consumer SKILL.md (`leader` / `butler`) for full procedure.

### [5] QA (AC verification)

When the agent reports completion, **the consumer itself** verifies each AC item.

**Status label transition (#497 LABEL-S1):** At the moment the agent posts completion, the consumer auto-transitions the label from `status:in-progress` to **`status:user_confirming`** when AC verification still requires the user (T) to physically touch a device / browser / external service. The consumer's BG completion handler picks the label based on the unchecked AC keyword scan (`verify_bg_completion.py` recommendation — `user_confirming` is the default when completion signal is present but AC is not all checked). When all AC are already ✅, the consumer skips this state and goes straight to `[6] Close`.

- `status:user_confirming` = "implementation done, T physical verification pending" — distinct from `status:action_required` (refinement / decision / physical action remaining) and from `status:in-progress` (BG agent still actively dispatched).
- Once T finishes the manual checks and AC are all ✅, advance to `[6] Close`.

| Verification means | Example |
|--------------------|---------|
| File / code review | `git diff` to confirm the change matches intent |
| Static check | `python3 -c "import ast; ast.parse(open('x.py').read())"` |
| Service health check | `systemctl restart <unit>` → `is-active` → `journalctl -u <unit> -n 30` |
| Behavior check | Run a script end-to-end and verify expected output |
| Output comparison | Compare generated file / log against expected snapshot |

Record per-AC results as a `## QA results` table on the issue.

#### [5e] Mandatory E2E gate (web / UI / pipeline / voice changes)

For any change that touches a **web endpoint, UI/SPA, a runnable pipeline, or a voice / conversational interface** (anything a user or another system actually exercises at runtime), the consumer **itself runs an automated end-to-end test** as part of QA — it is NOT optional and is NOT deferred to the user (team rule: web/UI changes require an automated playwright E2E).

Rules:

1. **Exercise the REAL path, not just mocks.** A mock-only E2E can pass while the real integration is broken. Drive the actual service / live external dependency where feasible. *(Concrete lesson — #688: the mock Kindle-sync E2E passed for weeks, but the first **live** run harvested 0 books because Amazon had silently changed its library DOM (`data-asin` → `li#library-item-option-{ASIN}`). Only a live E2E surfaced it. (verify adversarially).)*
2. **Playwright + screenshot evidence for UI.** Capture before/after screenshots, save under `verification/issue-<N>/`, commit them, and embed them **inline** in the issue comment (raw GitHub URLs) so T can judge from the screenshots alone.
3. **Full-pipeline run for pipeline changes.** Run one real item end-to-end through every stage to completion (e.g. source → OCR → summary → TTS → mp3) and verify each artifact exists and is non-trivial (adversarial inspection of the actual output, not a success marker).
4. **Deploy before E2E for live-service code.** If the change modifies code that a **long-running service / daemon** executes (a voice daemon, a web backend, any resident process), the E2E must run against the **deployed, restarted** service — first fast-forward the running checkout to the merged commit, restart the service, then verify the live behavior (its actual output, not the source tree). **Merging to the main branch does NOT auto-deploy to a running daemon**, and an isolated worktree merge leaves the main checkout behind. *(Lesson: a voice multi-day-weather change was correct on the main branch, but the running daemon kept executing old code until the main checkout was fast-forwarded and the service restarted — until then it answered "I only know today / tomorrow." Implementation success ≠ deployed.)*
5. **Gate ordering.** The automated E2E must pass **before** the issue advances to `status:user_confirming`. `user_confirming` is then only T's lightweight physical confirm (glance at screenshots / one real device tap) — never a substitute for the consumer's own E2E.
6. If an E2E is genuinely impossible to automate (hardware-only, irreducible human judgment), say so explicitly in the QA results and name what T must verify — do not silently skip.
7. **Mock the server contract, not just the happy echo (voice / API E2E).** For a voice or conversational interface — or any E2E that fakes a client↔server protocol (websocket, RPC) — the fake endpoint MUST validate each client→server event against the **real server's contract** and reject what the real API would reject. A mock whose `send()` merely parses-and-echoes a "success" is not an E2E: it accepts malformed payloads the real API rejects, so a whole class of bug (missing required param, wrong discriminator) is structurally invisible. Verify the **real API contract separately** via an **opt-in real-WS / real-endpoint smoke** that is off by default (CI/pre-commit must not auto-run or get charged): gate it on minimum modality (e.g. text-only, no audio) plus a cost-meter hard limit, and skip when the opt-in flag or credential is absent. *(Concrete lesson — #701/#701b: a green voice E2E (8 passed) still let production crash because a `session.update` omitted the required `session.type`; the fake WS echoed it without contract checking. #702 added `assert_valid_realtime_client_event` to every fake `send()` (no-cost, always-on) + an opt-in text-only real-WS smoke gated by the #588 cost-meter. (verify adversarially).)*
8. **Every web / UI AC is classified at refinement time as (A) machine-verifiable or (B) user-manual — no ambiguous middle.** At [2b] each web/UI acceptance criterion must be written as exactly one of:
   - **(A) machine-verifiable** → the consumer drives a **playwright E2E**, captures a screenshot, AND **the agent itself AI-vision-checks that screenshot** (open the PNG with the vision-capable Read tool and confirm the *expected UI state* is actually rendered — not merely that a file was produced or the DOM node exists), then **embeds the screenshot inline in the issue** (raw GitHub URL) as evidence. Capturing/decoding a screenshot without the agent visually confirming the rendered result is **not** a passed AC.
   - **(B) user-manual** → explicitly worded "T verifies by operating X" for the irreducible-human-operation part (a real device tap, a physical purchase, subjective taste). Name exactly what T does.
   A web AC that is neither (e.g. "looks good", "displays correctly" with no E2E+vision and no named T action) is **not refined** — rewrite it into (A) or (B) before the AC Gate. *(Lesson — #708: a Kindle-cover AC captured a screenshot but the rendered result was only truly judged once the agent looked at the image; the rule formalizes that the agent must AI-vision-check, not just capture, and post the evidence inline.)*

Record E2E outcome (path exercised, artifacts, screenshots) in the `## QA results` table.

**If any item is ❌**:
- If the AC was insufficient, return to [2] Refinement.
- If the implementation is buggy, send the agent back to [4] with a fix task.

### [6] Close the ticket

Once all AC are ✅ (and [3.5] approval covered commit/push):

1. **Diff review**: run `git diff` to confirm the final diff matches intent.
2. **Commit**: one or a few commits per issue. English, subject + body, **include the issue number** (e.g. `Fix #NNN: <summary>`).
3. **Direct push** to the current branch (no PR workflow): `git push origin $(git symbolic-ref --short HEAD)`
4. **Update the issue**:
   - `gh issue edit <num> --add-label status:done --remove-label status:in-progress`
   - Post a result comment (must end with the consumer's bot marker)
   - `gh issue close <num>`
