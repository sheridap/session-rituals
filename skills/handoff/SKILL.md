---
name: handoff
description: End-of-session handoff protocol — Part A per lane (commits ∧ memories ∧ tickets ∧ journal arc) in the repos that lane touched; Part B once for the day (journal index line, deploy if configured, resume prompt) by a single closer. Invoke when the operator says "EOD", "end of day", "run handoff", "wrap up the session", or signals session-close. Detect from the lock table (if configured) whether other lanes are live and run the right parts — the operator should not have to know.
---

# /handoff — End-of-session handoff

Closes a working session to a durable state, and — once per day — closes the day.

**Two completion bars, don't conflate them.** *Your lane* is complete when Part A's artifacts are committed and pushed, your arc included. *The day* is complete when one session has additionally run Part B. A lane that finishes Part A and pushes is done and may stop, whether or not the day has closed.

The journal is the artifact most often skipped — it has no natural surfacing the way `git status`, the memory index, and ticket queries do. Do not claim your lane complete until your arc is written, committed, and pushed — and never claim *the day* complete unless you actually ran Part B.

> **Operator-protocol note:** this skill runs *only* when the operator explicitly signals session-close. It closes the loop they asked to close — it never suggests closing one.

## Parameters

Override per repo in that repo's `CLAUDE.md` under a `## Session rituals` heading, one `- key: value` line each. Absent keys take the default.

| key | default | what it changes |
|---|---|---|
| `journal_dir` | `journal/` relative to the repo root | Where your arc is appended, as `YYYY-MM-DD.md`. May be an absolute path to a **shared journal** used by several repos. |
| `journal_index` | none | A file holding a one-line-per-day index (e.g. `journal/README.md` under a `### Recent entries` heading). When unset, Part B step 5 is skipped. |
| `journal_repos` | none | Comma-separated paths of other local repos that keep their own `journal/`. Used to route your arc and to census lanes in Part B. |
| `ticket_system` | `none` | `linear`, `github`, or `none`. Controls step 3. |
| `ticket_prefix` | none | Issue-id prefix used to recognise ticket references. |
| `lock_status_cmd` | none | Prints every live session holding a repo lock, with claim time. When unset, **this session is assumed to be the only one** and runs Part A then Part B. |
| `lock_check_cmd` | none | `<cmd> <repo-path>` — exits non-zero if another live session holds that repo. |
| `lock_claim_cmd` | none | `<cmd> <repo-path>` — claims a repo lock before an authorised cross-lane append. |
| `deploy_cmd` | none | The command that deploys the production surface. **When unset, Part B step 6 performs no deploy and prints one line saying so.** The skill never composes an ssh command; if your deploy runs remotely, put the whole invocation in this value. |
| `deploy_gap_cmd` | none | A read-only command that prints the commits merged but not yet deployed (fetch + diff on the production checkout). When unset and `deploy_cmd` is set, the skill asks the operator whether anything prod-affecting merged today instead of guessing. |
| `prod_paths` | none | Comma-separated path globs that count as prod-affecting (e.g. `src/**,deploy/**,requirements*.txt`). Docs/journal-only days skip the deploy. |
| `memory_dir` | `default` | `default` = Claude Code's auto-memory for the current project. |

**Never assumed:** no ssh, no deploy, no write to any external system beyond the ticket system named above. Every mutation this skill makes is a git commit, a memory file, or a ticket update.

Use the wall-clock date injected at the top of the session for `YYYY-MM-DD` if your harness injects one — do not guess; the model's date-awareness is frozen at session-start. Otherwise run `date +%F` once and reuse it.

## Concurrency — decide this first

Operators often run several lanes at once, one Claude Code session per repo. The steps below are not uniform under that: some are per-session, and three are **once-per-day singletons** no matter how many lanes ran.

