---
name: atlas
description: A one-screen read of what to do next on the project you are in — where you are in the pipeline, what is in flight, one ranked table of what to pick up (marked startable, yours, or blocked, sized, and grouped by what can run in parallel), and the shape of everything after that. Use when the user asks what is next, what to pick up, where things stand, or invokes /atlas.
---

`/atlas` answers one question — **what do I do next here** — in one screen, and stops.

It is a **read**. It never creates, edits, labels, closes or comments on anything, and it never starts the work it names. The whole output is a briefing the user acts on, or doesn't.

## What it must not become

The failure mode is a status dump: every open ticket, every label, a paragraph each. That is the thing the tracker already does, and worse — the reason to run this is that the tracker's list does *not* say which item is startable, which is blocked, which is the user's own to do, and which few matter. **Anything that does not help choose the next action is cut.** The caps below are caps, not targets; four good rows beat eight padded.

## 1. Learn how this repo works

Never assume the stack. Read, in this order, whatever exists:

- **`CLAUDE.md`** — the workflow section names the pipeline, the branch rule, and what a ticket must do before it counts as done (a deploy, a build, a device check). That last one is a *gate*, and it belongs in the briefing wherever it applies.
- **`docs/agents/issue-tracker.md`** (or equivalent) — which tracker, and the exact commands for it. Where it exists it is authoritative over anything guessed below.
- **`README.md`** — the pipeline stages, where CLAUDE.md is thin.

If there is no tracker at all — no `gh`, no configured issue source — say so in one line, fall back to `TODO`/plan files or the user's own words, and skip the tables rather than inventing them.

## 2. Take the snapshot

Two cheap reads. Prefer them over many small ones.

**The tracker.** On GitHub, one call carries numbers, titles, labels, bodies and dependency edges together:

```sh
gh api "repos/{owner}/{repo}/issues?state=open&per_page=100" --paginate \
  --jq '.[] | select(.pull_request == null) | {n: .number, t: .title, labels: [.labels[].name], blocked: .issue_dependencies_summary.blocked_by, blocking: .issue_dependencies_summary.total_blocking, body: .body[0:600]}'
```

`select(.pull_request == null)` is load-bearing — the REST issues list returns pull requests too, and an unfiltered count is wrong every time there is an open PR. Where `issue_dependencies_summary` is absent or all zeroes the repo isn't using native edges: fall back to `Blocked by: #n` / `Depends on #n` / `Parent epic: #n` lines in the bodies, which is why the body is fetched.

`blocked_by` is a **count**, not a list. A row marked blocked has to name *which* ticket blocks it, so where the count is non-zero and the body doesn't say, resolve the edge before writing the row:

```sh
gh api "repos/{owner}/{repo}/issues/<n>/dependencies/blocked_by" --jq '.[] | select(.state=="open") | .number'
```

**The worktree.** Current branch, commits ahead of the default branch, dirty files, and open PRs with their review state:

```sh
git status --short --branch
git log --oneline <default>..HEAD
gh pr list --state open --json number,title,isDraft,headRefName,statusCheckRollup
```

## 3. Rank

**Cut only one thing: epics and specs.** An issue that other open issues name as their parent (`Parent epic: #n`, `Part of #n`), or that carries a spec's shape — user stories, a solution section, sub-issues — is a container, not a task. It has no blockers and will otherwise sit at the top of the table forever. Epics belong in the second table.

Everything else stays in one list and gets a **state**, because "blocked" and "only a person can do this" are answers to *what do I do next*, not reasons to hide a ticket:

| State | Marker | Means |
|---|---|---|
| Startable | `>` | An agent can begin it now. |
| Yours | `@` | Only a person can do it — `ready-for-human` or the repo's equivalent. A device check, a decision, a playtest. |
| Blocked | `x #n` | Has an open blocker, named. |

**Order the whole list by:**

1. **State.** Every `>` first, then every `@`, then any `x` worth showing. A blocked row earns its place only when it is one ticket away and that blocker is already in the table — otherwise it belongs in the second table.
2. **What it unblocks.** A ticket that frees three others outranks one that frees none. `total_blocking` says this directly.
3. **What decays.** Fallout from work already merged — review findings, follow-ups, a warning that has gone stale — gets cheaper to fix the sooner it's done and more expensive the further the code moves. These are small and they age badly; they belong high.
4. **Where the epic wanted it.** Inside one epic, the authored sequence is a real signal — its own tickets were written in an order.
5. **Cost.** A tie breaks toward the cheaper one.

A `@` row often gates agent work that no dependency edge records — a decision three tickets are waiting on, a device check nothing can proceed past. Say so in its reason when it's true; that is the whole reason it shares a table with the rest.

**Then cut the list into parallel groups.** A group is a set of rows safe to run as simultaneous branches; groups are numbered in the order to take them, and a row's group never puts it above a row it ranks below. Two tickets share a group only when all of these hold:

- **Disjoint files.** Read the bodies for what each names. Two tickets editing the same test file, the same settings asset, or the same central config are consecutive groups, not one — however unrelated their titles.
- **Neither needs the other's answer.** A dependency edge is the obvious case; the common case is unrecorded — a ticket that tunes numbers another ticket is about to rewrite belongs a group later, with the reason in its row.
- **A shared exclusive resource doesn't split them, but it gets named.** One device, one Editor, one owner to be grilled: tickets contending for something single may share a group, and the contention goes in one line under the table — parallel branches, serialized gates.

