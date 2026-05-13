# Grass skill for Claude Code

A Claude Code skill that dispatches tasks to a [Grass](https://grass.dev) remote sandbox VM instead of running them in the local session.

## What it does

When you ask your coding agent to do something "using grass" (or "on grass", "via grass", "with grass", etc.), the skill:

1. Extracts the task from your message.
2. Reads your bearer token from `~/.grass/auth`.
3. Detects the current repo (from `origin`) and branch (from `git`).
4. POSTs the task to the Grass dispatch API.
5. Tells you to check the Grass mobile app for results.

It does **not** run the task locally — submission is the whole job.

## Install

Copy the `grass/` folder into a Claude Code skills directory:

```bash
# Personal (available in every project)
mkdir -p ~/.claude/skills
cp -r grass ~/.claude/skills/

# OR project-local (only this repo)
mkdir -p .claude/skills
cp -r grass .claude/skills/
```

The skill is now available. Restart Claude Code if it's already running.

Other Agent Skills–compatible tools follow the same pattern: drop the `grass/` directory into wherever that tool reads skills from.

## Prerequisites

- A Grass account and a token saved at `~/.grass/auth` (the file's contents are the bearer token, no extra formatting).
- A git checkout with an `origin` remote pointing at GitHub (the API needs `owner/repo`).
- `curl` and `jq` on `PATH`.

## Configuration

The skill has no configurable endpoints. It only ever contacts `https://api.codeongrass.com` over HTTPS — the host is hardcoded in the skill and cannot be overridden by env vars, config files, or user input. See the "Security & network" section in [`skills/grass-dispatch/SKILL.md`](skills/grass-dispatch/SKILL.md) for the full disclosure.

## Use

Just ask normally and add a Grass hint:

```
Run a thorough security audit of this repo. Check all callsites. Do it using grass.
```

```
/grass refactor the auth middleware to use the new session store
```

You should see a one-line confirmation that the task was submitted. Open the Grass mobile app to follow progress.
