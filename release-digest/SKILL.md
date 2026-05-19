---
name: release-digest
description: "Digest of Claude Code changes since the last time you checked. Run instead of or right after /release-notes. Triggers: /release-digest, \"what's new in Claude Code\", \"CC release digest\", \"changes since I last checked\"."
argument-hint: "[N]   (N = days to look back; omit = only what's new since last run; --reset = mark all seen)"
model: sonnet
allowed-tools:
  - Bash
  - WebFetch
  - Read
  - Write
user-invocable: true
---

<objective>
`/release-notes` is a Claude Code built-in that dumps the entire changelog with
no dates and no memory of what you've already seen. This skill is the self-paced
companion: it keeps a checkpoint and, each run, reports ONLY the versions
released since the last time it ran. The delta is measured against the latest
**released** version upstream (not the locally-installed one), so it surfaces
changes even before you update.
</objective>

<cost-contract>
**This skill makes EXACTLY TWO network calls. No more, no exceptions.**

1. One `Bash` call: `curl` the npm registry → authoritative latest version
   AND every version's publish date, in a single JSON response (step 3).
2. One `WebFetch`: the GitHub raw `CHANGELOG.md`, sliced to the delta range,
   verbatim (step 5).

FORBIDDEN — these caused a 5-minute run before and are banned:
- Re-fetching the CHANGELOG to "check latest" — the npm call already gives
  `latest`. Fetch the CHANGELOG once, after the delta is known.
- `WebSearch`, releasebot.io, claudelog, claudefa, or any aggregator. The npm
  registry is the only date source. Dates never require a search.
- Any third network call for any reason. If something fails, STOP and report
  it — do not improvise a fallback fetch.

**Why this was slow before, and what fixes it:** the 5-minute run happened
because (a) Opus was streaming the verbatim text token-by-token, and (b) the
skill made 6 network round-trips instead of 2. The fix is the two-call cap
above AND `model: sonnet` in the frontmatter — Sonnet streams the same
verbatim text noticeably faster than Opus. There is no documented Claude Code
mechanism to skip model-generated streaming, so this is the realistic
ceiling. Do not try to "speed it up" by trimming the verbatim changelog —
faithfulness to `/release-notes` is the whole point.
</cost-contract>

<context>
A bare number is the days to look back:

- `/release-digest`        → only versions newer than the checkpoint
                             (self-paced default; prints nothing if current).
- `/release-digest 7`      → every version released in the last 7 days,
                             regardless of the checkpoint.
- `/release-digest --reset`→ mark everything as seen, print one line, stop.

No other arguments.

Checkpoint file: `~/.claude/state/release-notes-checkpoint.json`
Schema (example values — not real data):
`{ "last_version_seen": "X.Y.Z", "last_checked": "YYYY-MM-DD",
"latest_at_last_check": "X.Y.Z" }`
</context>

<process>

### 1. Handle `--reset` first
If `--reset` is present: do step 3 (npm call) to get `latest`, write the
checkpoint with `last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Print one line:
`Checkpoint reset to v<latest> (<today>). Nothing to report.` — then STOP.
(That is the only network call for a `--reset` run.)

### 2. Read the checkpoint
Read `~/.claude/state/release-notes-checkpoint.json`.

- **Missing file** → bootstrap mode (8-day window, unless a bare number was
  passed — then that many days).
- **Malformed JSON** → do NOT silently rebuild. Print:
  `⚠️ Checkpoint at ~/.claude/state/release-notes-checkpoint.json is
  unreadable: <error>. Re-run with --reset to rebuild, or fix it by hand.`
  Then STOP. (Fail loud — never emit a wrong digest off a corrupt bookmark.)

### 3. NETWORK CALL 1 — npm registry (latest version + all dates)
Single Bash call:

```bash
curl -s "https://registry.npmjs.org/@anthropic-ai/claude-code" \
  | jq -r '{latest: (."dist-tags".latest),
            dates: (.time | to_entries
              | map(select(.key|test("^[0-9]+\\.[0-9]+\\.[0-9]+$")))
              | map({(.key): (.value[0:10])}) | add)}'
