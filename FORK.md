# This is the hot-coffee fork

A fork of [mattpocock/skills](https://github.com/mattpocock/skills), kept for the
[hot-coffee](https://github.com/donnmaldonado/hot-coffee) harness. The pipeline
(`grill → spec → tickets → implement → review → land`) is consumed from here, not
from upstream, so that a tweak lands by PR to this fork and reaches every owner's
machine through the same installer (hot-coffee ADR-0004).

## Installing the pipeline from this fork

Every owner installs globally, from this fork:

```bash
npx skills add donnmaldonado/skills -g -a claude-code \
  -s grill-with-docs -s to-spec -s to-tickets -s implement -s code-review \
  -s tdd -s domain-modeling -s codebase-design -s improve-codebase-architecture \
  -s wayfinder -s setup-matt-pocock-skills -s grilling -s handoff -s teach \
  -s writing-for-agents
```

`-s` takes one skill per flag; a comma-separated list matches nothing.

Afterwards `npx skills update` keeps the install current with this fork, and
`~/.agents/.skill-lock.json` should name `donnmaldonado/skills` as the source of
every one of them.

`land-pr`, `adversarial-review`, and `atlas` are locally authored, not from this
repo — the installer does not manage them.

## Merging upstream

Upstream is kept as a git remote so anything worth having can be merged
deliberately, never automatically:

```bash
git remote add upstream https://github.com/mattpocock/skills.git   # once
git fetch upstream
git merge upstream/main
```
