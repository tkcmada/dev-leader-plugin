---
name: wrap-up
purpose: Session wrap-up, good-bye end-of-session, and retrospective flow details.
---

# Wrap-up / Good-bye / Retrospective

This reference expands the three end-of-session flows of the leader skill. The main `SKILL.md` keeps only the trigger phrases; the detailed steps live here.

## TOC

- [Session wrap-up (user still present)](#session-wrap-up-user-still-present)
- [Good-bye (user gone — autonomous)](#good-bye-user-gone--autonomous)
- [Retrospective — review and improve](#retrospective--review-and-improve)
- [Retrospective output format](#retrospective-output-format)
- [What NOT to flag](#what-not-to-flag)

## Session wrap-up (user still present)

Triggered by **"wrap up"**. Performs a session summary while the user is still present. Pairs with the standup as the session's closing procedure.

1. **Today's summary** — Finalize the `## Work Done` and `## Decisions` sections in today's `memory/dev/daily/YYYY-MM-DD.md`. Merge any fragmented mid-session notes into a clean, readable form.
2. **Tomorrow's Next Actions** — Write the items to start on tomorrow under `## Next Actions`.
3. **Surface uncommitted changes** — Present a summary of `git status` / `git diff` and ask the user whether to commit. If approved, commit and push (this user approval overrides the project's default "do not auto-commit" constraint).
4. **Update daily index** — Refresh today's row in `memory/dev/daily/index.md` with the final summary.
5. **Briefing** — Concisely report today's summary (what was done, what was decided, tomorrow's Next Actions) to the user.

## Good-bye (user gone — autonomous)

Triggered by **"good bye"** or **"bye"**. Assumes the **user is no longer present**. "Bye" contains "wrap up" but runs autonomously without waiting for user responses.

1. **Run wrap-up steps 1, 2, and 4** — Finalize today's daily log (Work Done / Decisions, merge fragments), write tomorrow's Next Actions, update the daily index.
2. **Tidy memory** — Clean up the daily log and merge fragmented sections. Verify that the `decisions/`, `notes/`, and `retrospectives/` index files are consistent with the actual files on disk; fix any drift.
3. **Run a retrospective** — Execute the "Retrospective" flow below.
4. **Defer items needing confirmation** — For anything that requires user confirmation (e.g. uncommitted changes), do **not** commit. Instead, record them in the retrospective file under "Carry-over (needs confirmation)" so they surface in tomorrow's standup.
5. Steps 1–4 run **autonomously without waiting for user responses**.

## Retrospective — review and improve

Triggered by **"retrospective"** or **"retro"** as a standalone command. Also runs as part of the "good bye" flow.

Review the session and propose concrete improvements to the leader / dev-workflow / agent setup.

### What to review

1. **Task accuracy** — Were priorities correct? Was anything missed, re-ordered, or added mid-session? Note any pattern.
2. **Agent routing** — Did the routed agent actually solve the problem? Did it need re-routing or manual fixing? If so, the routing table or agent SKILL.md may need updating.
3. **Skills** — Did any skill mislead (wrong wiring, wrong command, outdated info)? Propose a specific fix.
4. **Leader behavior** — Did standup/recommendations match what was actually worked on? Were there surprises the standup should have surfaced?

## Retrospective output format

For each finding, output a Proposal with a unique ID (R-1, R-2, ...) and one of:

> **R-N — Routing fix** — Change `<agent>` routing rule: [proposed diff]
> Status: pending

> **R-N — Skill fix** — In `.claude/skills/<name>/SKILL.md`, update [section]: [proposed change]
> Status: pending

> **R-N — Process fix** — Add/change [leader behavior rule]: [proposed change]
> Status: pending

> **R-N — No change needed** — [explain why this session went smoothly]
> Status: n/a

Then record the retrospective to `memory/dev/retrospectives/YYYY-MM-DD.md`. Each Proposal is captured under a `## Proposals` section with its current status. The next standup walks through pending Proposals and asks the user to accept / reject / defer (see Standup step in `standup.md`).

The file captures:

- **Good points** — what went well
- **Improvements** — what could be better
- **Process / skill issues** — workflow or skill-definition problems
- **Proposals** — each finding as R-N with a status line (pending / accepted / rejected / deferred / n/a)
- **Action Items** — concrete actions to try next session
- **Carry-over (needs confirmation)** — items awaiting user confirmation (e.g. commit approval); surfaced in tomorrow's standup

Add a row to `memory/dev/retrospectives/index.md` (columns: Date / Summary / Reported / Adoption). Leave "Reported" and "Adoption" blank — the next standup fills them in after briefing the user and walking through the Proposals.

## What NOT to flag

- One-off external surprises (can't prevent in a skill)
- Correct behavior that just took longer than expected
- Things that are already captured in `memory/dev/notes/`
