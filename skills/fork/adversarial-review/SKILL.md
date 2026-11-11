---
name: adversarial-review
description: Adversarial review of a diff, a PR, or a branch — every finding carrying a burden of proof, capped, and ending in a verdict rather than a list. Use when the user wants a PR, branch or diff reviewed, wants a review made less noisy or a review loop that will not converge, or is writing a CI review brief; and when another skill needs the rules deciding what a finding must prove before it is reported.
---

A review is an **adjudication**, not a sweep. The reviewer that treats it as a completeness exercise reports everything that could be an issue, and the author pays a round trip for each one — the measured shape of that failure is 79% of comments technically accurate and not actionable, and precision below 10% for automated reviewers in the literature. The reviewer that adjudicates reaches a **verdict** and attaches only what overturns it. Volume is evidence against precision, never for it: the highest-precision automated reviewers post one or two findings per review.

## The verdict

The deliverable is a verdict in one of two states, and it is the first line of the report:

- `Verdict: no blocking findings`
- `Verdict: N blocking findings`

**The default verdict is `no blocking findings`.** A finding earns its place by overturning that default. Reach the verdict, then justify it — never assemble findings and count them at the end.

**Blocking** is one tier and it is narrow: the change would break behaviour, lose or leak data, breach a standard the project has written down, or deliver something the spec did not ask for. Everything else is **non-blocking** and never delays a merge.

Hold to the standard of review: *"there is no such thing as 'perfect' code—there is only better code"*, and approve *"once it is in a state where it definitely improves the overall code health of the system being worked on, even if the CL isn't perfect."* Prefer no finding to a weak finding.

When nothing clears the bar, say so in that shape and stop:

```
Verdict: no blocking findings
Standards 0 · Spec 0 · Adversarial 0
```

Projecting these rules into a committed brief for a CI runner that cannot read skills: [`CI-BRIEF.md`](CI-BRIEF.md).

## The burden of proof

Every finding, in every lens, carries four things:

1. a `file:line` anchor **on a line this diff changed** — or, where the defect is an omission, the exact place the change should have reached and what belongs there;
2. at most one quoted line as evidence;
3. the concrete input, state or sequence that makes it manifest;
4. what it costs when it does.

Missing any one of the four, the candidate is **discarded and leaves no trace** — no ledger of what was checked, no "possible", no "consider", no demotion to a lesser severity. A candidate you cannot prove is not a smaller finding; it is not a finding.

Calibrate on confidence, and post at 80 or above: at 100 the evidence directly confirms the defect; at 80 you traced it and it will be hit in practice; at 50 it is real but rare or trivial; at 25 you could not verify it. Better to miss a theoretical issue than to flood the report — a reviewer that cries wolf gets ignored, which costs more than the finding was worth.

## Never flag

Silence on these is the review's precision, and it is where most of the work is:

- anything a linter, formatter, type checker or compiler catches, and anything the project's CI already enforces;
- a convention no project document states — measure the file before claiming its style, and if the claim rests on your own taste, drop it;
- a finding the author already answered, and anything materially adjacent to one;
- an issue on lines this diff did not touch, and defects that predate the change;
- a rule the code deliberately silences (`eslint-disable`, `# noqa`, `@ts-expect-error`) or that a project document endorses;
- test coverage, documentation and general code quality in the abstract, unless the project asks for them or the spec named them;
- a change in behaviour that reads as intentional and belongs to the work at hand;
- theoretical risk needing unlikely preconditions, defence-in-depth on top of adequate defences, and "consider using library X";
- a limitation the author has already disclosed or tracked elsewhere;
- the review conversation itself — a document that records a round, a tally that counts this diff's own edits;
- a nitpick a senior engineer would not say out loud.

## The kill mandate

Between finding a defect and reporting it, try to kill it. Read the file rather than the hunk, find the callers, look for the comment or doc that explains the design, and construct the failing scenario in full. **The default outcome of this pass is refuted** — roughly four in five candidates die here, and the ones that die are the ones that would have cost a round. Give this pass more context than the pass that found the candidate had: a defect that survives only inside the diff's own field of view is refuted by the file around it.

## The ratchet

Each round is narrower than the one before, so the loop converges instead of grinding.

- **Round 1** — the verdict, the blocking findings, and a capped non-blocking list.
- **Round 2 and after** — read your own last report and the **delta since it**, not the conversation. Blocking findings still unresolved carry forward verbatim; findings the author fixed vanish; findings the author answered, declined or acknowledged are **settled** and stay settled. Raise no new non-blocking findings at all.
- A defect **introduced by a fix you asked for** clears the same burden of proof as anything else, and is otherwise unspoken. A fix that spawns a finding that spawns a fix is the loop failing, not the review working.

The verdict is the stop condition. `no blocking findings` ends the review; the non-blocking list is the author's to take or leave.

## The three lenses

Separate, never merged and never ranked against each other — code can follow every standard and implement the wrong thing.

- **Standards** — what the project has written down, in its own priority order. Carry the design smells (Fowler, *Refactoring* ch. 3) as a baseline underneath, gated: a smell is reportable only when it names the concrete future change it makes expensive, it is never blocking, and a project standard that endorses the shape overrides it.
- **Spec** — against the issue or PRD this change claims, quoting the line: requirements missing or partial, behaviour nobody asked for, requirements implemented wrongly. No spec resolves, no lens; do not guess which issue was meant.
- **Adversarial** — where this change breaks. Edge cases, ordering and lifetime, cost at the scale it will actually run, and logic that is testable but untested.

## The report

Under 400 words, whole report, and it is a list of defects rather than an essay:

```
Verdict: 1 blocking finding
Standards 1 · Spec 0 · Adversarial 0

**BLOCKING** `RunBootstrap.cs:212` — the material is created at runtime, so its shader
ships stripped and every gray-box body draws magenta on device.
`var mat = new Material(Shader.Find("Universal Render Pipeline/Lit"));`
Triggers on any device build; Play mode looks correct.

Non-blocking: Data Clump — Share takes a tally and a number derived from it (Wave.cs:88).
```

- At most **3 blocking** and **5 non-blocking**. Past the cap, a count: `plus 4 similar`. A change with more than three blocking findings has one finding — it needs restructuring or splitting.
- One finding, one bullet, one sentence of claim. The author writes the fix.
- Non-blocking findings collapse onto one trailing line, semicolon-separated.
- No praise, no summary of what the change does, and no superlative — naming a worst issue per lens invents a finding for each empty lens.

## Worked examples

Reported:

- `WaveSchedule.cs:64` — `emptyLaneChance` at 1.0 leaves the composer with no lane to pick, and the `while` retry never terminates: Wave 12 hangs the Run. **Blocking**: traced, reproducible, names the input.
- `Bow.cs:30` — the spec's acceptance criteria name harshening, and no case exercises it. **Blocking**: quotes the criterion, and the gap is the whole requirement.

Discarded:

- "`docs/adr/0002:6` is a 120-column line in a file that wraps at ~98." No document states a wrap width; the number was inferred from neighbouring lines. **Never flag** — an invented convention.
- "The Wave-level case lost the precondition its two siblings assert. Redundant against every production mutation I could construct." Its own text refutes it. **Burden of proof** — no trigger, so no finding, and no mention.
- "Duplicated Code — the README and the deploy script both delete the results file." Declined in round 1 with a reason ("different paths on purpose"). **The ratchet** — settled, and it stays settled in round 4 wearing a new hat.