| | steps | why |
|---|---|---|
| **Part A — lane-local** | commits · memories · tickets · **your journal arc, appended yourself** | per-repo, per-fact, per-ticket, per-append — safe in parallel |
| **Part B — the closer** | journal index line · deploy · resume prompt | *one* line, *one* prod surface, *one* prompt the operator reads |

**Every session runs Part A. Exactly one session additionally runs Part B.**

**Decide this yourself — do not ask the operator which parts to run.** They say "EOD"; you work out the rest.

- **`lock_status_cmd` unset** → you are the only session. **A then B.**
- **`lock_status_cmd` set** → run it and read the table:

| what it shows | run |
|---|---|
| **no other live session** | **A then B.** A solo day needs no ceremony — close it and report the day done. |
| **other lanes live, and one of them claimed earlier than you** | **A only.** Report which lanes are still live and that a closer is still owed. |
| **other lanes live, but you claimed earliest** | **A**, then say you are the fallback closer and ask whether to close now or wait — closing early means deploying before their merges land. |

The operator can always override — "close the day", "run Part B", "you're the closer" — and an explicit instruction beats the table. But the default is that `/handoff` behaves the same from the operator's side whatever the topology; the concurrency handling is the skill's problem, not theirs.

**Write your own arc — do not hand it to anyone.** An **append** is safe where a whole-file write is not: two appends either coexist or conflict *loudly*, and neither silently loses text. A whole-file write silently wins. So the journal *file* is not a single-writer resource and never needed a closer; the genuine singletons are the **index line**, the **deploy**, and the **resume prompt**. An arc that lives only in a hand-off message exists in two transcripts and nowhere durable.

**Where your arc goes depends on whether your repo keeps its own journal.** Derive it, don't memorise it:

```bash
for j in <journal_dir> <each journal_repos entry>/journal; do
  [ -d "$j" ] && printf '%-30s last entry %s\n' "$j" \
    "$(ls "$j"/2*.md 2>/dev/null | sort | tail -1 | xargs -I{} basename {} .md)"
done
```

- **Your repo keeps a *live* journal** — a `journal/` tree with recent entries → write it there. No contention at all.
- **It does not, or its tree is months stale** → your arc goes in the **shared journal** (`journal_dir` when it is an absolute path). Append it yourself (step 4). A dormant `journal/` is a trap: writing into a tree nobody reads makes your arc invisible. If the last entry is not recent, treat the repo as having no journal and say so.

