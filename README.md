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

- journal_dir: journal/                      # or an absolute path to a journal shared by several repos
- journal_index: journal/README.md           # one-line-per-day index; omit to skip
- journal_repos: ../other-repo,../another    # repos that keep their own journal/
- ticket_system: linear                      # or: github, none
- ticket_prefix: ACME-
- lock_status_cmd: scripts/session-lock.sh status
- lock_check_cmd: scripts/session-lock.sh check
- lock_claim_cmd: scripts/session-lock.sh claim
- sync_sweep_cmd: scripts/repo-sync.sh --dry-run
- sync_apply_cmd: scripts/repo-sync.sh
- deploy_cmd: none                           # /handoff deploys only when this is set
- deploy_gap_cmd: none                       # read-only: prints merged-but-undeployed commits
- prod_paths: src/**,deploy/**
- memory_dir: default
```

Any key you leave out takes the default shown in the skill. Each `skills/<name>/SKILL.md` opens with the subset it uses and what each key changes. The keys mean the same thing in every skill.

| key | standup | lane-reset | handoff |
|---|---|---|---|
| `journal_dir`, `journal_repos` | reads | — | appends |
| `journal_index` | — | — | writes one line |
| `ticket_system`, `ticket_prefix` | reads | comments | reconciles |
| `lock_check_cmd` | checks | — | checks |
| `lock_status_cmd`, `lock_claim_cmd` | — | — | decides closer / claims |
| `sync_sweep_cmd` | runs (read-only) | — | — |
| `sync_apply_cmd` | proposes only | — | — |
| `deploy_cmd`, `deploy_gap_cmd`, `prod_paths` | — | — | step 6, only if set |
| `memory_dir` | reads | — | writes |

## License

MIT.
