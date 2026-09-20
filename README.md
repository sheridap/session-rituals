# session-rituals

Three Claude Code skills that make a working session start, reset and close against **durable state** instead of the transcript: a journal on disk, the project's auto-memory, your ticket system, and git.

| Command | When | What it does |
|---|---|---|
| `/standup` | New day, or after a multi-hour break | Reads the latest journal entry, relevant memories, open tickets and git divergence, then hands you a short briefing and one recommended first action. |
| `/lane-reset` | A ticket is done and the next one is unrelated | Commits and pushes the finished ticket, leaves one ticket comment, emits a small paste-able resume prompt, then you `/clear` and keep going. |
| `/handoff` | End of the session | Commits and pushes, memory notes, ticket updates and a journal arc — the artifacts `/standup` reads back next time. Once per day, one session also writes the index line, runs the deploy if you configured one, and emits the resume prompt. |

`/standup` and `/handoff` are mirrors: one writes the four artifacts, the other regenerates a briefing from them.

## Install

```
/plugin marketplace add sheridap/session-rituals
/plugin install session-rituals@session-rituals
```

Updates arrive with `/plugin marketplace update session-rituals`.

## What it assumes

`git`, and that Claude Code loads `CLAUDE.md` from the repo you are working in. Everything else is optional and off by default:

- **No deploy step** runs unless you configure `deploy_cmd`, and even then `/handoff` states what it will deploy and asks first.
- **No ssh** to any host, ever. The skills never compose a remote command; a remote deploy is only whatever you put in `deploy_cmd`.
- **No ticket system** is required. With `ticket_system: none` the rituals lean on git and the journal.
- `gh` is used only when it is installed and the repo's `origin` is on github.com; otherwise PR checks report "not checked".
- Linear MCP tools are used only with `ticket_system: linear`.
- Claude Code auto-memory is used only if a memory index already exists; otherwise the memory steps are skipped and say so.
- `/lane-reset` and `/handoff` **commit and push** (push defaults to asking once). `/standup` never mutates.
- `/handoff` and `/lane-reset` are operator-invoked only (`disable-model-invocation`); casual phrases like "end of day" never trigger them.

## Parameters

Each skill opens with a **Parameters** block listing the keys it uses, with a safe default. You do not edit the plugin. You override values in the `CLAUDE.md` at the root of the repo you are working in, under this heading. If several `CLAUDE.md` files load (user-level, repo root, a nested directory, `CLAUDE.local.md`), the one closest to your working directory wins per key.

```markdown
## Session rituals

- journal_dir: journal/                      # or an absolute path to a journal shared by several repos
- journal_index: journal/README.md           # one-line-per-day index; omit to skip
- journal_repos: ../other-repo,../another    # repos that keep their own journal/
- lane_name: api                             # default: the repo directory's basename
- ticket_system: linear                      # or: github, none
- ticket_prefix: ACME-
- ticket_team: ACME                          # Linear team key; default: ticket_prefix without the dash
- lock_status_cmd: scripts/session-lock.sh status
- lock_check_cmd: scripts/session-lock.sh check
- lock_claim_cmd: scripts/session-lock.sh claim
- lock_release_cmd: scripts/session-lock.sh release
- sync_sweep_cmd: scripts/repo-sync.sh --dry-run
- sync_apply_cmd: scripts/repo-sync.sh
- commit_branch: current                     # or: ticket
- push: ask                                  # or: always, never
- deploy_cmd: none                           # /handoff deploys only when this is set, after confirming
- deploy_gap_cmd: none                       # read-only: prints merged-but-undeployed commits
- prod_paths: src/**,deploy/**
- memory_dir: default
```

Any key you leave out takes the default shown in the skill. The keys mean the same thing in every skill.

| key | standup | lane-reset | handoff |
|---|---|---|---|
| `journal_dir`, `journal_repos`, `lane_name` | reads | — | appends |
| `journal_index` | — | — | writes one line |
| `ticket_system`, `ticket_prefix`, `ticket_team` | reads | comments / closes | reconciles; proposes follow-ups |
| `lock_check_cmd` | checks | — | checks before a cross-lane append |
| `lock_status_cmd`, `lock_claim_cmd`, `lock_release_cmd` | — | — | decides closer; claims and releases |
| `sync_sweep_cmd` | runs (read-only) | — | — |
| `sync_apply_cmd` | proposes only | — | — |
| `commit_branch`, `push` | — | commits / pushes | commits / pushes |
| `deploy_cmd`, `deploy_gap_cmd`, `prod_paths` | — | — | step 6, only if set, after confirming |
| `memory_dir` | reads | — | writes |

### Lock command contract

If you set the lock keys, your script must follow this shape. `check <repo>` exits 0 when no *other* live session holds the repo and non-zero when one does, printing the holders; it must exclude the calling session. `status` prints one line per live holder, tab-separated: repo path, ISO-8601 claim time, session id, and a trailing tab plus `*` on the calling session's own rows. `claim <repo>` and `release <repo>` do what they say. How your script identifies a session (pid, tty, an env var) is up to you; the skills only read the exit code and the `*` mark.

### Journal contract

`<journal_dir>/YYYY-MM-DD.md`, one `## <lane_name> lane` section per repo that worked that day, with `### Shipped`, `### Decisions`, `### Learnings`, `### Deferred`, `### Next` under it. `/standup` reads `### Next`. A repo that journals in its own tree leaves a one-line stub in the shared journal so the day's census stays complete. The optional index line is `- **YYYY-MM-DD** — summary`, newest first, under `### Recent entries` in `journal_index`.

## License

MIT.
