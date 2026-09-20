---
name: handoff
description: End-of-session handoff protocol — Part A per lane (commits ∧ memories ∧ tickets ∧ journal arc) in the repos that lane touched; Part B once for the day (journal index line, deploy if configured, resume prompt) by a single closer. Operator-invoked only, when they explicitly signal session-close.
disable-model-invocation: true
---

# /handoff — End-of-session handoff

Closes a working session to a durable state, and — once per day — closes the day.

**Two completion bars, don't conflate them.** *Your lane* is complete when Part A's artifacts are committed and pushed, your arc included. *The day* is complete when one session has additionally run Part B. A lane that finishes Part A and pushes is done and may stop, whether or not the day has closed.

The journal is the artifact most often skipped — it has no natural surfacing the way `git status`, the memory index, and ticket queries do. Do not claim your lane complete until your arc is written, committed, and pushed — and never claim *the day* complete unless you actually ran Part B.

> **Operator-protocol note:** this skill runs *only* when the operator explicitly signals session-close. It closes the loop they asked to close — it never suggests closing one.

## Parameters

Override in the `CLAUDE.md` at the root of the repo you are working in, under a `## Session rituals` heading, one `- key: value` line each. Absent keys take the default; the `CLAUDE.md` closest to `cwd` wins per key.

| key | default | what it changes |
|---|---|---|
| `journal_dir` | `journal/` relative to the repo root | Where your arc is appended, as `YYYY-MM-DD.md`. May be an absolute path to a **shared journal**. If that path is inside a git repo, the skill commits and pushes *in that repo*; if it is not inside any git repo, the skill appends the file and does no git operation there. |
| `journal_index` | none | A file holding a one-line-per-day index. When unset, Part B step 5 is skipped. |
| `journal_repos` | none | Comma-separated paths (absolute, or relative to the repo root) of other local repos that keep their own `journal/`. Used to census lanes in Part B. |
| `lane_name` | the repo's directory basename | The name in your section heading, `## <lane_name> lane`. |
| `ticket_system` | `none` | `linear`, `github`, or `none`. Controls step 3. |
| `ticket_prefix` | none | Issue-id prefix used to recognise ticket references in commits. |
| `lock_status_cmd` | none | Prints one line per live session holding any repo lock. When unset, **this session is assumed to be the only one** and runs Part A then Part B. |
| `lock_check_cmd` | none | `<cmd> <repo-path>` — exits 0 when no *other* live session holds that repo, non-zero when one does. Run before any cross-lane write. |
| `lock_claim_cmd` | none | `<cmd> <repo-path>` — claims a repo lock before an authorised cross-lane append. |
| `lock_release_cmd` | none | `<cmd> <repo-path>` — releases a lock. This skill releases only locks *it* claimed in this run, never the session's own. |
| `commit_branch` | `current` | `current` = commit on the checked-out branch. `ticket` = if on the default branch, create `<ticket-id-lowercased>-<slug>` first. |
| `push` | `ask` | `ask` = ask once before the first push of this run, then push everything; `always` = push without asking; `never` = commit only and say the push is owed. A rejected push is reported as "arc committed, not durable" and the self-check fails. |
| `deploy_cmd` | none | The command that deploys the production surface. **When unset, Part B step 6 performs no deploy and prints one line saying so.** The skill never composes an ssh command; if your deploy runs remotely, put the whole invocation in this value. |
| `deploy_gap_cmd` | none | A read-only command that prints the commits merged but not yet deployed. When unset and `deploy_cmd` is set, the skill asks the operator whether anything prod-affecting merged today instead of guessing. |
| `prod_paths` | none | Comma-separated path globs that count as prod-affecting. When `deploy_cmd` is set and this is unset, every non-journal commit counts. |
| `memory_dir` | `default` | `default` = Claude Code's auto-memory directory for the current project. |

**Assumed tools:** `git`. `gh` only when installed and `origin` is on github.com. Linear MCP tools only when `ticket_system: linear`.

**Every mutation this skill can make, in full:** git commits; git pushes (per `push`); `git pull --rebase` only to recover a rejected journal push; memory files under `memory_dir`; ticket status updates and comments; **new** follow-up tickets, each confirmed with the operator before creation; a lock claim and its release (per `lock_claim_cmd` / `lock_release_cmd`); and `deploy_cmd`, run only after the operator confirms in this run. No ssh is ever composed. Nothing outside the current repo, the journal's repo, `memory_dir` and the ticket system is touched.

## Conventions shared with `/standup` and `/lane-reset`