```

`latest` = the newest released version (authoritative — do NOT derive this
from the CHANGELOG). `dates` = every version → `YYYY-MM-DD` publish date.
This is the ONLY source of `latest` and of dates. If curl/jq fails, STOP with
the error — no fallback fetch.

### 4. Compute the delta set (no network)
Start point:
- **Bare number `N`** → every version whose publish date ≥ (`latest`'s date
  − `N` days). Normal run; checkpoint WILL move (step 8).
- else if checkpoint exists → every version `>` `last_version_seen`.
- else (bootstrap) → every version within the last 8 days of `latest`'s date.

Compare versions numerically by component (`2.1.139` < `2.1.140` < `2.2.0`);
handle minor/major rollover.

**No-op fast path — checkpoint mode only, if `last_version_seen` ≥ `latest`:**
emit EXACTLY these two lines and nothing else, then go to step 8 (refresh
date) and STOP. (Does NOT apply to a bare-number run — that explicitly asked
for a window.)

```
No new releases since <last_checked> (latest is still v<latest>).
Installed: v<x> · Changelog: v<latest>
```

(Omit the second line only if `claude --version` errors.) Do NOT restate this
in prose, explain the checkpoint, add "I'll have a delta when…" filler, or
comment on installed-vs-changelog skew. One fact, stated once. Any sentence
beyond these two lines is a skill violation.

### 5. NETWORK CALL 2 — CHANGELOG (verbatim, sliced, once)
ONE `WebFetch` of
`https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md`.
Ask it to return, **verbatim and complete**, every bullet for every version in
the delta set computed in step 4 (oldest delta version through `latest`) —
do not summarize, do not drop lines. One fetch only; the delta range is
already known, so there is never a reason to fetch twice.

The CHANGELOG is the content source and must be byte-faithful to what
`/release-notes` shows. Aggregator sites editorialize 30 bullets into 3 — they
are never the content source and were already banned in the cost contract.

If the raw CHANGELOG is unreachable, STOP with a clear error — do not fall
back to a summary site (that produces "summary of a summary" shrinkage).

### 6. Render the FULL CHANGELOG in chat (verbatim, dates from step 3)
Emit, newest version first, every version in the delta with its **complete,
verbatim** bullet list from the source — drop nothing. Head each section
`## <version> (<date>)` using the dates from step 3. This is the scroll-up
reference, in chat where you want it. Start with one header line:

```
What's new — v<start> (<date>) → v<latest> (<date>)

FULL CHANGELOG
```

### 7. Then the curated digest (below a hard divider)
After the verbatim section, emit a hard divider line and the curated digest —
terse, one line each, grouped under bold sub-labels if it helps:

```
════════════════════════════════════════════════════════════

KEY
```
Curated, terse, tailored to how Claude Code is actually used. One tight line
each; group under short bold sub-labels if it helps. Prioritize:
- Model / effort / `/fast` / context-window changes.
- Behavior changes that alter muscle memory (changed defaults, `/model`
  scope, renamed commands).
- Plugin / skill / MCP ecosystem changes.
- OS stability: file-access / path / sleep-wake / daemon / background-session.
- Subagent / orchestration / background-agent / hooks.

```
MINOR
```
Everything else worth a glance, one line each. Don't pad; skip pure-internal
noise but keep what a power user cares about.

Final line: `Installed: v<x> · Latest: v<y>` — `<x>` from `claude --version`
(binary at `~/.local/bin/claude`); omit the line if it errors. Informational
only — never the delta anchor.

No file is written. Everything goes in chat — the verbatim section above for
scrollback, the curated digest below for at-a-glance.

### 8. Move the checkpoint
Unless this was the malformed-checkpoint abort, write
`~/.claude/state/release-notes-checkpoint.json` with
`last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Create `~/.claude/state/` if absent.
Bare-number, bare, and bootstrap runs all advance the checkpoint (including
`last_checked = today`) — once shown, a version counts as seen.

</process>

<notes>
- User-level skill → globally available in every project.
- `model: sonnet` is intentional: fetch + slice + curate doesn't need Opus,
  and Sonnet streams the verbatim output noticeably faster.
- Two network calls, fixed: npm registry (latest + dates), then one CHANGELOG
  fetch. Output is fully in-chat (verbatim + curated); no files written.
</notes>
