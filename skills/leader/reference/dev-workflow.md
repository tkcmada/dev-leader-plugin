# dev-workflow — development task standard flow

This reference is loaded by the leader skill. It defines how a `type:story` issue is processed end-to-end, including the brainstorming → AC → approval → implementation → QA → close pipeline.

All paths in this doc are **repo-relative**. The GitHub repo, push branch, and project root are resolved by the leader at runtime (`gh` / `git remote` / `git rev-parse --show-toplevel`), not hard-coded.

Issues labeled `type:story` are **always** processed by the 6-stage flow defined below.

## Ticket hierarchy: epic / story

```
type:epic ──(epic to story)──▶ type:story ──(6-stage flow [1]–[6])──▶ implement & close
```

- **`type:story` is the only implementation-unit ticket.** Single small tasks are also filed as `type:story`, not as a separate "dev" type. (Some consumers may add a separate fast-lane type — see the consumer's own SKILL.md.)
- A story can exist under an epic or on its own.
- The epic itself is **not** an implementation unit and does not go through the 6-stage flow. Close the epic once all child stories are `status:done`.
- **"add epic ticket"** / **"epic to story"** are leader recognition phrases handled by the leader skill (file `type:epic` / brainstorm refinement → spawn child stories). Refinement fills 5 sections: Goal / Use Case / Failure Scenarios & Edge Cases / Architecture / Acceptance Criteria.
- Slice child stories as **user-facing vertical slices** (e.g. "add logging feature"), not by component (e.g. "ATmega side only"). Link the epic number from the story body.

## When to use this skill

A GitHub Issue qualifies for this workflow if it does any of the following:

- Writing / modifying code
- Creating or significantly modifying skills (SKILL.md) or agent definitions
- Adding / changing external tool / service integration
- Modifying existing automation flows
- Investigating / fixing bugs

Issues that do not qualify (routine task management, ops notes, etc.) are handled by the normal leader flow.

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
[5] QA (leader verifies each AC)
   ↓
[6] Close the ticket
```

### [1] File a ticket

Create a new GitHub Issue.

- **Command**: `gh issue create --title "..." --body "..." --label "status:open,type:story"`
- **Labels**: `status:open` + `type:story`.
- **Title**: short, verb-led summary.
- **Required body sections**: `## Overview` (1–2 sentences) / `## Background` (why, related issues) / `## Next Action` (Owner: leader / agent / user; Action: 1–2 lines).

AC may be undefined at this stage (set in [2]).

### [2] Refinement (Brainstorming → AC)

Runs in two phases:

#### [2a] Brainstorming

**Required when**: creating a new skill/agent, effort ≥ 30 min, spans multiple files, introduces a new capability, impacts a running service, changes hardware behavior.
**Can skip when**: trivial bug fix in one function, doc/comment edits, pattern-matching refactor, renames/typos, dependency bumps, tests/logs only.

Procedure:

1. Explore the design space.
2. Append the **5-section structured template** to the issue body (User Story / Goals & Non-Goals / Approach comparison / Edge Cases / AC candidates — full template below).
3. Compress unknowns to **1–3 questions max** — avoid drip-feeding.
4. Each question must offer **2–4 concrete options** (label + trade-off). If you have a recommendation, place it first and tag `(recommended)`. **Never** use Greek letters (α/β/γ) for option labels — use **uppercase Latin `A`/`B`/`C`/`D`/`E`** only. Full Options principle below.
5. Once aligned, finalize the sections.

#### [2b] Write Acceptance Criteria

Take the "AC candidates" from brainstorming and detail them to the level of verification commands.

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
| Brainstorming complete | For important / complex tasks, the [2a] 5 sections are present |
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
3. On "Approve", flip the label to `status:in-progress`, post a start marker, proceed to [4].
4. **Do not start implementation without approval.**

**Exception**: If the user has previously said "no need to re-approve" or "go ahead automatically", you may skip approval (log that fact in the issue).

### [4] Automated implementation

Launch a background agent (`general-purpose` etc.). Include in the agent brief:

- Issue number (`#NNN`)
- All AC items
- Constraints (do not push, do not touch unrelated files, do not break existing tests, etc.)
- Comment format on completion

The agent must post a result comment on the issue when it finishes (ending with `<!-- leader-bot -->` or the agent's identifier).

### [5] QA (AC verification)

When the agent reports completion, **the leader itself** verifies each AC item.

| Verification means | Example |
|--------------------|---------|
| File / code review | `git diff` to confirm the change matches intent |
| Static check | `python3 -c "import ast; ast.parse(open('x.py').read())"` |
| Service health check | `systemctl restart <unit>` → `is-active` → `journalctl -u <unit> -n 30` |
| Behavior check | Run a script end-to-end and verify expected output |
| Output comparison | Compare generated file / log against expected snapshot |

Record per-AC results as a `## QA results` table on the issue (`<!-- leader-bot -->` trailer).

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
   - Post a result comment (must end with `<!-- leader-bot -->`)
   - `gh issue close <num>`

---

## AC perspectives — 6 viewpoint checklist

When writing AC, sweep through the following viewpoints. **You do not need every viewpoint to apply**, but do not skip any viewpoint that does apply.

### 1. Functional
- [ ] The main use case works end-to-end
- [ ] I/O contract (type, format, range) is satisfied
- [ ] Existing functionality is preserved (no breaking changes)

### 2. Behavior / Performance
- [ ] Latency / processing time is within budget
- [ ] Memory / storage usage is within limits

### 3. Error / Edge cases
- [ ] Expected error cases behave gracefully (no crash, logged)
- [ ] Behavior is defined for empty / invalid / timed-out inputs
- [ ] Existing failure modes are not regressed

### 4. Observability / Operations
- [ ] Necessary information is logged
- [ ] Failures are surfaced to the user (GitHub comment etc.)
- [ ] Configuration changes (if any) are documented

### 5. Verification means
- [ ] Each AC item states **how to verify it**
- [ ] Verification commands are concrete (copy-pasteable)
- [ ] Expected output / state is stated

### 6. Documentation (when applicable)
- [ ] CLAUDE.md / SKILL.md / agent definitions are updated as needed
- [ ] Paths to related skills / scripts are correct

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
- [ ] `ssh pi4 hostname` returns `pi4` without prompting for a password
- [ ] `~/.ssh/config` contains a `Host pi4` entry with HostName/User/IdentityFile set
- [ ] `python3 raspi_test.py --duration 180` keeps reg19=0 and reg22=0 for 3 minutes
```

---

## Options principle (full)

Brainstorming questions must **always offer 2–4 concrete options**. Each option has a `label` (short headline) and a `description` (trade-off).

**Why:**
- Open-ended questions ("how do you want to implement this?") push the design space back to the user.
- Options make the leader's design proposals visible — the user just accepts or rejects.
- If you have a recommendation, place it first and append `(recommended)` to its label.

**Option label convention:** Use **uppercase Latin letters** `A` / `B` / `C` / `D` / `E` (e.g. `Option A`, `案 B`). **Never use Greek letters** (`α` / `β` / `γ` / `δ` / `ε`) — they are hard to type and read aloud.

**Good example:**

> Q: What I2C polling interval?
> - 10 ms (recommended) — good balance of latency and WDT headroom
> - 5 ms — higher responsiveness, more CPU
> - 20 ms — lower load, slight peripheral delay
> - Event-driven — large refactor, out of scope this time

**Bad example:**

> Q: How many ms?
> (forces the user to pick a number; trade-offs are invisible)

**Exceptions:**
- For free-form numeric or date values where options are hard to enumerate, present 3–4 representative options plus a "free input" fallback.
- If the user explicitly says "you decide" or "your call", do not ask — the leader picks the best option and proceeds.

---

## 5-section brainstorming template (full)

Append this to the issue body after [2a] brainstorming aligns direction with the user:

```markdown
## Agreed design (brainstorming result)

### 1. User Story
As a [actor],
I want [what to achieve],
so that [value gained / problem solved].

### 2. Goals & Non-Goals
**Goals**:
- (goal 1)
- (goal 2)

**Non-Goals** (explicitly out of scope this time):
- (item)

### 3. Approach comparison
| Option | Outline | Pros | Cons |
|--------|---------|------|------|
| A | ... | ... | ... |
| B | ... | ... | ... |

**Chosen**: B
**Why**: (why B over A)

### 4. Edge Cases & Failure Modes
- (edge case 1 and behavior)
- (fallback behavior, recovery, etc.)

### 5. AC candidates
AC candidates derived from the alignment (will be finalized in [2b]):
- (...)
```

**User Story — bad vs good example:**

Bad:
> "Save encoder settings"
> (Who? What problem does it solve?)

Good:
> As a Rover operator, I want EncoderDriver4ch settings persisted in EEPROM, so that channel direction/scale survives unintended WDT resets and I don't have to reconfigure mid-run.

---

## Commit conventions (full table)

| Item | Policy |
|------|--------|
| Granularity | One issue = one (or a few) commits |
| Message | English, subject + body, includes `#N` |
| Push | Direct push to the current branch (no PR workflow) |
| Approval gate | **[3.5] user approval doubles as commit/push permission** |
| Agent diff review | The leader always runs `git diff` before committing |
| On failure | Do not amend existing commits — create a new commit |
| Exception | If the user said "I will commit myself" at approval time, stop just before commit in [6] and present the diff |

## Commit message example

```
Fix #234: Persist encoder settings in non-volatile storage

Settings currently live only in RAM and are wiped on unintended resets,
forcing manual reconfiguration mid-run. Persist them with a magic byte
for invalidation and reload on boot.

Verified: settings survive 5 forced resets.
```
