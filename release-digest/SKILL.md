---
name: release-digest
description: "Curated digest of Claude Code changes since you last checked — only what affects a daily user's workflow. For verbatim notes, run /release-notes. Triggers: /release-digest, \"what's new in Claude Code\", \"CC release digest\", \"changes since I last checked\"."
argument-hint: "[N]   (N = days to look back; omit = only what's new since last run; --reset = mark all seen)"
allowed-tools:
  - Bash
  - Read
  - Write
  - Agent
user-invocable: true
---

<objective>
A CURATED digest of Claude Code changes since the last time you checked —
not a verbatim mirror. The built-in `/release-notes` is the verbatim source
of truth; this skill exists for the curation. Pulls only changes that affect
a typical user's daily workflow; hard-skips niche flags, internals, narrow
bug fixes, and platform-edge cosmetic fixes.

Failure mode: emitting items the user would skim past muttering "who uses
this." Cutting too aggressively is forgiven; over-inclusion is the bug.
</objective>

<architecture>
The skill runs the heavy reading + curation inside a sub-agent on Sonnet:

1. **Credit-gate safety.** The session may be on Opus 1M-context, and the
   WebFetch payload (~700 lines for a multi-version delta) plus session
   history could trip the paid-tier gate. The sub-agent has a fresh small
   context — never trips, regardless of session state.
2. **Output is small either way.** Sub-agent reads ~700 verbatim bullets but
   returns only ≤8 curated items, so the orchestrator's relay is tiny and
   fast on any model.

Orchestrator only does cheap deterministic work: curl, checkpoint read/write,
delta math, sub-agent dispatch. The orchestrator does NOT read the verbatim
changelog — that stays inside the sub-agent's context and is discarded when
it finishes.
</architecture>

<cost-contract>
EXACTLY TWO network calls across the run:

1. `curl` the npm registry (on main) — authoritative `latest` AND every
   version's publish date, in one JSON response.
2. One `WebFetch` (inside the sub-agent) — GitHub raw `CHANGELOG.md`, the
   delta range, verbatim. The sub-agent reads it and curates; it is never
   relayed verbatim to chat.

For `--reset` and the no-op fast path, only call 1 happens — no sub-agent.

FORBIDDEN — caused a 5-minute run before, banned:
- Re-fetching the CHANGELOG to "check latest" — npm gives `latest`.
- `WebSearch`, releasebot.io, claudelog, claudefa, any aggregator. The npm
  registry is the only date source.
- Any third network call. If something fails, STOP and report — no fallback.
- Emitting the verbatim changelog. That is what `/release-notes` is for.
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

Pure Bash — no model generation, no credit-gate risk. Parse `latest` and
`dates`. If curl/jq fails, STOP with the error.

### 4. Compute the delta set (no network)
Start point:
- **Bare number `N`** → every version whose publish date ≥ (`latest`'s date
  − `N` days). Normal run; checkpoint WILL move (step 7).
- else if checkpoint exists → every version `>` `last_version_seen`.
- else (bootstrap) → every version within the last 8 days of `latest`'s date.

Compare versions numerically by component.

**No-op fast path — checkpoint mode only, if `last_version_seen ≥ latest`:**
emit EXACTLY these two lines and nothing else, then go to step 7 (refresh
date) and STOP:

```
No new releases since <last_checked> (latest is still v<latest>).
Installed: v<x> · Changelog: v<latest>
```

(Omit line 2 only if `claude --version` errors.) One fact, stated once. Any
sentence beyond these two lines is a skill violation.

### 5. Delegate curation to a Sonnet sub-agent
Spawn one Agent tool call:

- `subagent_type`: `general-purpose`
- `model`: `sonnet`
- `description`: `Curate Claude Code release digest`
- `prompt`: use the template below, filling in placeholders

Prompt template (NETWORK CALL 2 happens inside):

