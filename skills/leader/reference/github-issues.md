# GitHub Issues — issue body template + label policy + closure rule

This reference documents the issue conventions used by the leader skill.

## TOC

- [Issue body template](#issue-body-template)
- [Label policy](#label-policy)
- [Closure rule (AC verification gate)](#closure-rule-ac-verification-gate)
- [Status label vocabulary](#status-label-vocabulary)
- [Type label vocabulary](#type-label-vocabulary)
- [Commit message convention](#commit-message-convention)

## Issue body template

Every issue created by the leader uses this template:

```markdown
## Goal
[What is to be achieved and why]

## Subtasks
- [ ] ...

## Decisions

## Acceptance Criteria
- [ ] ...

## Outcome
```

For `type:story` issues, AC may start empty — the dev-workflow skill's [2] Refinement fills them in.

## Label policy

Every issue MUST have:

1. **Exactly one `status:*` label** — the lifecycle state.
2. **At most one `type:*` label** — the workflow category (epic / story / dev / improvement / ...).
3. **Optional `priority:*` / `project:*` / `user:*` / `snooze:*` labels** — consumer-specific.

Issue status is tracked **only by labels**. Do not maintain status in the body — labels are the source of truth.

## Closure rule (AC verification gate)

An issue MUST NOT be closed (`status:done`) unless every `## Acceptance Criteria` checkbox is `[x]`.

If any AC remains `[ ]` at would-be-close time:

- If external verification is still pending, keep `status:in-progress` and add a comment recording the pending step.
- If the AC has an explicit written justification in a closing comment ("AC4 dropped because <reason>"), the leader may close — but the justification must be visible in the issue thread.

The leader runs this check at every "task completes" event before invoking `gh issue close`.

## Status label vocabulary

| Label | Meaning |
|-------|---------|
| `status:open` | Not yet started |
| `status:in-progress` | Active work happening (background agent running or human working) |
| `status:action_required` | Human action needed (pushed button, OAuth login, physical purchase, ...) |
| `status:blocked` | External blocker (waiting on API, hardware fault, third-party) |
| `status:done` | Closed and verified |

## Type label vocabulary

| Label | Owner | Flow |
|-------|-------|------|
| `type:story` | leader + dev-workflow | 6-stage flow (file → refine → AC gate → approve → implement → QA → close) |
| `type:epic` | leader | Coarse goal; "epic to story" decomposes into child stories |

Consumers may add additional types (e.g. `type:dev` for a fast-lane, `type:improvement` for retrospective-derived auto-tickets). The leader skill itself only knows `type:story` and `type:epic`.

## Commit message convention

All commits must be written in English and include enough context to understand the change without reading the diff:

```
<concise subject line: what was added/changed/fixed>

<why: what problem or request prompted this change>
<effect: what behavior changes as a result>
```

**Requirements:**
- Subject line: ≤72 chars, imperative mood ("Add X", "Fix Y", "Require Z")
- Body: always include — never commit with subject only
- Reference the issue with `#N` somewhere in subject or body
- Language: English only

**Good example:**

```
Fix #234: Persist encoder settings in non-volatile storage

Settings currently live only in RAM and are wiped on unintended resets,
forcing manual reconfiguration mid-run. Persist them with a magic byte
for invalidation and reload on boot.

Verified: settings survive 5 forced resets.
```
