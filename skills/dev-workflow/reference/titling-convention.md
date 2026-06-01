# Titling convention — `[code]` / `[code-Sn]` epic / story prefix

A consumer that uses epic / story hierarchy can adopt this short common title prefix convention to make standup / list views scannable.

## TOC

- [Convention](#convention)
- [Examples](#examples)
- [Rules](#rules)
- [Cross-references](#cross-references)

## Convention

An epic and its child stories share a **short common title prefix `code`**:

- **epic** = `[code] <name>` (e.g. `[speed] Realtime latency improvement`)
- **child story** = `[code-Sn] <name>` (e.g. `[speed-S1] Latency measurement baseline`)

## Examples

| code | epic | child stories |
|------|------|---------------|
| `[speed]` | `[speed] Latency improvement` | `[speed-S1]`, `[speed-S2]`, `[speed-S3]` |
| `[lib]` | `[lib] Shopping library refactor` | `[lib-S1]`, `[lib-S2]` |

## Rules

- **`code` is ≤ 3 characters**. Latin letters, numerals, or non-Latin glyphs (kanji / kana / hangul) are all allowed.
- The `code` is assigned **when the epic is created** (mnemonic, non-colliding) and **reused by every child story**.
- A standalone story with no parent epic needs no prefix.
- The prefix lives **only in the title**. In-body / GitHub-rendered cross-references stay as plain `#N` so GitHub auto-links them.
- New epic registration: pick a fresh ≤3-char `code` not already used by any open or closed epic in the same repo.

## Cross-references

- In Issue body / comment that will be rendered by GitHub → use bare `#N` (GitHub auto-links).
- In any text output that is shown **outside** GitHub render (e.g. chat reply, voice briefing) → consumers may require explicit hyperlinks `[#N <short-title>](<issue-url>)`. See the consumer's own SKILL.md for that requirement.
