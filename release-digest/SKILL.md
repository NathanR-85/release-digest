---
name: release-digest
description: "Digest of Claude Code changes since the last time you checked. Run instead of or right after /release-notes. Triggers: /release-digest, \"what's new in Claude Code\", \"CC release digest\", \"changes since I last checked\"."
argument-hint: "[N]   (N = days to look back; omit = only what's new since last run; --reset = mark all seen)"
allowed-tools:
  - Bash
  - WebFetch
  - WebSearch
  - Read
  - Write
user-invocable: true
---

<objective>
`/release-notes` is a Claude Code built-in that dumps the entire changelog with
no dates and no memory of what you've already seen. This skill is the self-paced
companion: it keeps a checkpoint and, each run, reports ONLY the versions
released since the last time it ran — full list on top, curated Key/Minor
digest below a hard divider. The delta is measured against the **latest
released** version upstream (not the locally-installed one), so it surfaces
changes even before you update.
</objective>

<context>
$ARGUMENTS is normally a single bare number — the days to look back:

- `/release-digest`        → show only versions newer than the checkpoint
                             (self-paced default; prints nothing if current).
- `/release-digest 7`      → show every version released in the last 7 days,
                             regardless of the checkpoint.
- `/release-digest --reset`→ mark everything as seen, print one line, stop.

No other arguments. A bare integer is the lookback window; that's the whole
interface besides bare (checkpoint mode) and `--reset`.

Checkpoint file: `~/.claude/state/release-notes-checkpoint.json`
Schema (example values — not real data):
`{ "last_version_seen": "X.Y.Z", "last_checked": "YYYY-MM-DD",
"latest_at_last_check": "X.Y.Z" }`
</context>

<process>

### 1. Handle `--reset` first
If `--reset` is present: determine latest version (step 3), write the
checkpoint with `last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Print one line:
`Checkpoint reset to v<latest> (<today>). Nothing to report.` — then STOP.

### 2. Read the checkpoint
Read `~/.claude/state/release-notes-checkpoint.json`.

- **Missing file** → first-run / bootstrap mode (8-day window, unless a bare
  number was passed — then use that many days).
- **Malformed JSON** → do NOT silently rebuild. Print:
  `⚠️ Checkpoint at ~/.claude/state/release-notes-checkpoint.json is
  unreadable: <error>. Re-run with --reset to rebuild, or fix it by hand.`
  Then STOP. (Fail loud — never emit a wrong digest off a corrupt bookmark.)

### 3. Get the AUTHORITATIVE changelog (not a summary site)

The FULL CHANGELOG section MUST be byte-faithful to what `/release-notes`
shows — same source, same granularity. Aggregator sites (releasebot, etc.)
editorialize and collapse 30 bullets into 3; they are NEVER the content
source. They are only acceptable for attaching dates.

- **Content (required):** `WebFetch https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md`
  — ask it to return, **verbatim and complete**, every bullet for each version
  in the delta range (do not summarize, do not drop lines). If the page is
  large, fetch and slice to the needed version range, but keep every bullet
  within that range intact.
- **`latest`** = the newest version heading in that CHANGELOG.
- **Dates (enrichment only):** one `WebSearch`/`WebFetch` to a dated source
  (e.g. releasebot.io) solely to map the delta version numbers → release
  dates for the headers. Dates missing → show version without a date; never
  let date lookup gate or shrink the content.

If the GitHub CHANGELOG is unreachable, STOP with a clear error — do NOT
fall back to a summary site for content (that produces the "summary of a
summary" shrinkage). A dateless-but-complete digest is fine; a
complete-content-less one is not.

### 4. Compute the delta set
Start point:
- **Bare number `N` given** → start point = the latest release date minus `N`
  days (date-based window). This is a normal run — the checkpoint WILL move
  (step 6).
- else if checkpoint exists → `last_version_seen`.
- else (no number, no checkpoint — bootstrap) → all versions released within
  the last 8 days counting back from the latest release date.

Delta = every version strictly greater than the start point and ≤ `latest`.
Compare numerically by component (`2.1.139` < `2.1.140` < `2.2.0`); handle
minor/major rollover.

**No-op case — if `last_version_seen` ≥ `latest`:** emit EXACTLY these two
lines and nothing else, then go to step 6 (refresh date only) and STOP:

```
No new releases since <last_checked> (latest is still v<latest>).
Installed: v<x> · Changelog: v<latest>
```

(Omit the second line only if `claude --version` errors.) Do NOT restate this
fact in prose, do NOT explain the checkpoint, do NOT add "I'll have a delta
when…" filler, do NOT comment on installed-vs-changelog skew. One fact, stated
once. Any sentence beyond these two lines is a skill violation.

### 5. Render output — two stacked parts

```
What's new since v<start> (<date>) → v<latest> (<date>)

FULL CHANGELOG
```
Then, newest version first, every version in the delta with its **complete,
verbatim** bullet list from the source — drop nothing. This is the scroll-up
reference.

```

════════════════════════════════════════════════════════════

KEY FEATURES
```
Curated, terse, tailored to how you actually use Claude Code. Prioritize and
call out:
- Model / effort / `/fast` / context-window changes.
- Behavior changes that alter muscle memory (defaults that changed, `/model`
  scope, renamed commands).
- Plugin / skill / MCP ecosystem changes.
- OS stability: file-access / path / sleep-wake / daemon / background-session
  fixes.
- Subagent / orchestration / background-agent / hooks features.
Each as one tight line, grouped under short bold sub-labels if it helps.

```

MINOR FEATURES & FIXES
```
Everything else worth a glance, one line each. Don't pad; skip pure-internal
noise but keep anything a power user would care about.

Final line (optional, informational only):
`Installed: v<x> · Latest: v<y>` — get `<x>` from `claude --version` (binary at
`~/.local/bin/claude`); if it errors, omit the line. This is NOT the delta
anchor — never gate the digest on the installed version.

### 6. Move the checkpoint
Unless this was the malformed-checkpoint abort, write
`~/.claude/state/release-notes-checkpoint.json` with
`last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Create `~/.claude/state/` if absent.
(Bare-number, bare, and bootstrap runs all advance the checkpoint — once
you've been shown a version it counts as seen.)

</process>

<notes>
- This is a user-level skill → globally available in every project.
- Source has no per-bullet dates; version order is authoritative, dates are
  best-effort enrichment from the web source.
- Keep the FULL CHANGELOG faithful (verbatim) — the curated section is where
  judgment/compression happens, never the full list.
</notes>
