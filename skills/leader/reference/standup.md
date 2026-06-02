---
name: standup
purpose: Full step-by-step standup procedure executed on "hi leader".
---

# Standup — full step-by-step + priority heuristics

This reference expands the "On Activation — Standup" section of SKILL.md.

## TOC

- [Steps in detail](#steps-in-detail)
- [Priority heuristics](#priority-heuristics)
- [Recommendation format](#recommendation-format)
- [Status label display map](#status-label-display-map)
- [When the dream skill was dispatched](#when-the-dream-skill-was-dispatched)

## Steps in detail

### Step 0: Dream cache check (always first)

Before any standup work, run the 24h `cache/dream_last_at` check from SKILL.md. If due, **touch the cache first** (race protection), then dispatch dream in the background. Do not block the standup on dream's completion.

### Step 1: Today's daily file

```bash
TODAY=$(date +%Y-%m-%d)
DAILY="memory/dev/daily/${TODAY}.md"
if [[ ! -f "$DAILY" ]]; then
  printf '# %s\n\n## Work Done\n\n## Decisions\n\n## Next Actions\n' "$TODAY" > "$DAILY"
fi
```

### Step 2: Read daily index (last 2–3 entries)

```bash
head -10 memory/dev/daily/index.md
```

### Step 3: Read the most recent daily log

The previous day's `## Next Actions` is the primary input for "what to do today".

### Step 4: List open issues

```bash
gh issue list --state open --limit 30
```

Group by status label. For each `status:in-progress` issue, also run `gh issue view <num>` to read the current state.

For `type:story` issues, note that they follow the `dev-workflow` skill's 6-stage flow. For `type:epic` issues, note pending refinement (epic to story).

### Step 5: Read decisions index

```bash
head -10 memory/dev/decisions/index.md
```

Pull the last 5 decisions for the standup briefing.

### Step 6: Read retrospectives index

Identify the most recent retrospective whose "reported" column is blank. Read the file and prepare the proposal walk-through.

### Step 7: Brief the user

Sections in order:

1. **Yesterday** — one-sentence summary from yesterday's daily log.
2. **Open issues** — table with `#`, `Status`, `Title`, `Updated`.
3. **Recent decisions** — any from the last week worth noting.
4. **Retrospective report** — summarize the unreported retrospective. Update "reported" column after briefing.
5. **Retrospective proposals — adoption review** — walk through each pending R-N. Accept / reject / defer per proposal.
6. **Carry-over items (needs confirmation)** — uncommitted changes etc.

### Step 8: Recommendation

Apply priority heuristics. Output a single concrete recommendation. Do NOT ask "what would you like to work on?".

## Priority heuristics

1. **in-progress before open** — continue momentum, avoid context-switching.
2. **blocked issues last** — don't suggest something that can't move.
3. **dependency order** — if issue B requires issue A to be done first, suggest A.
4. **shortest path to done** — prefer an issue where the next action is concrete and executable right now.
5. **recency** — if two issues are otherwise equal, prefer the one updated most recently.

## Recommendation format

```
Recommended next: #NNN — [title]
[One sentence explaining why: what unblocks, what momentum it continues, or what it enables.]
```

## Status label display map

When showing the task list to the user, map status labels (no emojis required — consumer may add its own conventions):

- `status:done` → done
- `status:open` → open
- `status:in-progress` → in-progress
- `status:blocked` → blocked
- `status:action_required` → action_required

## When the dream skill was dispatched

If the standup-time dream cache check dispatched dream, mention it briefly in the briefing footer:

> Dream skill is running in the background (memory consolidation + R-D / R-S / R-SH suggestions). Results will surface in tomorrow's standup.

If dream was skipped because < 24h had elapsed, do not mention it.
