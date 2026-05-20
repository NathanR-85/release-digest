# Release Digest

A short, curated summary of what's changed in Claude Code since the last time you looked.

The built-in `/release-notes` shows the entire changelog every time. This one remembers where you left off and only shows what's new — split into Features and Bugs, with the most important things first.

## Usage

```
/release-digest          # only what's new since you last ran it
/release-digest 7        # everything from the last 7 days
```

The first time you run it, it just marks where you are now and exits without showing anything. From the next run on, you'll only see what's new. If you want a digest right away, type a number after the command (like `/release-digest 7`) — that's how many days back to look.

## Install

In Claude Code, ask it to install this skill from this repo. Tell it **user-level** if you want the command available in every project, or **project-level** if you only want it in one.

Your run history is stored locally and never published.

## License

MIT.
