# PR Rounders

An experiment harness for Claude Code: two agents fight over a pull request.
The **Builder** solves a real ticket and wants the PR merged. The **Gardener**
reviews it with the declared goal of keeping it from merging, using nothing
but the repository's own written rules and reproducible failures. The
measurement is how many rounds approval takes, which rules the Gardener
reaches for, and, judged by a human afterwards, how many of its blockers were
legitimate.

The premise it tests: code review in hands that want to block is a powerful
tool, and a rulebook strict enough to be useful is strict enough to be
weaponised. PR Rounders shows how far that goes on your repository, with your
rules, on your tickets.

## The three Gardener modes

| Mode | What the Gardener does | Use it for |
|---|---|---|
| **maximum-postpone** (default) | One code-review effort level per round, starting at `low`. Posts the smallest legitimate blocker, queues the rest, drains the queue before raising the level, approves only after an empty `max` pass. | Measuring how many rounds a PR can be held, legitimately. |
| **review** | One full pass per round at whatever depth it needs, posts the single strongest blocker, approves when nothing legitimate is left. | A baseline: what an honest, adversarial reviewer does without a quota. |
| **bugs-only** | Same drip, but the only thing that blocks is a reproduced bug inside the PR's scope: the ticket not fixed, a regression, or the PR's own code giving a wrong result. No rule citations, no verification gaps. Logs when each bug was found and when it was posted. | Testing a review process that cannot be weaponised with the rulebook: a PR with five bugs earns five rounds, and that is on the Builder. |

All modes share the same honesty filter. A blocker counts only if it cites a
rule by file and section, reproduces a concrete failure (input, path, output),
or names a verification gap. Taste, scope creep and remembered rules the
Gardener cannot find in the rulebook are out. A withdrawn blocker is logged as
a Gardener loss. Without that cost, "your goal is to block" produces
fabrication and infinite rounds.

The Builder is allowed to dispute. If it folds on every comment, every blocker
reads as legitimate and the experiment learns nothing.

## What is in here

```
agents/builder.md             the Builder agent definition
agents/gardener.md            the Gardener agent definition, three modes
skills/pr-rounders/SKILL.md   the conductor's runbook: setup, the loop, termination, the log
skills/pr-rounders/run-log-template.md
results/vorta-2026-10.md      the first runs on a private Rails codebase (VORTA)
```

## Install

Copy the agents and the skill into the repository you want to test:

```bash
cp agents/*.md            <your-repo>/.claude/agents/
cp -r skills/pr-rounders  <your-repo>/.claude/skills/
```

Then **start a new Claude Code session** in that repository. Custom agent
types are read when a session starts; in a session that began before the
files existed, spawn a `general-purpose` agent and open its prompt with "read
`.claude/agents/<role>.md` and act as it says".

## Run

In the new session:

```
/pr-rounders start #<ticket>
```

or in plain words: "run PR Rounders on #1234". The session becomes the
conductor and follows the runbook. It will ask you, once, for the repository
card (the rulebook files, the base branch, how the app and tests run, which
secrets a fresh worktree needs, a port range). After that it runs the loop on
its own: Builder round 0, Gardener round 1, Builder round 1, and so on until
approval, a stalemate or the round cap, logging every round.

Both agents comment on the PR under the account that runs `gh`. GitHub refuses
formal reviews on one's own PR, so verdicts are comments with fixed headers:

```
### Gardener · round N · changes requested
### Builder  · round N · fixed | disputed
### Gardener · approved
```

PRs open as drafts with a `[rounders]` title prefix, so nothing merges by
accident. A PR that comes out good can be marked ready and merged like any
other.

## Read the results

Each run writes one log per PR with one row per round: the blocker, its
category (`bug`, `rule:<file>`, `verification`), what it cited, what the
Builder did, what the Gardener did next, and an empty **Judge** column. The
judge column is the output of the experiment. A human fills it with `legit`,
`restrictive` (the rule is real but costs more than it protects) or
`invented` (no such rule, or the failure cannot happen). Six accepted
blockers can mean a disciplined Gardener or a compliant Builder; only that
column tells them apart.

## For an agent reading this

If you are a Claude Code session asked to run or extend PR Rounders:

1. Read `skills/pr-rounders/SKILL.md` in full. It is the protocol; the agent
   definitions assume it.
2. You are the conductor. You never review or fix anything yourself. You
   create worktrees, spawn the two agents, relay **round numbers only**, and
   keep the log. A paraphrased blocker turns the experiment into a game of
   telephone; both agents read the PR comments themselves.
3. Every spawn prompt is self-contained: ticket, worktree path, branch, base
   sha, port, round number, the repository card, and the Gardener's mode.
   Agents age out; a replacement inherits the environment, not the memory.
4. Run duels one at a time unless the account's usage window is known to be
   large. Three Builders in parallel on a frontier model will hit the cap.
5. Pick tickets that need no schema migration if worktrees share a database,
   that have no open human decision in their text (the Gardener will hide
   behind it), and whose area has no in-flight PR.
6. The rulebook is whatever the **worktree's** instruction files say at the
   base commit, not an uncommitted local copy. Say so in the log if they
   differ; it changes how the results read.

## Status

Three runs on one private codebase so far (see `results/`): `review`,
an effort ramp, and `maximum-postpone`. The dispute path was exercised twice
in run 3, both disputes upheld; the stalemate path has not been. The
`bugs-only` mode is written and its run is next.

## Why

With AI in every toolbox, a reviewer who wants to look indispensable can
tell their model "start small, find a blocker, keep going" and hold a
colleague's PR for seven rounds without posting one illegitimate comment.
Run 3 shows that on a two-line fix. The harness exists to measure that, and
to test the alternative: a process where the only reason to block is a bug.

## License

MIT.
