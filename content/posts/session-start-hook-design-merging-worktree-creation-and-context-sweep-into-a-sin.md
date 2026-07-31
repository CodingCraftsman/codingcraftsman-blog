---
title: 'Session-Start Hook Design: Merging Worktree Creation and Context Sweep into
  a Single Automated Step'
date: '2026-07-30T23:09:14.394284-04:00'
tags:
- lumis
- engineering
description: "Three vault signals from 2026-07-13 (scores 7-8) all cluster around\
  \ automating the worktree creation + context pre-loading sequence at session start\u2026"
---

I noticed the pattern before I decided to fix it: three separate signals in my own vault, all logged on the same day, all hovering in the 7-8 range, all pointing at the same seam in my session lifecycle. That's usually how these things surface for me — not as a single "aha" but as a cluster of low-grade friction that finally crosses a threshold where it's cheaper to fix than to keep tolerating. The seam was the gap between creating a git worktree for a new task and the moment the agent actually has enough context to do anything useful in it.

## The two-step dance

Here's the workflow I'd been running for months: I'd spin up a new worktree for whatever branch of work I was starting, then wait. The context-injection hook would fire, pull in whatever it was configured to pull in, and only then would the agent be oriented enough to start. Two steps, two separate triggers, and a dead zone in between where I was effectively babysitting the handoff — confirming the worktree existed, confirming the hook had fired, confirming the context that landed was the context I actually needed for *this* worktree and not some stale snapshot from the last one.

It's a small tax. But small taxes paid every session compound into something worth killing. And the failure mode wasn't dramatic — it was quiet mis-orientation. The agent would come up in a fresh worktree with context that assumed the previous branch's state, and I'd burn the first few exchanges just correcting course.

## Why merge instead of tune

The instinct when something's annoying is to tune it — make the context hook faster, make the worktree script smarter. I did some of that. But the three vault signals kept landing on the same underlying claim: this isn't a latency problem, it's a sequencing problem. Worktree creation and context loading are logically one event — "start a new unit of work" — that I'd implemented as two independent triggers that happened to usually fire close together. Treating them as separate meant every edge case (worktree created without the hook firing, hook firing against the wrong directory, race conditions on fast machines) had to be handled twice, in two different pieces of automation that didn't know about each other.

Consolidating into a single hook — worktree creation *is* the trigger, and context loading is a mandatory step inside that same trigger, not a downstream listener — collapses that class of bug entirely. There's no "did the hook fire" question because the hook and the creation are the same event.

## Borrowing patterns instead of inventing them

I didn't want to design this from scratch, partly because I don't trust my own taste in hook architecture when I'm the one annoyed and impatient. The patterns worth stealing from the broader tooling community aren't exotic:

**Worktree hooks** — treating `git worktree add` (or whatever wraps it) as a lifecycle event with pre/post stages, the same way people treat `post-checkout` or `post-merge`. The insight is that the hook shouldn't just run *after* the worktree exists — it should be aware of *why* the worktree was created, which requires passing metadata (branch name, parent task, intent) at creation time rather than inferring it afterward.

**Context manifests** — instead of the injection hook deciding at runtime what context is relevant, a manifest file declares it up front: this worktree corresponds to this task, load these files, skip these. It turns context loading from a heuristic into a lookup, which is both faster and far more debuggable when it goes wrong.

**Progressive disclosure** — don't dump the entire project context at session start. Load a thin orientation layer immediately (current task, recent decisions, open threads) and let the agent pull deeper context on demand. This matters more than it sounds like it should, because the failure mode of over-loading context isn't just token cost, it's that the agent orients around the wrong things first.

None of these are novel ideas. What's useful is that they compose — the manifest tells the hook what to load, progressive disclosure tells it what order to load it in, and the worktree-hook-as-single-trigger tells it *when*.

## Where this sits in the session lifecycle

This isn't an isolated fix — it's one joint in a lifecycle I've been slowly making more explicit: session-start, work, pre-compact flush, session-end. Each of those transitions has its own version of the same problem — state needs to move across a boundary without the agent losing orientation or the human having to manually confirm the handoff worked. The pre-compact flush and the pending MEMORY.md automation work are solving the same class of problem at a different boundary. Getting session-start right first matters because it's the highest-frequency transition — I hit it every time I branch into new work, which is often several times a day — so any friction here gets multiplied more than friction at, say, session-end.

## What's actually hard about this

The honest part: merging the two triggers is conceptually simple and mechanically annoying. Worktree vs. main-tree detection turned out to be the sharpest edge — the hook needs to behave differently depending on whether it's firing in a freshly created worktree or in the main tree after a rebase or branch switch, and those two cases look similar enough at the git level that naive detection gets it wrong. I ended up needing to check for the presence of a worktree-specific marker rather than trusting `git rev-parse` output alone, which cost more time than I want to admit.

Cache invalidation is the other one, unglamorously. If the context manifest gets stale relative to the actual state of the worktree — say the task metadata was written before a rebase changed what "current" means — the hook confidently loads the wrong context, which is worse than loading no context, because it looks correct. I don't have a fully satisfying answer for this yet beyond timestamping the manifest and invalidating aggressively when the underlying branch state changes underneath it.

Hook ordering is the last piece, and it's the one I'm least settled on: if worktree creation and context loading are one event, what happens when context loading fails? Right now it fails loud rather than falling back silently to a stale context, which feels correct but means a broken manifest can block starting work entirely. That's a tradeoff I'm still watching rather than one I'd call resolved.

The bigger pattern here is one I keep running into across this whole system: friction that looks like a UX annoyance is usually a sequencing bug wearing a UX costume. Fixing the sequence tends to be more durable than tuning the friction away.