**Who closes Part B:** the operator names the closer at EOD. Absent a call: **the earliest claim time among all live locks closes** — chosen for determinism, not seniority. **If you are that lane, you are the closer. Do not wait for anyone.** A lane finishing after you appends its own arc, amends the index line, and deploys its own work (step 4's late-arrival clause). Waiting is what produces a day nobody closes.

**Lock age is not staleness.** `/lane-reset` resets the transcript, not the session, so a lane can stay claimed for days while working normally. You cannot determine from the lock table whether a lane is still working; if you need to know, ask the operator. Never reap a lock by hand on an age heuristic.

**Cross-lane writes.** This skill authorises **two** narrow writes into a repo another lane owns, and nothing else: (1) your arc, appended to the shared journal; (2) the index line, if you close while anchored elsewhere. Both follow the same rule: **claim that repo's lock first (`lock_claim_cmd`), stage only that one path, and ignore every other dirty path in that tree — it belongs to the lane that owns the repo.**

The three failures this structure prevents, each seen in practice: two lanes overwriting the same journal file (one arc lost); two index entries for one day; and a deploy nobody ran because a global step sat inside a per-lane checklist — everyone's job, therefore nobody's.

---

# Part A — lane-local (every session runs this)

### 0. Scope — the repos *this lane* touched
List the repos this session actually worked in. Check them, and **only** them:

```bash
for d in <this lane's repos, not a glob>; do
  s=$(git -C "$d" status --short)
  a=$(git -C "$d" log --oneline @{u}.. 2>/dev/null)
  [ -n "$s$a" ] && echo "=== $d ===" && [ -n "$s" ] && echo "$s"; [ -n "$a" ] && echo "ahead: $a"
done
```

**Do not sweep every repo on the machine.** A concurrent lane's dirty tree is not yours to report or clean, and seeing it invites action across a seam you do not own. If you believe another repo needs attention, say so to the operator — do not touch it.

### 1. Commits — every change committed + pushed
For each dirty/ahead repo:
- Review the diff before committing — never commit unreviewed. If your project runs an adversarial pre-commit review on code diffs, it applies here.
- **Stage explicit paths only — never `git add -A`/`.` or `git commit -a`.** The working tree may hold a concurrent session's uncommitted edits; blanket staging sweeps them into your commit under the wrong message. Confirm every staged path is something *this* session changed.
- One commit, one repo — never mix repos in a single commit or branch.
- Commit on a branch if on the default branch and the work warrants it; otherwise commit + push.
- End state: `git status --short` shows only runtime artifacts. Nothing else uncommitted.

### 2. Memories — capture learnings
- Write new `feedback` / `reference` / `project` memories for anything non-obvious learned this session (corrections, gotchas, reference procedures). One fact per file, with frontmatter.
- Add a one-line pointer to the memory index.
- Update or delete any memory this session proved stale. Don't duplicate — check for an existing file first.
- Skip anything the repo already records (code structure, git history, `CLAUDE.md`).

### 3. Tickets — reconcile state
- `none`: skip; the commits and the journal arc carry status.
- `linear` / `github`: update status on every ticket touched this session. File follow-up tickets for deferred items, so nothing lives only in the transcript. Reconcile today's commits against open tickets — close what shipped, comment progress on what advanced. Keep bodies plain; some ticket APIs reject long or richly formatted content.

### 4. Your journal arc — write it yourself, now

**Append** it (never a whole-file write) to whichever journal the concurrency section routed you to. Set the date once, from the injected wall-clock — a lane wrapping after midnight is still writing *yesterday's* day:

```bash
D=YYYY-MM-DD                      # ← the injected wall-clock date
J=<journal_dir>/$D.md
( set -o noclobber; printf '# %s\n\n> One section per lane.\n\n' "$D" > "$J" ) 2>/dev/null || true
cat >> "$J" <<'ARC'
## <lane> lane (HH:MM–HH:MM)
…
ARC
git add "$J" && git commit -m "docs(journal): <lane> arc for $D" && git push
```

`noclobber` is load-bearing, not style. A plain `[ -f "$J" ] || printf … > "$J"` is a **truncating** write, and two lanes that both find the file absent will both run it — the second wiping the first's section. Under `noclobber` exactly one lane creates the file and the losers no-op.

Lanes sharing one checkout serialise on commits; what you may actually hit:
- **`fatal: Unable to create '.git/index.lock': File exists`** — another lane is mid-commit. Wait a moment and retry; it is contention, not corruption.
- **Your commit carrying another lane's section.** Fine — both belong on the default branch; say so in the message.
- **`git status` clean and your text already committed** — the other lane's `git add` swept it in. Confirm with `git log -p -1 -- "$J"` before concluding anything was lost, then just push.

*(Only if two checkouts are in play — a worktree, another machine — can a push be rejected. Then `git pull --rebase`, resolve by **keeping both sections**, `git add "$J"`, `GIT_EDITOR=true git rebase --continue`, push. `GIT_EDITOR=true` matters: a bare `rebase --continue` opens an editor and hangs a session with no tty. **Never `--ours`, `--theirs`, or `push --force`** — each resolves the conflict by deleting a lane's record.)*

Once pushed, your arc is durable and **your lane is done** — you do not wait for the closer.

**If the day is already closed** (Part B has run), you inherit Part B for your own work: append as above, amend the single index line — amend, never add a second — and run step 6 yourself if anything you landed is prod-affecting and `deploy_cmd` is set.

Structure — h2 for the lane, h3 for its sections, **always**, even on a one-lane day, so a day that gains a second lane later stays well-formed:

```markdown
## <lane> lane (HH:MM–HH:MM)

### Shipped        — tickets closed, decisions recorded, commits landed (link ticket ids, commits)
### Decisions      — strategic calls; if load-bearing, reference the decision record
### Learnings / SOPs
### Deferred       — what got punted + where it's tracked
### Next           — top 1-3 priorities for this lane
```

The `lane` suffix in the heading is required — Part B's census greps for it.

If your record lives in your own repo's journal and a shared journal exists, leave a two-line pointer stub in the shared journal naming the repo and the commit, so the day reads complete and `/standup` can follow it.

### Part A self-check
- [ ] This lane's repos: working tree clean (only runtime artifacts). If you appended to a shared journal from another lane, that tree's *other* dirty paths are not yours — ignore them.
- [ ] This lane's repos: pushed, up-to-date with the default branch on origin
- [ ] New memories written + index updated
- [ ] Tickets updated + follow-ups filed for deferred work (or `ticket_system: none`)
- [ ] Arc **appended, committed and pushed** — in your own repo's live journal, or the shared journal; pointer stub left if the former

---

# Part B — closing the day (exactly one session)

Run this if you are the closer. **Do not wait for other lanes.** Count the lanes you can see, name them, and close.

### 5. Index the day — one line
Skip if `journal_index` is unset. Otherwise, account for every lane before writing it:

```bash
grep -n '^## .* lane' <journal_dir>/$D.md          # lanes that recorded here
# plus the last-entry loop from the concurrency section, for lanes that journal in their own repo
```

The `.* lane` suffix matters — a bare `^## ` also matches other sections and over-counts. Before concluding a lane is absent, check whether it journaled in its own repo. Name the lanes you counted.

The line is a bold one-line summary spanning **every** lane that ran, leading with the day's shape and naming each lane's headline — including lanes that journaled elsewhere. One line per day, never one per lane. If a load-bearing decision was made today, confirm it's captured in your decision records and referenced from the journal. Commit + push.

### 6. Deploy — production reflects what merged
- **`deploy_cmd` unset → print "deploy: not configured, skipped" and move on. No deploy, no ssh, no remote command of any kind.**
- `deploy_cmd` set: merging to the default branch does not deploy by itself, and the session that merged a prod-affecting change should have deployed it in the same unit of work — so this step is a *backstop* that should find nothing to do. It still runs. Judge "prod-affecting" against **the whole day's merges, not just your lane's**, using `prod_paths` (docs/journal-only changes never count):
  - `deploy_gap_cmd` set: run it; deploy only if it lists prod-affecting commits.
  - `deploy_gap_cmd` unset: ask the operator whether anything prod-affecting merged today. Do not guess, and do not trust a dry-run flag to report staleness unless you know it really fetches.
  - Then run `deploy_cmd` exactly as configured, confirm whatever success signal your deploy emits, and name the unit of work that left the gap.

### 7. Resume prompt
Emit **one** short resume prompt for the whole day, not one per lane:
- Top 1-3 priorities across all lanes (mirror each lane's `### Next`, deduplicated and ranked)
- The ticket ids and decision records in play
- The memory docs and branches needed to resume context
- Any open PRs per repo and their state
- **No line prefixes** — a fenced code block or plain paragraphs, never `>` or pipes, so the paste is clean (same rule `/lane-reset` follows).

Keep it tight — a paste-ready block, not a recap of the day.

### Part B self-check — do not report EOD-complete until all pass
- [ ] Every lane accounted for: arc present in the journal, **or** confirmed in that lane's own repo journal (name them either way)
- [ ] `<journal_dir>/<date>.md` exists for today and holds every lane that had finished when you closed, committed + pushed
- [ ] **Exactly one** index line for today, covering every lane — or `journal_index` unset
- [ ] Deploy: `deploy_cmd` unset and said so — or gap checked, prod-affecting merges from **any** lane deployed, or N/A (docs-only day)
- [ ] One resume prompt emitted