- **Date.** `D` is the date the harness injected at session start, if it injects one; otherwise `date +%F` run once at the *first* ritual invocation this session and reused. A lane wrapping after midnight still writes `D`'s day. Never re-derive `D` mid-ritual.
- **Journal layout.** `<journal_dir>/D.md`; one `## <lane_name> lane` h2 per repo, h3 sections under it (see step 4). A repo that journals in its own tree leaves a one-line stub in the shared journal: `## <lane_name> lane — journaled in <repo>/journal/D.md (<sha>)`. The stub is an h2 so the census counts it.
- **Index line.** In `journal_index`, under a `### Recent entries` heading (create it if absent), newest first: `- **D** — <one bold-free sentence naming every lane's headline>`. To amend, find the line whose prefix is `- **D**` and replace it; never add a second line for `D`.
- **Memory layout.** `<memory_dir>/MEMORY.md` is the index, one line per fact; each fact is its own file with frontmatter `name`, `description`, `type` (`feedback` / `reference` / `project`). If no index exists and `memory_dir` is `default`, step 2 is skipped and says so.
- **Resume prompt.** Under ~1,500 tokens; a pointer, not a summary; no line prefixes (fenced block or plain paragraphs, never `>` or pipes). The next morning the operator pastes it as the first message and then runs `/standup`.

## Concurrency — decide this first

Operators often run several lanes at once, one Claude Code session per repo. Some steps are per-session, and three are **once-per-day singletons** no matter how many lanes ran.

| | steps | why |
|---|---|---|
| **Part A — lane-local** | commits · memories · tickets · **your journal arc, appended yourself** | per-repo, per-fact, per-ticket, per-append — safe in parallel |
| **Part B — the closer** | journal index line · deploy · resume prompt | *one* line, *one* prod surface, *one* prompt the operator reads |

**Every session runs Part A. Exactly one session additionally runs Part B.** Decide this yourself — do not ask the operator which parts to run:

- **`lock_status_cmd` unset** → you are the only session. **A then B.**
- **`lock_status_cmd` set** → run it. Its output contract: one line per live holder, tab-separated `<repo-path>	<ISO-8601 claim time>	<session-id>`, with the **calling session's own rows marked by a trailing `	*`** (your script decides how it identifies sessions — pid, tty, an env var — the skill only reads the mark). Then:

| what it shows | run |
|---|---|
| **no rows without `*`** | **A then B.** A solo day needs no ceremony — close it and report the day done. |
| **other rows exist, and one claimed earlier than your earliest row** | **A only.** Report which lanes are still live and that a closer is still owed. |
| **other rows exist, but yours is the earliest** | **A**, then say you are the fallback closer and ask whether to close now or wait — closing early means deploying before their merges land. |

The operator can always override — "close the day", "run Part B", "you're the closer" — and an explicit instruction beats the table.

**Write your own arc — do not hand it to anyone.** An **append** is safe where a whole-file write is not: two appends either coexist or conflict *loudly*, and neither silently loses text. A whole-file write silently wins. So the journal *file* is not a single-writer resource; the genuine singletons are the **index line**, the **deploy**, and the **resume prompt**. An arc that lives only in a hand-off message exists in two transcripts and nowhere durable.

**Where your arc goes.** Derive it, don't memorise it. `JD` is `journal_dir` resolved to an absolute path:

```bash
for j in "$(git rev-parse --show-toplevel)/journal" "$JD" <each journal_repos entry>/journal; do
  [ -d "$j" ] && printf '%-40s last entry %s\n' "$j" \
    "$(ls "$j"/2*.md 2>/dev/null | sort | tail -1 | xargs -I{} basename {} .md)"
done
```

- **This repo keeps a *live* journal** (its own `journal/` with an entry in the last few weeks) → write there. No contention.
- **It does not, or its tree is months stale** → write to `JD`, the shared journal. A dormant `journal/` is a trap: writing into a tree nobody reads makes your arc invisible; treat it as absent and say so.

