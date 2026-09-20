---
name: lane-reset
description: Cheap ticket-boundary context reset — bank the current ticket to a durable state, emit a small paste-able resume prompt, then the operator runs /clear and keeps working. Not an end-of-day ritual; work continues immediately. Operator-invoked only.
disable-model-invocation: true
---

# /lane-reset — Ticket-boundary context reset

**This does not stop work.** It resets the *transcript*, not the session. You bank the ticket, the operator runs `/clear`, pastes the resume prompt, and the next ticket starts in the same breath. Zero downtime. That is the whole point — see § Why below.

`/handoff` is the session-close ritual (four artifacts × every repo touched). `/standup` is its heavy read-side mirror. Both are expensive, and that expense is *why* sessions run all day: the only sanctioned way to stop was costly, so nobody stopped. This is the cheap one. Use it many times a day.

## Parameters

Override in the `CLAUDE.md` at the root of the repo you are working in, under a `## Session rituals` heading, one `- key: value` line each. Absent keys take the default; the `CLAUDE.md` closest to `cwd` wins per key. This skill uses only these:

| key | default | what it changes |
|---|---|---|
| `ticket_system` | `none` | `linear` (Linear MCP tools, which must be connected), `github` (`gh issue comment` / `gh issue close`, only when `gh` is installed and `origin` is on github.com), or `none` (step 2 becomes a line in the commit message instead). |
| `ticket_prefix` | none | Issue-id prefix (e.g. `ACME-`) used in commit messages, branch names and the resume prompt. |
| `commit_branch` | `current` | `current` = commit on whatever branch is checked out. `ticket` = if on the default branch, create `<ticket-id-lowercased>-<slug>` first. |
| `push` | `ask` | `ask` = ask once before pushing; `always` = push without asking; `never` = commit only and say the push is owed. |

**Mutations this skill makes:** a git commit; a git push if `push` permits; one ticket comment or close if `ticket_system` is not `none`. Nothing else — no journal, no memory, no cross-repo sweep, no deploy, no ssh. Those belong to `/handoff`.

## Conventions shared with `/standup` and `/handoff`

- **Resume prompt.** Under ~1,500 tokens. A pointer, not a summary. Contains: the next ticket id + one line on the goal; the repo and branch; any *non-obvious* state the next context cannot re-derive from git + tickets; the concrete first action. **No line prefixes** — a fenced code block or plain paragraphs, never `>` or pipes, so copy-paste is clean. `/handoff` emits the same shape for the whole day.

## Why (read this before "optimizing" it away)

Measured on one long-running deployment from the API's own usage fields: the interactive main loop was **about 89% of all cache reads**, at an average context of roughly **400k tokens**. The fixed preamble (system prompt, `CLAUDE.md`, memory index, tool schemas) was only about a tenth of that. **The rest is accumulated transcript**, and every API call re-reads all of it.

There is **no supported way to force earlier auto-compaction** — no settings key, no env var, no CLI flag — and with million-token context windows it effectively never fires on its own. So transcript length is not self-limiting, and **session discipline is the only lever on the bulk of the spend.**

A transcript that never resets grows superlinearly in cost: each new turn is billed against every turn before it. Resetting at a natural boundary — where the next ticket doesn't need the last one's context anyway — is nearly free in information and enormous in cost.

## When to invoke

Invoke at a **genuine boundary**, where the next work doesn't need this work's transcript:

- A ticket is done (committed, pushed, ticket updated) and the next one is unrelated.
- You're switching repos or lanes.
- A long investigation concluded and its findings are now written down somewhere durable.
- The context is fat with spent material — big file reads, screenshots, a long debugging trail — and none of it is load-bearing for what's next.

**Do not** invoke mid-ticket, mid-debug, or when the next step depends on reasoning that only exists in the transcript. Write it down first, or don't reset.

## Procedure

Cheap by construction: **one repo, one ticket, no journal, no cross-repo sweep, no memory audit.** If you find yourself doing those, you're running `/handoff` — stop and say so.

### 1. Bank the work (durable state)

- `git status --short` in **this repo only**. Review the diff before committing — never commit unreviewed. If your project has a pre-commit review step (a reviewer agent, a checklist), run it on code diffs here.
- **Stage explicit paths only** — never `git add -A` / `.` / `-a`. A shared working tree may hold another session's uncommitted edits; blanket staging sweeps them into your commit.
- Commit per `commit_branch`; push per `push`.
- If the work isn't in a committable state, say so plainly and **do not reset** — an uncommitted lane cannot survive a `/clear`.

### 2. Land the state in the ticket system (one comment, short)

- `linear` / `github`: one comment on the ticket — what landed, commit SHA, what's next. If the ticket is done, close it with a resolution note. Keep it plain prose; some ticket APIs reject long or richly formatted bodies. Never *create* a ticket here; if a follow-up is needed, put it in the resume prompt for `/handoff` to file.
- `none`: the commit message carries the same three facts. Nothing else.

Skip the journal. Skip the memory sweep. Those are `/handoff`'s job, once a session.

### 3. Emit the resume prompt

Per the shared convention above. If everything is re-derivable from git + tickets, say so and keep the prompt short — a short prompt is a success, not a gap. Surface any usage gotcha **in the same message as the prompt**, not after the operator trips on it.

### 4. Hand off the reset

Close with the two-step the **operator** performs, in this order and as the last thing in the message:

1. the operator runs `/clear`
2. the operator pastes the prompt

Only the operator can run `/clear`; never try to emit it as a command. Then they are working again immediately. The pasted prompt **is** the next ticket's bootstrap — there is no `/standup` at a ticket boundary; `/standup` is the cold-start tool (new day, multi-hour break), and running it at every seam re-incurs the cost this reset just avoided. Do not editorialize about stopping, resting, or resuming later — nothing is being deferred. The operator decides when to step away; this skill never raises it.

## What this skill must never become

- A second `/handoff`. If it grows a journal step, a cross-repo sweep, or a memory audit, it stops being cheap and stops getting used — and then the bulk of the spend goes unaddressed again.
- An excuse to reset mid-problem. A reset that drops load-bearing reasoning costs more than the tokens it saves.
