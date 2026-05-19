---
name: release-digest
description: "Digest of Claude Code changes since the last time you checked. Run instead of or right after /release-notes. Triggers: /release-digest, \"what's new in Claude Code\", \"CC release digest\", \"changes since I last checked\"."
argument-hint: "[N]   (N = days to look back; omit = only what's new since last run; --reset = mark all seen)"
allowed-tools:
  - Bash
  - Read
  - Write
  - Agent
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

<architecture>
The heavy work (one large WebFetch + slicing the verbatim changelog + emitting
the digest) runs inside a sub-agent on Sonnet, not on the main thread. Reasons:

1. **Model isolation from credit gates.** The session model may be Opus with
   1M-context, which requires paid usage credits — emitting hundreds of
   verbatim lines on the main thread can trip that gate. A sub-agent on Sonnet
   has its own model and a fresh context, so it never triggers the gate.
2. **Speed.** Sonnet streams the verbatim section meaningfully faster than
   Opus.
3. **Resilience.** If the session model changes, the skill keeps working.

The orchestrator (main thread) does only cheap, deterministic work: argument
parsing, checkpoint read/write, one `curl` to npm, delta computation. Then it
delegates the rest to the sub-agent. The orchestrator relays the sub-agent's
final message verbatim — no preface, no commentary.
</architecture>

<cost-contract>
EXACTLY TWO network calls across the whole run:

1. `curl` the npm registry (on main) → authoritative `latest` version AND
   every version's publish date in one JSON response.
2. One `WebFetch` (inside the sub-agent) → GitHub raw `CHANGELOG.md`, sliced
   to the delta range, verbatim.

For `--reset` and the no-op fast path, only call 1 happens — no sub-agent.

FORBIDDEN (banned because they caused a 5-minute run before):
- Re-fetching the CHANGELOG to "check latest" — npm already gives `latest`.
- `WebSearch`, releasebot.io, claudelog, claudefa, any aggregator — for
  content or for dates. The npm registry is the only date source.
- Any third network call for any reason. If something fails, STOP and report
  the error — no fallback fetch, no improvisation.
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

### 1. Handle `--reset` first (no sub-agent)
If `--reset` is present: run the npm curl from step 3 to get `latest`. Write
the checkpoint with `last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Print one line:
`Checkpoint reset to v<latest> (<today>). Nothing to report.` — STOP.

### 2. Read the checkpoint
Read `~/.claude/state/release-notes-checkpoint.json`.

- **Missing file** → bootstrap mode (8-day window, unless a bare number was
  passed — then that many days).
- **Malformed JSON** → do NOT silently rebuild. Print:
  `⚠️ Checkpoint at ~/.claude/state/release-notes-checkpoint.json is
  unreadable: <error>. Re-run with --reset to rebuild, or fix it by hand.`
  Then STOP. Fail loud.

### 3. NETWORK CALL 1 — npm registry (on main, no sub-agent)
Single Bash call:

```bash
curl -s "https://registry.npmjs.org/@anthropic-ai/claude-code" \
  | jq -r '{latest: (."dist-tags".latest),
            dates: (.time | to_entries
              | map(select(.key|test("^[0-9]+\\.[0-9]+\\.[0-9]+$")))
              | map({(.key): (.value[0:10])}) | add)}'