```
You are curating a Claude Code release digest. You run on Sonnet in a fresh
context to keep the parent session clean of the ~700-line verbatim payload.

WORLD-CLASS STANDARD: ruthless curation, no padding, no completeness theater.
The user's complaint is verbosity — they see this output AS-IS and want only
items they actually care about. Cutting too much is forgiven; including a
"who-uses-this" item is the failure mode.

Inputs from the orchestrator:
- delta_versions:  <comma-separated, newest first, e.g. "A.B.C,A.B.D">
- dates:           <JSON map: {"A.B.C":"YYYY-MM-DD", ...}>
- latest:          <e.g. "A.B.C">
- start_version:   <oldest in delta>
- start_date:      <date of start_version>
- latest_date:     <date of latest>
- installed:       <output of `claude --version`, or "" if it errored>

Do ONE WebFetch to
https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md
asking for every bullet, verbatim and complete, for every version in
delta_versions. You read these to decide what to include — you do NOT emit
them. If the fetch fails, return exactly: `ERROR: changelog fetch failed —
<reason>` and stop. No fallback to aggregator sites.

Then apply this STRICT curation filter. Hard cap: 8 items total across the
whole digest. Aim for 4–6.

INCLUDE (a change qualifies only if it clearly hits one of these):
- Model behavior changes — Opus/Sonnet/Haiku/`/fast`/effort/context-window.
- Default changes to commands a typical user invokes interactively
  (`/model`, `/resume`, `/clear`, `/plugin`, `/branch`, `/loop`, common
  slash commands).
- Renamed or removed commands (muscle-memory shift).
- Major new top-level features — a new view, a new mode, a new top-level
  command. NOT new flags on existing commands.
- Architecture-level changes to the skill / hook / MCP / subagent systems
  that change how someone builds with them.
- Security or auth changes that affect everyone.
- A bug fix ONLY IF it was a regression that broke daily-driver functionality
  (startup hang, common crash, a slash command suddenly not working) and the
  changelog itself frames it that way.

EXCLUDE — every one of these is an automatic cut, no exceptions:
- Niche subcommands and flags most users never type: `claude respawn`,
  `claude --bg --name`, `claude agents --json`, `claude plugin validate`,
  `claude logs <id>`, etc.
- Bug fixes with narrow scope ("Fixed X when Y happens in Z environment").
- Internals: OTEL spans, daemon behavior, file-descriptor exhaustion,
  GraphQL query updates, cache/lock-file/encoding fixes.
- Platform-edge fixes (Windows-specific, WSL, network-drive, CJK rendering,
  PowerShell version-specific) unless cross-platform impact is explicit.
- Plugin-author / MCP-author internals.
- Cosmetic UI fixes: color bleeds, ghost characters, spinner color counts,
  table-wrapping artifacts, fullscreen-only quirks.
- "Improved X" performance items under ~10% impact.
- Restoring/re-enabling obscure features after a regression.
- Anything where the answer to "would I notice if this didn't ship?" is no.

If you find yourself listing more than 8 items, you are inventorying — re-cut
until ≤8 survive the bar.

Worked examples of the bar (from real changelogs):

KEEP:
- "Fast mode now uses Opus 4.7 by default (previously Opus 4.6)" — model
  behavior, daily impact.
- "/model now changes model for current session only — press d to set a new
  default" — muscle-memory shift on a daily-used command.
- "/extra-usage renamed to /usage-credits" — renamed command.
- "Startup hang on unreachable api.anthropic.com fixed (was hanging up to
  75s)" — daily-driver regression with clear scope.
- "Plugins with root-level SKILL.md and no skills/ subdir now surfaced as a
  skill" — architecture-level change to the skill system.

CUT:
- "claude respawn <id> on stopped session now actually respawns" — niche
  subcommand.
- "Fixed ghost characters at left edge in Agent View on Windows Terminal
  with CJK content" — platform-edge cosmetic.
- "OTEL tracing improved: agent_id / parent_agent_id on spans" — internal.
- "claude agents --json flag added for scripting" — niche flag, scripting-
  only.
- "Fixed file descriptor exhaustion in skill-dir builds" — narrow internal.
- "MCP servers with paginated tools/list responses now return all pages" —
  MCP-author concern, not user-facing.
- "Fixed Bedrock and Vertex unable to select Opus (1M context) in /model
  picker" — narrow platform regression, niche to enterprise users.

Emit your FINAL message in this exact shape — no preface, no commentary,
no closing remarks:

What's new — v<start_version> (<start_date>) → v<latest> (<latest_date>)
(For the full verbatim list, run /release-notes.)

- **<short label>:** <one tight line, plain English, ≤25 words>
- **<short label>:** <one tight line>
...

Installed: v<installed> · Latest: v<latest>
(Omit the final line entirely if `installed` is empty.)

Return only the digest. The orchestrator will relay your message verbatim.
```

### 6. Relay the sub-agent's final message
Output the sub-agent's final message exactly as returned. No preface, no
trailing summary, no commentary. The user sees only the curated digest.

### 7. Advance the checkpoint
Unless step 2 aborted on malformed JSON, write
`~/.claude/state/release-notes-checkpoint.json` with
`last_version_seen = latest`, `last_checked = today`,
`latest_at_last_check = latest`. Create `~/.claude/state/` if absent.

</process>

<notes>
- User-level skill → globally available in every project.
- The skill's value is the curation, not the source data. `/release-notes` is
  always the verbatim path. Don't reintroduce verbatim emission here.
- Sub-agent is the explicit shield against the session's 1M-context credit
  gate AND keeps the verbatim payload out of the orchestrator's context.
  Don't remove it.
- Two network calls, fixed: npm registry (main), one CHANGELOG fetch (in
  sub-agent). Curated output only — no files, no verbatim.
</notes>
