# session-rituals

Three Claude Code skills that make a working session start, reset and close against **durable state** instead of the transcript: a journal on disk, the project's auto-memory, your ticket system, and git.

| Command | When | What it does |
|---|---|---|
| `/standup` | New day, or after a multi-hour break | Reads the latest journal entry, relevant memories, open tickets and git divergence, then hands you a short briefing and one recommended first action. |
| `/lane-reset` | A ticket is done and the next one is unrelated | Banks the finished ticket to a durable state, emits a small paste-able resume prompt, then you `/clear` and keep going. |
| `/handoff` | End of the session | Commits, memory notes, ticket updates and a journal arc — the artifacts `/standup` reads back next time. |

`/standup` and `/handoff` are mirrors: one writes the four artifacts, the other regenerates a briefing from them.

## Install

```
/plugin marketplace add sheridap/session-rituals
/plugin install session-rituals@session-rituals
```

Updates arrive with `/plugin marketplace update session-rituals`.

## What it assumes

Only git. Everything else is optional and off by default:

- **No deploy step** runs unless you configure a deploy command.
- **No ssh** to any host, ever, unless you configure one.
- **No ticket system** is required; without one the rituals lean on git and the journal.

## Parameters

Each skill opens with a **Parameters** block that lists every environment-specific value it uses, with a safe default. You do not edit the plugin. You override values in the `CLAUDE.md` of the repo you are working in, under a heading the skills look for:

```markdown
## Session rituals

- journal_dir: journal/
- ticket_system: linear        # or: github, none
- ticket_prefix: ACME-
- lock_check_cmd: scripts/session-lock.sh check
- sync_sweep_cmd: scripts/repo-sync.sh --dry-run
- deploy_cmd: none
- memory_dir: default
```

Any key you leave out takes the default shown in the skill. See each `skills/<name>/SKILL.md` for the full list and what each key changes.

## Status

`/standup` is generalized. `/lane-reset` and `/handoff` follow.