The order must be **auditable**: every row says why it sits where it does, and a row whose reason is only "it's open" doesn't belong at all.

## 4. The briefing

Exactly this, in this order, and nothing else.

**Pipeline** — the chain from the repo's own docs, with the stage the work is *actually in* marked:

```
grill-with-docs -> to-spec -> to-tickets -> [ implement ] -> land-pr
```

**Mark a stage only when something is in flight.** The brackets say *work is underway here*, not *here is where you would start if you started*. A clean tree on the default branch with no open PR is not a stage — nothing has begun — so print the chain bare, with no brackets anywhere in it, and let the table below say what to pick up:

```
grill-with-docs -> to-spec -> to-tickets -> implement -> land-pr
```

Read the stage off the worktree, not off the tickets: an open PR means `land-pr`, whatever state its checks are in — a red review is work inside `land-pr`, not a stage before it; commits ahead with no PR, or a dirty tree, still means `implement`; no tickets against an existing spec means `to-tickets`; no spec means `to-spec`. Tickets sitting ready are not work in flight — they are the queue, and a queue marks nothing. **A review is not a stage.** Where a repo's own docs list one — `/code-review`, an approval step — it belongs to the stage around it: at the keyboard it is the tail of `implement`, and on an open PR it is part of `land-pr`. Leave it out of the chain. Use the repo's real stage names otherwise — do not import the ones above into a repo that doesn't use them.

**In flight** — one line. Branch, commits ahead, dirty files, open PRs and whether their checks are green. Name the discrepancy where there is one, because that is the whole value of the line: a branch named for a ticket with nothing on it, a PR sitting red, work uncommitted on the default branch. When the tree is clean, on the default branch, with no PRs, say so in four words and move on.

**Next** — one table, at most **eight** rows, of which at most five are `>`, at most three are `@`, and at most two are `x`:

| | Grp | # | Ticket | Sz | Why it's there | Unblocks |
|---|---|---|--------|----|----------------|----------|
| `>` | 1 | 11 | Stale startup warning | S | already written in the worktree, unlanded | — |
| `>` | 2 | 12 | Cull box measures the back rank | S | same view file #11 is in, so it follows | — |
| `@` | — | 13 | Three-numeral tier reads on device | S | only the phone can answer it | — |
| `x #14` | — | 15 | Three kinds of the Cursed | L | next in the epic once the telegraph lands | #16 |

- **First column** — the state marker from step 3, and nothing else. Blocked rows carry their blocker's number so the table reads as a sequence, not a status.
- `Grp` — the parallel group from step 3. Equal numbers are safe as simultaneous branches; a higher number waits for the lower ones it conflicts with. `@` and `x` rows take `—` — a person's work and a blocked ticket don't contend for branches.
- `#` — the number, so it can be pasted into `/implement`.
- `Ticket` — a short phrase, not the full title. The number is the identifier; this column is for recognition.
- `Sz` — effort, one of `S` / `M` / `L` / `?`, read off the ticket's own acceptance criteria rather than its title. **S** — one file or one dial, plus its test. **M** — a few files and new tests, no new decision to make. **L** — a new type or subsystem, guards and docs to update with it. **?** — no letter is honest: the ticket is too thin to size, a decision has to be made before code can start, or it touches everything. A repo gate the ticket triggers (a device deploy, a build) moves it up one size, because the gate is part of finishing it — so a gated `L` is a `?`. `L` is the largest size there is; `?` is where everything past it lands, and it reads as *look at this one before picking it up*, never as a guess dressed as a letter.
- `Why it's there` — the ranking reason from step 3, plus any gate the repo imposes (a device deploy, a build, a human check). One clause, no full stop.
- `Unblocks` — the tickets this frees, or `—`.

Rows read top to bottom as the order to do them in; equal `Grp` numbers mean the order between them doesn't matter and they can run at once. One legend line under the table, or none if every marker in it is self-evident: `> startable · @ yours · x blocked by`.

**Beyond that** — a second table, at most **five** rows, grouping everything left over. Orientation, not a queue: a reader should finish it knowing the shape of the next month.

| Cluster | What it is | Count |
|---------|------------|-------|
| #10 decision layer | the Run's rebuild into two power axes | 6, two startable |
| Scene-tuning guards | holes the #9 fixes left open | 3, all S |

- `Cluster` — an epic, a module, or a theme. A ticket number only when one carries a whole cluster.
- `What it is` — half a line.
- `Count` — how many tickets, and a word on how much of it is reachable or what it sizes to. Never a time estimate.

## Style

Terse. No preamble, no restating the question, no offer to start the work — the user asks for that in their next message. No status emoji and no severity badges: the marker column is the only status vocabulary, it is ASCII, and it stays that way. Numbers where they are known, and nothing invented to fill a column: an unsized ticket gets `?`, not a letter chosen to look decisive.

If something is genuinely ambiguous — two tickets tied, an epic whose order isn't stated — pick one, put the reason in the row, and move on. A briefing that hedges every row is a briefing that decided nothing.
