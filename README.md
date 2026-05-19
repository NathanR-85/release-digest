# Release Digest

A self-paced companion to Claude Code's built-in `/release-notes`.

`/release-notes` dumps the entire changelog every time, with no dates and no
memory of what you've already seen. `release-digest` keeps a small local
checkpoint and, each run, shows **only** the versions released since the last
time you looked; the full verbatim changelog on top, then a curated **Key Features**, then **Minor Features**.

## Usage

```
/release-digest          # only what's new since you last ran it
/release-digest 7        # everything released in the last 7 days
/release-digest --reset  # mark everything as seen, print one line, stop
```

A bare number is the lookback window in days. That's the whole interface.

## Install

Copy the `release-digest/` folder into your Claude Code skills directory:

```
~/.claude/skills/release-digest/SKILL.md
```

It registers as a user-level skill (`/release-digest`) available in every
project. 

State is stored locally at
`~/.claude/state/release-notes-checkpoint.json` and is never published.

## Notes

The delta is measured against the **latest released** version upstream (the
GitHub `CHANGELOG.md`), not your locally-installed build, so it surfaces
changes even before you update.
