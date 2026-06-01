# Outcome loop — R-N / R-D / R-S / R-SH relationship

The **outcome loop** is a 3-layer record-improvement pipeline. Each layer captures different signal at different times in an issue's life. This reference describes how dream-skill outputs (R-D / R-S / R-SH) integrate with the broader loop maintained by the consumer (`leader` and/or `butler`).

## TOC

- [The 3 layers](#the-3-layers)
- [Layer 1: Issue `## Outcome` section](#layer-1-issue--outcome-section)
- [Layer 2: Retrospective Proposals (R-N)](#layer-2-retrospective-proposals-r-n)
- [Layer 3: Dream R-D / R-S / R-SH](#layer-3-dream-r-d--r-s--r-sh)
- [How the layers reinforce each other](#how-the-layers-reinforce-each-other)
- [Prefix table](#prefix-table)
- [Design rationale](#design-rationale)

## The 3 layers

| Layer | Where | Written by | When | Surfaced by |
|-------|-------|-----------|------|-------------|
| **Issue `## Outcome` section** | GitHub issue body | consumer at [6] Close | per-issue, at close time | Standup "open issues" / `gh issue view` |
| **Retrospective Proposals (R-N)** | `memory/dev/retrospectives/YYYY-MM-DD.md` | consumer at `wrap up` / `good bye` / `retrospective` | per-session | Standup "Retrospective proposals" review |
| **Dream R-D / R-S / R-SH** | `memory/shared/notes/dream-*-suggestions-YYYY-MM-DD.md` | dream Phase 4 / 6 / 7 | when the operator invokes the dream skill manually | Standup "retrospective proposals" + auto-filed GitHub issues for R-S / R-SH |

## Layer 1: Issue `## Outcome` section

When closing an issue at [6], the consumer appends a `## Outcome` section to the issue body capturing what actually happened (vs. what the AC promised):

```markdown
## Outcome

- AC<N> ✅ <how verified, command + expected output>
- AC<N+1> ✅ <...>
- AC<N+2> ❌ rolled back because <reason>; follow-up Issue #<next>

Side-effects observed:
- <surprise behavior caught at QA>

Commit: <hash>
```

This is the **per-issue record** — the source of truth for "did this story actually achieve what it intended?". Future agents reading the closed issue see both the AC and the Outcome.

## Layer 2: Retrospective Proposals (R-N)

End-of-session retrospective collects observed problems with the consumer's own behavior, the agent routing, or the dev-workflow itself, and writes them as R-1, R-2, ... Proposals in `memory/dev/retrospectives/YYYY-MM-DD.md`.

The next standup walks through pending Proposals and asks the user one of:

- **accept** → apply the fix in this session; small change = inline edit + commit/push gate; large change = file a `type:story` issue + run the 6-stage flow
- **reject** → mark closed with a one-line reason
- **defer** → leave `pending`; resurface next standup

The retrospective index tracks adoption status (`N/M adopted (K deferred, J rejected)`) so the maintainer can see whether the team is actually closing the improvement loop.

## Layer 3: Dream R-D / R-S / R-SH

When invoked manually by the operator, the dream skill runs and emits three flavors of suggestion:

- **R-D** (Phase 4) — cross-cutting behavioural patterns observed across multiple daily logs / JSONL signals. Written to `memory/shared/notes/dream-suggestions-YYYY-MM-DD.md`. Surfaced at the next standup, adjudicated accept / reject / defer.
- **R-S** (Phase 6) — Skill / agent improvement proposals, derived from friction signals. **Each R-S is filed as its own GitHub issue** (`status:action_required` + `type:dev`). Adjudication on the issue itself: accept = start work / reject = close / defer = `status:blocked`.
- **R-SH** (Phase 7) — Skill Health Check detections (SKILL.md line-count anomalies, missing TOCs, nested references, oversized descriptions). Same flow as R-S — auto-filed as a GitHub issue.

R-S / R-SH file at generation time precisely because they target the skill definitions themselves (long-lived, version-controlled); a forgotten suggestion in a markdown file is much less likely to be acted on than a tracked GitHub issue.

## How the layers reinforce each other

1. **AC fails at QA** → consumer records `❌` in Outcome → opens follow-up issue (Layer 1 → new ticket).
2. **Same routing mistake recurs** → consumer's retrospective flags R-N "agent routing" Proposal (Layer 2).
3. **Skill misled the consumer multiple times in JSONL** → dream Phase 6 emits R-S targeted at that skill (Layer 3).
4. **SKILL.md grew too large** → dream Phase 7 emits R-SH proposing a section to extract into `reference/<topic>.md` (Layer 3).

The dream skill is intentionally read-only at the source layer (never edits skills / scripts / settings); it converts friction into actionable issues so the maintainer always remains in the approval seat for code/skill changes.

## Prefix table

| Prefix | Source | Output file | Standup surface | adoption row | Auto-file issue |
|--------|--------|-------------|-----------------|--------------|-----------------|
| R-N | daily log (Phase 5) | `memory/users/<actor>/daily/YYYY-MM-DD.md` `## Retrospective` | yes | accept/reject/defer | accept only |
| R-D | dream consolidate (Phase 4 trailer) | `memory/shared/notes/dream-suggestions-YYYY-MM-DD.md` | yes | accept/reject/defer | accept only |
| R-S | dream self-improvement (Phase 6) | `memory/shared/notes/dream-skill-suggestions-YYYY-MM-DD.md` | yes | accept/reject/defer | **filed at generation** (`status:action_required` + `type:dev`) |
| R-SH | dream skill health check (Phase 7) | same file (append) | yes | accept/reject/defer | **filed at generation** (`status:action_required` + `type:dev`) |

R-S / R-SH are pre-filed at generation; the maintainer's adjudication becomes "accept (= start work) / reject (= close issue) / defer (= keep `status:blocked`)".

## Design rationale

- **Per-issue (Layer 1)** is enough for "did this one ship cleanly?" but does not scale to systemic process problems.
- **Per-session (Layer 2)** captures process problems that one human noticed in one sitting.
- **Cross-session (Layer 3)** catches patterns that only become visible after dozens of sessions have left their JSONL trace — exactly the kind of pattern a human would forget.

All three layers feed back into either (a) closing existing work cleanly, (b) filing new work, or (c) editing skill definitions after user approval. The loop is **never auto-applied** at the skill-definition level — every R-S / R-SH still requires a human accept before the corresponding SKILL.md changes.
