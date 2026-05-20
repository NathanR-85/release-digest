# Release Digest

A short, curated summary of what's changed in Claude Code since the last time you looked.

The built-in `/release-notes` shows the entire changelog every time. This one remembers where you left off and only shows what's new — split into Features and Bugs, with the most important things first.

## Usage

```
/release-digest          # only what's new since you last ran it
/release-digest 7        # refresher: everything from the last 7 days
```

The first time you run it, it shows the last 8 days. After that, the bare command shows only what's new. 

Add a number when you want a refresher window — it'll show you the last N days even if you've already seen some of it. 

Every run updates "the last time you looked."

## Install

In Claude Code, ask it to install this skill from this repo. Tell it **user-level** if you want the command available in every project, or **project-level** if you only want it in one.

## License

MIT.
