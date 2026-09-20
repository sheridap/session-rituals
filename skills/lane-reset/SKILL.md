---
name: lane-reset
description: Cheap ticket-boundary context reset — bank the current ticket to a durable state, emit a small paste-able resume prompt, then /clear and keep working. Not an end-of-day ritual; work continues immediately. Invoke as "lane reset", "reset the lane", "ticket boundary", or when a ticket is done and the next one is unrelated.
---

# /lane-reset — Ticket-boundary context reset

**This does not stop work.** It resets the *transcript*, not the session. You bank the ticket, `/clear`, paste the resume prompt, and the next ticket starts in the same breath. Zero downtime. That is the whole point — see § Why below.

`/handoff` is the session-close ritual (four artifacts × every repo touched). `/standup` is its heavy read-side mirror. Both are expensive, and that expense is *why* sessions run all day: the only sanctioned way to stop was costly, so nobody stopped. This is the cheap one. Use it many times a day.

## Parameters

Override per repo in that repo's `CLAUDE.md` under a `## Session rituals` heading, one `- key: value` line each. Absent keys take the default. This skill uses only these:

| key | default | what it changes |
|---|---|---|
| `ticket_system` | `none` | `linear` (Linear MCP tools), `github` (`gh issue comment` / `gh issue close`), or `none` (step 2 becomes a line in the commit message instead). |
| `ticket_prefix` | none | Issue-id prefix (e.g. `ACME-`) used in commit messages and the resume prompt. |

**Never assumed:** no journal, no cross-repo sweep, no memory audit, no deploy, no ssh. Those belong to `/handoff`.

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

- `git status --short` in **this repo only**. Review the diff before committing — never commit unreviewed. If your project runs an adversarial pre-commit review on code diffs, it applies here.
- **Stage explicit paths only** — never `git add -A` / `.` / `-a`. A shared working tree may hold another session's uncommitted edits; blanket staging sweeps them into your commit.
- Commit on a ticket branch; push.
- If the work isn't in a committable state, say so plainly and **do not reset** — an uncommitted lane cannot survive a `/clear`.

### 2. Land the state in the ticket system (one comment, short)

- `linear` / `github`: one comment on the ticket — what landed, commit SHA, what's next. If the ticket is done, close it with a resolution note. Keep it plain prose; some ticket APIs reject long or richly formatted bodies.
- `none`: the commit message carries the same three facts. Nothing else.

Skip the journal. Skip the memory sweep. Those are `/handoff`'s job, once a session.

### 3. Emit the resume prompt

Target **under ~1,500 tokens.** It is a pointer, not a summary — the durable artifacts (commits, tickets, decision records, code) hold the truth; the prompt just tells the next context where to look and what to do first.

Include, and nothing more:
- The next ticket ID and one line on the goal.
- The repo and branch to work in.
- Any *non-obvious* state the next context cannot re-derive from git + tickets (a gotcha hit, a decision made and why, a dead end already ruled out). If everything is re-derivable, say so and keep the prompt short — a short prompt is a success, not a gap.
- The concrete first action.

**No line prefixes.** Never prepend `>`, pipes, or decoration to the prompt lines — use a fenced code block or plain paragraphs so copy-paste is clean.

Surface any usage gotcha **in the same message as the prompt**, not after the operator trips on it.

### 4. Hand off the reset

Close with the two-step, in this order and as the last thing in the message:

1. `/clear`
2. paste the prompt

Then the operator is working again immediately. The pasted prompt **is** the next ticket's bootstrap — there is no `/standup` at a ticket boundary; `/standup` is the cold-start tool (new day, multi-hour break), and running it at every seam re-incurs the cost this reset just avoided. Do not editorialize about stopping, resting, or resuming later — nothing is being deferred. The operator decides when to step away; this skill never raises it.

## What this skill must never become

- A second `/handoff`. If it grows a journal step, a cross-repo sweep, or a memory audit, it stops being cheap and stops getting used — and then the bulk of the spend goes unaddressed again.
- An excuse to reset mid-problem. A reset that drops load-bearing reasoning costs more than the tokens it saves.
