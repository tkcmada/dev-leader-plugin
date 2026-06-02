---
name: handoff-from-butler
purpose: Optional consumer-layer pattern routing dev work from a higher-level skill into leader.
---

# Handoff from a consumer skill (e.g. butler ↔ leader)

This reference documents an optional consumer-layer handoff pattern where a higher-level skill (e.g. a household assistant `butler`) routes development-related work to `leader`. The pattern is **opt-in** — vanilla `leader` consumers (e.g. a pure development project) ignore this file.

## TOC

- [Why](#why)
- [Two-skill model](#two-skill-model)
- [Handoff direction (one-way)](#handoff-direction-one-way)
- [Trigger phrases that move work to leader](#trigger-phrases-that-move-work-to-leader)
- [Trigger phrases that stay with the consumer](#trigger-phrases-that-stay-with-the-consumer)
- [Fast-lane / trivial-change types](#fast-lane--trivial-change-types)

## Why

Some projects run **two top-level personas** in the same repo:

- A general-purpose consumer skill (e.g. `butler`) for everyday tasks (calendar, reminders, lookups).
- The `leader` skill for development work (implementing features, fixing bugs, processing `type:story` tickets).

Mixing both into one skill makes the consumer skill bloated and noisy. Splitting them keeps each persona focused.

## Two-skill model

| Skill | Trigger | Scope |
|-------|---------|-------|
| Consumer (e.g. `butler`) | "hi butler" / consumer-defined | Everyday tasks; per-actor / per-user concepts; integrations |
| `leader` | "hi leader" / "hey leader" | Development workflow (epic / story / 6-stage flow); dev memory |

## Handoff direction (one-way)

**Consumer → leader.** The consumer routes development requests to leader. The leader does NOT delegate back to the consumer; if a non-development request arrives at leader, leader replies "please ask the consumer skill for that".

## Trigger phrases that move work to leader

The consumer skill should hand off to leader when it detects:

- Explicit phrases: `epic`, `story`, `refinement`, `AC を定義`, `acceptance criteria`, `brainstorming`, `epic to story`
- Existing GitHub issue labels: `type:story`, `type:epic`
- Development intent words paired with a code area: `implement`, `refactor`, `fix bug in <file>`, `add feature to <module>`

Concrete handoff:

```
Skill(skill="leader")
```

or for a specific issue:

```
Skill(skill="leader", args="#NNN")
```

## Trigger phrases that stay with the consumer

The consumer should NOT hand off (these are non-development):

- "Show me the task list" / "What tasks are open?" — progress query, not implementation request
- "What is the status of #NNN?" — read-only status check
- Any everyday-task phrase the consumer handles natively (calendar, reminders, weather, ...)

## Fast-lane / trivial-change types

A consumer that wants a faster path for trivial changes (e.g. typo fix, 1-line label change) may declare a `type:dev` (or similar) label and process those issues **directly** instead of handing off:

| Type | Owner | Flow |
|------|-------|------|
| `type:dev` (consumer-defined) | Consumer | Single commit, no PR, no refinement, no [3.5] approval |
| `type:story` | leader + dev-workflow | Full 6-stage flow |
| `type:epic` | leader | Coarse goal, decomposed via "epic to story" |

This is **purely a consumer-side choice**. The `leader` skill itself only knows `type:story` and `type:epic`.

Mid-implementation escalation: if a `type:dev` change turns out non-trivial during implementation, the consumer should **stop**, relabel to `type:story`, and hand off to leader.
