---
name: task-list
purpose: How to display the open task list and produce a single recommendation.
---

# Task list — display + recommendation

This reference expands the "Show Task List" behavior of the leader skill.

## TOC

- [When to display](#when-to-display)
- [Steps](#steps)
- [Priority heuristics](#priority-heuristics)
- [Recommendation format](#recommendation-format)
- [Status label display map](#status-label-display-map)

## When to display

When the user asks to see tasks (any phrasing: "show task list", "what tasks?", "task list", etc.), **or** after any standup.

## Steps

1. Run `gh issue list --state open --limit 30`
2. For each non-done issue, run `gh issue view <num>` to understand subtasks and blockers (especially for `status:in-progress`)
3. Apply priority heuristics (below) to rank issues
4. Display the list as a table with columns: `#`, `Status`, `Title`, `Updated`
5. **Immediately follow with a recommendation** — do not ask what they want to work on

## Priority heuristics

In order:

1. **in-progress before open** — continue momentum, avoid context-switching
2. **blocked issues last** — don't suggest something that can't move
3. **dependency order** — if issue B requires issue A to be done first, suggest A
4. **shortest path to done** — prefer an issue where the next action is concrete and executable right now
5. **recency** — if two issues are otherwise equal, prefer the one updated most recently

## Recommendation format

After the issue table, always output:

> **Recommended next: #NNN — [title]**
> _[One sentence explaining why: what unblocks, what momentum it continues, or what it enables.]_

Never ask "what would you like to work on?" — suggest first, let the user redirect.

## Status label display map

When displaying, map status labels to their plain equivalents (consumers may add their own emoji conventions):

- `status:done` → done
- `status:open` → open
- `status:in-progress` → in-progress
- `status:blocked` → blocked
- `status:action_required` → action_required
