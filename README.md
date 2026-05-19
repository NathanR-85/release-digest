# Release Digest

A self-paced curated digest companion to Claude Code's built-in
`/release-notes`.

`/release-notes` dumps the entire changelog every time, with no dates and no
memory of what you've already seen. `release-digest` keeps a small local
checkpoint and, each run, shows **only** the versions released since the last
time you looked — ruthlessly curated down to ≤8 items that actually affect a
daily user's workflow, ranked most-useful-first.

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

A single ranked list — top item is the headline. Items are ordered by how
useful they are for someone actively building with Claude Code:

1. New capabilities (new view, mode, top-level command, skill / hook /
   subagent / MCP system changes)
2. Model / effort / `/fast` / context-window changes
3. Default changes to high-frequency commands (muscle-memory shifts)
4. Security or auth changes that affect everyone
5. Renamed or removed commands
6. Daily-driver regression fixes (startup hang, common crash, etc.)

Niche flags, bug fixes with narrow scope, internals (OTEL spans, daemon
behavior), platform-edge cosmetics, and MCP-author concerns are all
hard-cut. If you'd skim past it muttering "who uses this," it doesn't ship.

## Install

Copy the `release-digest/` folder into your Claude Code skills directory:

```
~/.claude/skills/release-digest/SKILL.md
```

It registers as a user-level skill (`/release-digest`) available in every
project.

State is stored locally at `~/.claude/state/release-notes-checkpoint.json`
and is never published.

## Notes

- Delta is measured against the **latest released** version upstream (the
  GitHub `CHANGELOG.md`), not your locally-installed build, so it surfaces
  changes even before you update.
- Exactly two network calls per run: one `curl` to the npm registry for the
  authoritative latest version and publish dates, then one `WebFetch` of the
  GitHub changelog. No aggregator sites, no web searches.
- The heavy reading + curation runs in a Sonnet sub-agent in a fresh
  context, so it's safe to run on an Opus 1M-context session without
  tripping the paid-credit gate.

## License

MIT — see [LICENSE](LICENSE).
