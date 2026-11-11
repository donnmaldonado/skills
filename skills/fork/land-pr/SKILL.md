---
name: land-pr
description: Drive a pull request from CI review to merge — adjudicate the review, fix what blocks, wait for the next one, land it or stand down.
disable-model-invocation: true
---

A PR **lands** or it **stands down**. This skill is the author in the review round-trip: it waits for the review of the commit actually on the branch, adjudicates every finding against a written bar, fixes what blocks, pushes, and waits again — closing the loop when nothing blocking survives, or handing back when the loop stops converging.

The argument is a PR number — or the issue number the PR will carry, or nothing at all, in which case it is the PR for the current branch. **A PR that does not exist yet is not a stop.** This skill is invoked beside the work about as often as after it, and a number that names an open issue names the PR that is coming: so it waits for that PR to open and starts the moment it does. The same is true one level down — a head with no finished review is not a missing review, it is a check that has not landed yet, and step 2 listens for it rather than declaring it absent.

## Process

### 0. Resolve the argument, or wait for the PR to open

```sh
gh pr view <n> --json number,state          # <n> is already a PR
gh pr view --json number                    # or the branch has one
```

Either resolves → step 1 with that number. Neither → the PR has not been opened yet, and there is one question worth asking before waiting for it:

```sh
gh issue view <n> --json number,title,state
```

- **It is an issue** — then it names the PR that is coming, and waiting is the right move.
- **It is nothing** — the number names neither a PR nor an issue in this repo. Say so and stop. That is the only argument which ends the run before it starts.

**Wait by listening, not by polling in the foreground.** The gap between invocation and the PR opening is minutes to hours — an implementing session finishing its work — and the wait owes you exactly one notification: the one that ends it. So arm it detached (Bash, `run_in_background: true`) and let it wake you:

```sh
branch=$(git rev-parse --abbrev-ref HEAD)
for _ in $(seq 240); do                                    # 30s apart, ~2 hours
  pr=$(gh pr list --state open --head "$branch" --json number --jq '.[0].number // empty')
  [ -z "$pr" ] && pr=$(gh pr list --state open --search "<n> in:title,body" \
                        --json number --jq '.[0].number // empty')
  [ -n "$pr" ] && { echo "PR #$pr is open"; exit 0; }
  sleep 30
done
echo "no PR for <n> after two hours"; exit 1
```

Two ways of finding it, because the branch may be cut before the wait is armed or after it. `--head` is the exact one and is preferred; the search is what catches a PR opened from a branch that did not exist when the wait started, and being a text match it is a *candidate* rather than an answer — `gh pr view` it and confirm the title or body actually names the issue before adopting it. Exit 0 hands you the number on stdout; exit 1 means the wait ran out, which is a report and a stop, not a retry.

Resume at step 1 when it fires, and run step 1's `gh pr view` again rather than trusting the line the wait printed: a PR opens as a draft about as often as not, and a draft is step 1's business.

**Do not fill the wait with work.** The branch belongs to whoever is implementing on it. A skill that tidies a file, turns a dial or runs a build while waiting is a second writer on one worktree, and the PR it eventually adjudicates is partly its own — which is the one thing an author in a review loop must never be. Wait, and nothing else.

### 1. Pin the PR

```sh
gh pr view <n> --json number,url,title,isDraft,state,headRefName,headRefOid,baseRefName,mergeable,mergeStateStatus
```

Three states end the run here, reported and untouched:

- **Closed or merged** — there is nothing to land.
- **`mergeStateStatus: DIRTY`** — conflicts with the base. Rebasing is the author's call, not this skill's. `UNKNOWN` is not a state, it is GitHub not having computed one yet: re-query until it resolves.
- **The worktree is elsewhere** — you are not on the PR's head branch at its head SHA and cannot move there cleanly (dirty tree, diverged local branch). Say which, and stop.

A **draft** is not a stop. Drafts are how a branch takes a push without a review, so the reviewer never fires on one: `gh pr ready <n>`, which is itself the trigger, and say so in the final report.

Record the head SHA. Every step below is about *that* commit.

### 2. Get the review that describes this head

The reviewer posts a **sticky comment** — one comment, updated in place. Its presence proves nothing about which commit it read, so a comment is never the thing you wait on. Correlate through the workflow run:

```sh
grep -rl 'anthropics/claude-code-action' .github/workflows/    # the review workflow
gh run list --workflow=<file> --json databaseId,headSha,status,conclusion,event --limit 20
```

Take the run whose `headSha` equals the head from step 1, **ignoring any whose conclusion is `skipped`** — those never entered the job at all (a comment that carried no trigger phrase), and one sitting at the right SHA will otherwise shadow the run you want. Then:

- **Terminal already** → read the sticky comment: `gh pr view <n> --json comments`, latest comment authored by the reviewing app.
- **Anything else** → **listen for it, the same way step 0 listened for the PR.** A run that has not been created yet and a run still thinking are the same thing from here: the check is not done, and it finishes on its own schedule rather than yours. One detached wait covers both, so a push is followed by a single listen instead of a poll you sit inside:

```sh
sha=<head SHA from step 1>; wf=<the review workflow file>
for _ in $(seq 50); do                                     # 30s apart, ~25 minutes
  run=$(gh run list --workflow="$wf" --json databaseId,headSha,status,conclusion \
          --jq "[.[] | select(.headSha == \"$sha\" and .conclusion != \"skipped\")][0]")
  [ -n "$run" ] && [ "$(echo "$run" | jq -r .status)" = "completed" ] \
    && { echo "review run terminal: $run"; exit 0; }
  sleep 30
done
echo "no terminal review run for $sha"; exit 1
```

Exit 0 hands you the run and you read its sticky comment. Exit 1 means nothing reached a conclusion for this SHA inside the window — no review is coming, so go to 2b. **The wait ends on *terminal*, never on *green*:** a red conclusion is not an absent review, and separating those two is the whole of what follows.

**Read the comment, not the colour.** Green, red and skipped all mean the same thing here — go look. Three states to tell apart, and only the comment's *content* separates them:

- **Green with no review.** The action refuses to run when the workflow file differs from the default branch, so a PR that edits the reviewer's own workflow is skipped in silence. The comment holds no report.
- **Red with the review posted and correct.** A run that finished and then turned out to have gone one turn over budget. The report is there. Use it.
- **A checklist and no report.** The run died mid-thought on its turn budget or its clock. Either colour. No usable review.

A comment counts as a review only when it carries findings. Otherwise, 2b.

### 2b. Review it locally instead

No reviewer wired into the repo, or a run that produced no report: invoke `/adversarial-review` on `origin/<base>...HEAD` and adjudicate that. The loop runs unchanged and may still land the PR — the final report says the round was reviewed locally, and why CI did not supply one.

### 3. Adjudicate

Load `/adversarial-review` and apply its bar. It already defines blocking narrowly, sets the burden of proof each finding must carry, and holds the never-flag list. Point at it and add nothing: one definition of *blocking*, in one place.

Sort every finding into three buckets:

- **Fix** — blocking under that bar.
- **Answer** — real but not blocking, or failing the burden of proof. It gets a reasoned dismissal on the PR thread, in one sentence, saying why it does not block.
- **Settled** — you already answered it in an earlier round.

**Findings that survive their own dismissal are the norm, not a bug.** A CI reviewer with no memory of its last report re-reviews the whole diff every pass, so anything you declined comes back on the next push. Keep a ledger across rounds — finding, bucket, one line of reasoning — and treat a recurrence of something already answered as settled. Nothing in the settled bucket is re-argued, re-posted, or counted against convergence.

Completion criterion: every finding in the report sits in exactly one bucket, with its reason recorded.

### 4. Fix, gate, push

Fix the **Fix** bucket and nothing else. A round that also tidies something is a round whose next review has new surface to find.

Then run the repo's own gate before pushing — a red round is a wasted round. Read it out of `CLAUDE.md` and `README.md` rather than assuming a stack, and honour what those documents say counts as a pass: a suite that reports zero tests, or a results file that never appeared, is a run that did not happen, not a green one. Where the repo defines a step beyond the tests — a device deploy, a build — the same documents say when it applies, and it applies.

Commit in the repo's own style, read off `git log`. Then push. **The push is the trigger** — it fires the review directly, so never also comment the reviewer's trigger phrase. The two arrive as different events, do not share a concurrency group, and would put two reviewers on one sticky comment.

Back to step 2 with the new head SHA.

### 5. Land, or stand down

**Land** when adjudication leaves the Fix bucket empty: post the dismissals for everything in Answer, then merge the way this repo merges — read `git log` for squash-versus-merge rather than assuming. Report the merge commit.

**Stand down** — post the state of play on the PR thread, hand the remaining findings back, and merge nothing — on any of:

- **The cap.** Three fix rounds spent.
- **Degradation.** A round's fix draws a blocking finding the round before it did not have, or turns the gate red for a reason the fix does not obviously explain.
- **Diminishing returns.** The same blocking finding survives two consecutive attempts, or the blocking count fails to fall from one round to the next.

Standing down is an outcome with a report, not a failure: name the findings still open, what the last round tried, and which of the three above ended it.

## Why the ledger

Everything else here is mechanism; the ledger is the reason the loop terminates. A reviewer that reads the diff afresh each pass has no way to know a finding was raised, weighed and declined an hour ago — so it raises it again, and a loop that re-adjudicates it burns a round on work it already did and reaches the cap with nothing gained. Carrying the settled bucket forward is what converts "the reviewer keeps saying this" from a signal into a fact you have already priced.