**Who closes Part B:** the operator names the closer at EOD. Absent a call: **the earliest claim time among all live rows closes** — chosen for determinism, not seniority. **If you are that lane, you are the closer. Do not wait for anyone.** A lane finishing after you appends its own arc, amends the index line, and deploys its own work (step 4's late-arrival clause).

**Lock age is not staleness.** `/lane-reset` resets the transcript, not the session, so a lane can stay claimed for days while working normally. You cannot tell from the lock table whether a lane is still working; if you need to know, ask the operator. Never release a lock you did not claim in this run.

**Cross-lane writes.** This skill authorises **two** narrow writes into a repo another lane owns, and nothing else: (1) your arc or stub, appended to the shared journal; (2) the index line, if you close while anchored elsewhere. Both follow one rule: **if `lock_check_cmd` is set, run it on that repo; if it reports a holder, `lock_claim_cmd` that repo (you become a co-holder, which is fine for an append); stage only that one path; ignore every other dirty path in that tree; `lock_release_cmd` it when done.** If no lock commands are configured, the same rule applies without the lock steps.

The three failures this structure prevents, each seen in practice: two lanes overwriting the same journal file (one arc lost); two index entries for one day; and a deploy nobody ran because a global step sat inside a per-lane checklist — everyone's job, therefore nobody's.

---

# Part A — lane-local (every session runs this)

**Reconstruct the day from durable sources, not from what this transcript remembers.** If `/lane-reset` ran earlier today, this context knows only its last resume prompt. So before step 0, gather: `git log --since="$D 00:00" --author="$(git config user.name)" --oneline` in the current repo (and in any repo the resume prompts named); tickets updated today if `ticket_system` is set; any arc already appended to today's journal. That list, not memory, is what steps 0–4 describe.

### 0. Scope — the repos *this lane* touched
List the repos this session actually worked in — the current repo plus any the reconstruction above names. Check them, and **only** them:

```bash
for d in <this lane's repos, not a glob>; do
  s=$(git -C "$d" status --short)
  a=$(git -C "$d" log --oneline @{u}.. 2>/dev/null)
  [ -n "$s$a" ] && echo "=== $d ===" && [ -n "$s" ] && echo "$s"; [ -n "$a" ] && echo "ahead: $a"
done
```

**Do not sweep every repo on the machine.** A concurrent lane's dirty tree is not yours to report or clean. If you believe another repo needs attention, say so to the operator — do not touch it.

### 1. Commits — every change committed + pushed
For each dirty/ahead repo:
- Review the diff before committing — never commit unreviewed. If your project has a pre-commit review step (a reviewer agent, a checklist), run it on code diffs here.
- **Stage explicit paths only — never `git add -A`/`.` or `git commit -a`.** The working tree may hold a concurrent session's uncommitted edits; blanket staging sweeps them into your commit under the wrong message. Confirm every staged path is something *this* session changed.
- One commit, one repo — never mix repos in a single commit or branch.
- Commit per `commit_branch`; push per `push`.
- End state: `git status --short` shows only paths matched by `.gitignore`-style runtime files you deliberately leave (say which). Nothing else uncommitted.

### 2. Memories — capture learnings
- Skip, and say so, if there is no memory index (see conventions).
- Write new `feedback` / `reference` / `project` memories for anything non-obvious learned today (corrections, gotchas, reference procedures) — drawn from today's commits and tickets, not only from this transcript. One fact per file, with frontmatter. Add a one-line pointer to `MEMORY.md`.
- Update or delete any memory this session proved stale. Don't duplicate — check for an existing file first. Skip anything the repo already records.

### 3. Tickets — reconcile state
- `none`: skip; the commits and the journal arc carry status.
- `linear` / `github`: update status on every ticket touched today. Reconcile today's commits against open tickets — close what shipped, comment progress on what advanced. For deferred items, **propose** follow-up tickets and create each only after the operator confirms, so nothing lives only in the transcript. Keep bodies plain; some ticket APIs reject long or richly formatted content.

### 4. Your journal arc — write it yourself, now

**Append** it (never a whole-file write) to the journal the concurrency section routed you to. All git operations run *in the journal's own repo*, found with `git -C`, so this works whether the journal is in this repo or a shared one:

```bash
D=YYYY-MM-DD                                  # ← the session's D
J="$JD/$D.md"                                 # JD = the routed journal dir, absolute
JR=$(git -C "$JD" rev-parse --show-toplevel 2>/dev/null || true)   # empty if not in a git repo
( set -o noclobber; printf '# %s\n\n> One section per lane.\n\n' "$D" > "$J" ) 2>/dev/null || true
cat >> "$J" <<'ARC'
## <lane_name> lane

### Shipped
…
ARC
[ -n "$JR" ] && git -C "$JR" add "$J" && git -C "$JR" commit -m "docs(journal): <lane_name> arc for $D"   # then push per `push`
```

`noclobber` is load-bearing, not style. A plain `[ -f "$J" ] || printf … > "$J"` is a **truncating** write, and two lanes that both find the file absent will both run it — the second wiping the first's section. Under `noclobber` exactly one lane creates the file and the losers no-op.

Lanes sharing one checkout serialise on commits; what you may actually hit:
- **`fatal: Unable to create '.git/index.lock': File exists`** — another lane is mid-commit. Wait a moment and retry; it is contention, not corruption.
- **Your commit carrying another lane's section.** Fine — both belong on the default branch; say so in the message.
- **`git status` clean and your text already committed** — the other lane's `git add` swept it in. Confirm with `git -C "$JR" log -p -1 -- "$J"` before concluding anything was lost, then push.

*(Only if two checkouts are in play — a worktree, another machine — can a push be rejected. Then `git -C "$JR" pull --rebase`, resolve by **keeping both sections**, `git add "$J"`, `GIT_EDITOR=true git rebase --continue`, push. `GIT_EDITOR=true` matters: a bare `rebase --continue` opens an editor and hangs a session with no tty. **Never `--ours`, `--theirs`, or `push --force`** — each resolves the conflict by deleting a lane's record.)*

Once pushed, your arc is durable and **your lane is done** — you do not wait for the closer.

**If the day is already closed** (an index line for `D` exists), you inherit Part B for your own work: append as above, amend the index line for `D` — amend, never add a second — and run step 6 yourself if anything you landed is prod-affecting and `deploy_cmd` is set.

Structure — h2 for the lane, h3 for its sections, **always**, even on a one-lane day, so a day that gains a second lane later stays well-formed:

```markdown
## <lane_name> lane

### Shipped        — tickets closed, decisions recorded, commits landed (ticket ids, short SHAs)
### Decisions      — calls made today; if your project keeps decision records (ADRs), reference the id
### Learnings      — gotchas, procedures, corrections worth a memory
### Deferred       — what got punted + where it's tracked
### Next           — top 1-3 priorities for this lane (what /standup reads tomorrow)
```

The ` lane` suffix in the heading is required — Part B's census greps for it. A time span after the name is optional.

If your record lives in this repo's own journal and a shared journal exists, append the one-line stub from the conventions to the shared journal, under the same cross-lane rule.

### Part A self-check
- [ ] Day reconstructed from git log / tickets / journal, not from transcript memory
- [ ] This lane's repos: working tree clean apart from deliberately-left ignored files. If you appended to a shared journal, that tree's *other* dirty paths are not yours — ignore them.
- [ ] This lane's repos: pushed (or `push: never` and the owed push named)
- [ ] Memories written + index updated, or "no index" stated
- [ ] Tickets updated; follow-ups proposed and, once confirmed, filed (or `ticket_system: none`)
- [ ] Arc **appended, committed and pushed** — in this repo's live journal, or the shared journal; stub left if the former
- [ ] Any lock this run claimed has been released

---

# Part B — closing the day (exactly one session)

Run this if you are the closer. **Do not wait for other lanes.** Count the lanes you can see, name them, and close.

### 5. Index the day — one line
Skip if `journal_index` is unset. Otherwise, account for every lane before writing it:

```bash
grep -n '^## .* lane' "$JD/$D.md"      # lanes that recorded here, stubs included
# plus the last-entry loop from the concurrency section, for lanes that journal in their own repo without a stub
```

The `.* lane` suffix matters — a bare `^## ` also matches other sections and over-counts. Before concluding a lane is absent, check whether it journaled in its own repo. Name the lanes you counted.

Write the index line per the conventions: one line per day covering **every** lane that ran, including lanes that journaled elsewhere. If a load-bearing decision was made today and your project keeps decision records, confirm it is recorded and referenced from the journal. Commit + push (in the index file's own repo, under the cross-lane rule if it is not this repo).

### 6. Deploy — production reflects what merged
- **`deploy_cmd` unset → print "deploy: not configured, skipped" and move on. No deploy, no ssh, no remote command of any kind.**
- `deploy_cmd` set: merging to the default branch does not deploy by itself, and whichever session merged a prod-affecting change should have deployed it at the time — so this step is a *backstop* that should usually find nothing. It still runs. Judge "prod-affecting" against **the whole day's merges, not just your lane's**, using `prod_paths` (journal and memory changes never count):
  - `deploy_gap_cmd` set: run it; a deploy is due only if it lists prod-affecting commits.
  - `deploy_gap_cmd` unset: ask the operator whether anything prod-affecting merged today. Do not guess, and do not trust a dry-run flag to report staleness unless you know it really fetches.
  - If a deploy is due: **state what will be deployed and ask the operator to confirm**, then run `deploy_cmd` exactly as configured, confirm whatever success signal your deploy emits, and name the merge that left the gap.

### 7. Resume prompt
Emit **one** short resume prompt for the whole day, per the shared convention, not one per lane:
- Top 1-3 priorities across all lanes (mirror each lane's `### Next`, deduplicated and ranked)
- The ticket ids and decision records in play
- The memory files and branches needed to resume context
- Any open PRs per repo and their state

Keep it tight — a paste-ready block, not a recap of the day. Tomorrow it is pasted first, then `/standup` runs.

### Part B self-check — do not report EOD-complete until all pass
- [ ] Every lane accounted for: arc or stub present in the journal, **or** confirmed in that lane's own repo journal (name them either way)
- [ ] `<journal_dir>/D.md` exists and holds every lane that had finished when you closed, committed + pushed
- [ ] **Exactly one** index line for `D`, covering every lane — or `journal_index` unset
- [ ] Deploy: `deploy_cmd` unset and said so — or gap checked, operator confirmed, prod-affecting merges from **any** lane deployed, or N/A (nothing prod-affecting merged)
- [ ] One resume prompt emitted