```

This is a pure Bash call — no model generation, no credit gate risk. Parse
`latest` and `dates`. If curl/jq fails, STOP with the error.

### 4. Compute the delta set (no network)
Start point:
- **Bare number `N`** → every version whose publish date ≥ (`latest`'s date
  − `N` days). Normal run; checkpoint WILL move (step 7).
- else if checkpoint exists → every version `>` `last_version_seen`.
- else (bootstrap) → every version within the last 8 days of `latest`'s date.

Compare versions numerically by component (`2.1.139` < `2.1.140` < `2.2.0`);
handle minor/major rollover.

**No-op fast path — checkpoint mode only, if `last_version_seen ≥ latest`:**
emit EXACTLY these two lines and nothing else, then go to step 7 (refresh
date) and STOP. (Does NOT apply to a bare-number run — that explicitly asked
for a window.)

```
No new releases since <last_checked> (latest is still v<latest>).
Installed: v<x> · Changelog: v<latest>
```

(Omit the second line only if `claude --version` errors.) Do NOT restate this
in prose, explain the checkpoint, add filler, or comment on installed-vs-
changelog skew. One fact, stated once. Any sentence beyond these two lines is
a skill violation.

Because step 1 ran via Bash and this fast path emits only two lines, it never
trips the 1M-context credit gate.

### 5. Delegate the digest to a Sonnet sub-agent (NETWORK CALL 2 happens inside)
Spawn one Agent tool call with these exact parameters:

- `subagent_type`: `general-purpose`
- `model`: `sonnet`
- `description`: `Render Claude Code release digest`
- `prompt`: use the template below, filling in the placeholders

Prompt template for the sub-agent:

```
You are rendering a Claude Code release digest. You run on Sonnet
specifically to (a) avoid the parent session's 1M-context credit gate and
(b) stream the verbatim changelog faster than Opus would.

WORLD-CLASS STANDARD applies to your output: no placeholders, no shortcuts,
no truncation of verbatim content, no near-misses. Every bullet from the
delta range appears, faithful to the source.

Inputs from the orchestrator (already computed):
- delta_versions:  <comma-separated list, newest first, e.g. "2.1.144,2.1.143,2.1.142">
- dates:           <JSON map: {"2.1.144":"2026-05-18", ...}>
- latest:          <e.g. "2.1.144">
- start_version:   <oldest in delta, e.g. "2.1.142">
- start_date:      <date of start_version>
- latest_date:     <date of latest>
- installed:       <output of `claude --version`, or "" if it errored>

ONE network call permitted (and required): a single WebFetch to
https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md
asking for every bullet, verbatim and complete, for every version in
delta_versions. Do not summarize, do not drop lines. If the fetch fails,
return exactly: `ERROR: changelog fetch failed — <reason>` and stop. No
fallback to aggregator sites.

Then emit your FINAL message in this exact shape — no preface, no
"Here's the digest:" framing, no trailing commentary. Your final message IS
what the user sees.

What's new — v<start_version> (<start_date>) → v<latest> (<latest_date>)

FULL CHANGELOG

## <version> (<date>)
- <bullet>
- <bullet>
...
(newest first, every delta version, every bullet verbatim)

════════════════════════════════════════════════════════════

KEY
**<sub-label>:** <one tight line>
**<sub-label>:** <one tight line>
...

Curate aggressively. Tailor to how Claude Code is actually used. Prioritize:
- Model / effort / /fast / context-window changes.
- Behavior changes that alter muscle memory (defaults that changed, /model
  scope, renamed commands).
- Plugin / skill / MCP ecosystem changes.
- OS stability: file-access / path / sleep-wake / daemon / background-session.
- Subagent / orchestration / background-agent / hooks.

MINOR
- <one line each, everything else worth a glance; skip pure-internal noise>

Installed: v<installed> · Latest: v<latest>
(Omit this final line if `installed` is empty.)

Return only the digest. The orchestrator will relay your message verbatim.
```

### 6. Relay the sub-agent's final message
Output the sub-agent's final message exactly as returned. Do not add a
preface ("Here's what's new…"), a trailing summary, or any commentary. The
user sees only the digest the sub-agent produced.

### 7. Advance the checkpoint
Unless step 2 aborted on malformed JSON, write
`~/.claude/state/release-notes-checkpoint.json` with
`last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Create `~/.claude/state/` if absent.
Bare-number, bare, and bootstrap runs all advance the checkpoint (including
`last_checked = today`).

</process>

<notes>
- User-level skill → globally available in every project.
- The sub-agent is the explicit shield against the session's model + credit
  gate. Don't remove it. Inline skill `model:` frontmatter has been observed
  not to shield reliably from 1M-context credit gates.
- Two network calls, fixed: npm registry (on main), then one CHANGELOG fetch
  (inside the sub-agent). All output is in chat — no files written.
</notes>
