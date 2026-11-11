# Projecting the review into a CI brief

A CI runner gets a fresh checkout and no skills, so a reviewer running there cannot read `SKILL.md`. It reads a **brief committed to the repo** — and that file, not this skill, is what a CI review means in that project. Write it as a self-contained copy: the verdict, the burden of proof, the exclusion list, the kill mandate, the ratchet and the report shape all restated in the brief itself.

## What the project layer adds

Only what a reviewer arriving cold could not know:

- **Doc priority order** — which files are standards, and which wins when two disagree.
- **How to resolve the spec** — where issue references live (PR body, title, branch, commit messages) and how to fetch one. No spec resolves, the Spec lens is skipped rather than guessed at.
- **The traps** — the handful of failures invisible to a generic reviewer and expensive in this project: the assembly boundary that a compile error catches too late, the asset reference whose absence renders magenta only on device, the rule about where a feature is allowed to land. Each with the document that states it.

Everything else in the brief is this skill.

## Keep the feedback loop symmetric

A brief that only ever grows drifts toward volume. Edit it in both directions:

- the review missed something → add a trap, with the document that states it;
- the review **invented** something → add a line to *Never flag*, naming the invention.

The second edit is the one that holds precision, and it is the one a brief usually lacks. Prefer it to adding rules: adding rules is measured to make reviewers noisier, not sharper, and a long brief dilutes the rules that matter most. When a blocking finding is declined by the author, that is a burden-of-proof failure and it belongs in *Never flag*.

## Wiring a push-triggered review

- Review **every push to a non-draft PR**, so a fix is reviewed by pushing it. Drafts wait for ready-for-review, which is what makes marking a PR draft the way to push without a review.
- Keep the manual re-run trigger for when nothing was pushed, and **push instead of commenting**. A comment event and a push event usually sit in different concurrency groups, so they do not cancel each other and two reviewers write the same sticky comment at once.
- A second push cancels the first push's review. Confirm that, because two reviews racing one comment is invisible until the report reads as two halves.
- Post the review as a comment and leave the check neutral. The verdict is what the loop reads; the bot does not hold the merge button.
- A run that dies on its turn budget leaves an unticked checklist and no report, so a green check is not proof a review happened — read the comment body. The ratchet is what keeps the budget flat: a reviewer that re-reads the whole conversation costs more every round, and raising the ceiling does not bend that curve.
- Where the action refuses to run when the workflow file differs from the default branch, the PR that edits the workflow gets no review at all. Say so in the workflow's own header.
- Where effort is a dial, run an every-push review at the low end — it reports only what it is most confident in — and escalate on demand.
