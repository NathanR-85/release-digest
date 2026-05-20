# Release Digest

A self-paced curated digest companion to Claude Code's built-in
`/release-notes`.

`/release-notes` dumps the entire changelog every time, with no dates and no
memory of what you've already seen. `release-digest` keeps a small local
checkpoint and, each run, shows **only** the versions released since the last
time you looked — ruthlessly curated to what actually matters for a daily
user, split into **Features** and **Bugs**, each ranked most-useful-first.

It is **not** a verbatim mirror. For the full bullet-by-bullet list, run
`/release-notes`. This skill exists for the curation.

## Usage

```
/release-digest          # only what's new since you last ran it
/release-digest 7        # everything released in the last 7 days
/release-digest --reset  # mark everything as seen, print one line, stop
```

A bare number is the lookback window in days. That's the whole interface.

## What you get

Two sections, **Features** then **Bugs**. Each bullet starts with a bold
TL;DR a busy reader can scan; the rest of the line is the detail. There's
no item cap — the bar is "would a busy reader thank you for surfacing
this?", so a quiet release might be 2 lines and a busy one might be 12.

Within each section, items are ordered most-impactful-first:

**Features ranking**
1. New capabilities (new view, mode, top-level command, skill / hook /
   subagent / MCP system changes)
2. Model / effort / `/fast` / context-window changes
3. Default changes to high-frequency commands (muscle-memory shifts)
4. Renamed or removed commands

**Bugs ranking**
1. Security or auth holes closed (affects everyone whether you noticed or
   not)
2. Daily-driver regressions (startup hang, common crash, slash command
   suddenly broken)

Niche flags, narrow bug fixes, internals (OTEL spans, daemon behavior),
platform-edge cosmetics, and MCP-author concerns are all hard-cut. If
you'd skim past it muttering "who uses this," it doesn't ship.

## Install

Point Claude Code at this repo and tell it where to scope the skill —
user-level by default (available globally in every project), or scoped
into a single project's `.claude/skills/` if you only want it there.

State stays local at `~/.claude/state/release-notes-checkpoint.json` and
is never published.

## Notes

- Delta is measured against the **latest released** version upstream (the
  GitHub `CHANGELOG.md`), not your locally-installed build, so it surfaces
  changes even before you update.
- Exactly two network calls per run: one `curl` to the npm registry for
  the authoritative latest version and publish dates, then one `WebFetch`
  of the GitHub changelog. No aggregator sites, no web searches.
- The heavy reading + curation runs in a Sonnet sub-agent in a fresh
  context, so it's safe to run on an Opus 1M-context session without
  tripping the paid-credit gate.

## License

MIT — see [LICENSE](LICENSE).
