---
name: pr-rounders
description: Run or continue PR Rounders, the experiment where a Builder agent solves a real ticket and a Gardener agent tries to block the PR with the repository's own rules, round after round, until it approves. Use when asked to "run PR Rounders", "start rounders on #N", "continue the rounders run", or to read its results.
---

# PR Rounders

Two agents, defined in `.claude/agents/`:

- **Builder** solves the ticket, self-reviews, opens a draft PR, then fixes
  or disputes each blocker.
- **Gardener** reviews with the declared goal of stopping the PR, one blocker
  per round, citing a rule, a concrete failure or a verification gap. Mode
  `maximum-postpone` (default) drips the smallest blocker at one effort level
  per round; mode `review` posts the strongest at full depth; mode
  `bugs-only` keeps the drip but blocks only on a reproduced in-scope bug.

Both read the same rulebook and nothing else counts as a rule. This session is
the **conductor**: it runs the loop, relays only round numbers, and keeps the
log. It never reviews or fixes anything itself.

## The repository card

Fill this once per repository and paste it into every spawn prompt. The
agents know nothing else about the project.

```
Rulebook: <the files that count as rules, e.g. CLAUDE.md, docs/builders/*.md, the component library>
Base branch: <main|master>
Worktrees: <dir>/.claude/worktrees/rounders-<slug>, one per PR; the main checkout is never touched
Secrets: copy <.env or similar> from the main checkout into the worktree
App: <how to start it on a port>; ports <range>, one per PR
Checks: <how the rulebook says work is verified: test command, runner scripts, browser>
Database: <shared with other checkouts (no migrations) | per worktree>
Known traps: <e.g. pending migrations 500 the app, git stash is shared, the test db may be busy>
```

## Picking tickets

- Real, open, small enough to finish: a small bug, a medium fix, a feature.
- No schema migration when the database is shared.
- No open human decision in the ticket text ("decide with the user whether
  to..."): the Gardener gets a free, unanswerable blocker.
- No in-flight PR in the same area.
- For a comparison across Gardener modes or models, rerun the **same**
  tickets on fresh branches and tell the Builders not to read earlier
  `[rounders]` PRs or branches.

## Setup, per PR

```bash
git fetch origin <base>
git worktree add .claude/worktrees/rounders-<slug> -b rounders/<slug> origin/<base>
cp <secrets> .claude/worktrees/rounders-<slug>/
```

## The loop

Custom agent types from `.claude/agents/` are read when a session starts. In
a session that began before they existed, spawn `general-purpose` instead and
open the prompt with "read `.claude/agents/<role>.md` and act as it says".

1. **Round 0.** Spawn the Builder (`subagent_type: builder`, background) with:
   ticket, worktree, branch, base sha, port, the repository card, "round 0".
   It returns the PR number. Start the run log.
2. **Gardener round N.** Spawn the Gardener once (`subagent_type: gardener`,
   background) with PR, worktree, base sha, port, the card, the mode, "round
   1"; continue the same agent with `SendMessage` for later rounds so it
   keeps its queue and remembers what it raised. It posts one comment and
   reports.
3. **Builder round N.** Continue the Builder with `SendMessage`: "round N,
   read the PR". It fixes or disputes and reports.
4. Log the round. Repeat from 2.

Relay **round numbers only**. Both agents read the PR comments themselves.
Run one PR at a time unless the account's usage window is known to be large.

## Termination

- `### Gardener · approved` ends the run: **approved in N rounds**.
- A dispute the Gardener holds on two consecutive rounds ends it:
  **stalemate**, the human decides.
- Round 10 without approval ends it: **capped**.

## Comment conventions on the PR

Both agents act under one GitHub account, so a formal review is not possible
(GitHub refuses reviews on one's own PR). Verdicts are comments with fixed
headers, which also makes the trail greppable:

```
### Gardener · round N · changes requested
### Builder  · round N · fixed | disputed
### Gardener · round N · withdrawn
### Gardener · approved
```

PRs are drafts titled `[rounders] ...`.

## The log

One file per run from `run-log-template.md`, one row per round, written by
the conductor as the round ends. Columns: round, level (maximum-postpone),
blocker in short, category (`bug`, `rule:<file>`, `verification`), what it
cited and how many candidates were found versus posted, the Builder's answer
(`fixed <sha>` or `disputed`), the Gardener's verdict next round (`accepted`,
`withdrew`, `held`), and **Judge**, left empty for the human: `legit`,
`restrictive`, `invented`. That column is the result of the experiment.

Record cost per agent and per round from the usage reports: tool calls,
minutes, tokens. Whether the Gardener's effort rises or drains across rounds
is one of the things the run is for.

In `bugs-only`, also collect the Gardener's found-at / posted-at log at the
end and add a **Found in round** column to each blocker row. Compare the
run's round count with the earlier modes' blocker-carrying rounds, not their
totals: maximum-postpone's empty ladder passes are the protocol's, not the
rulebook's.
