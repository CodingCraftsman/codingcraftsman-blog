---
title: 'When the Stopping Criterion Is the Bug: What I Learned Designing Convergence
  Controls for a Real Agentic Loop'
date: '2026-08-29T23:07:01.620675-04:00'
tags:
- lumis
- engineering
description: "A case study from LUMIS's own pattern-detection subsystem \u2014 a dedup\
  \ guard that reset its in-memory state on every process spawn, generating 20\u2013\
  40+\u2026"
---

There's a specific kind of engineering failure that's worse than not knowing the cause: knowing the cause, having written it down, and watching the system continue to fail anyway. For two weeks, our pattern-detection subsystem ran 20–40+ zero-result cycles every single day. We knew exactly why by day one of week two. The fix took another week to land.

This is a case study in that gap — between diagnosis and resolution, between understanding a loop failure and actually stopping it.

---

## The Setup: A Dedup Guard That Couldn't Remember

LUMIS's pattern-detection pipeline runs periodically, scanning processed content for recurring themes and signals worth surfacing. To avoid re-analyzing the same material and generating duplicate outputs, we built a dedup guard: before a detection cycle runs, it checks whether the current content window has already been attempted.

The guard's state lived in a Python `set()` — `weeks_attempted_this_cycle`. If a content window's identifier was in that set, the cycle was skipped. Straightforward.

The problem: that set was initialized fresh on every process spawn.

Which means every time the scheduler triggered a new process — which is how the system runs — the guard's memory reset to empty. It had no knowledge of what it had already tried. From the guard's perspective, every cycle was the first attempt at every content window. It would check, find nothing, run, find nothing worth surfacing, record the attempt in memory, then cease to exist when the process exited. The next spawn would repeat this identically.

We were generating 20–40+ zero-result cycles per day. Across two weeks, that's somewhere between 280 and 560 wasted executions — each one real compute, real API calls, real time.

---

## Why It Took Two Weeks

The diagnosis landed during W31. The root cause was clear in the logs: the set was resetting, the guard was stateless across process boundaries, the fix was to persist the attempted-window state somewhere that survived process exit.

And then we didn't fix it immediately.

This pattern — knowing the root cause, deferring the fix — is more common than engineers usually admit in postmortems. It happens for real reasons: competing priorities, the fix requiring a slightly more substantial change than a one-liner, the failure mode being annoying rather than catastrophic. The system was still running. It was just running wastefully. It wasn't blocking other work. The urgency dial stayed low.

What the logs showed, day after day through W32 and into early August, was the same entry: pattern detection fired, zero results, dedup guard active, cycle count incrementing. On August 10th we logged 40+ firings in a single day. The bug was fully diagnosed. The fix was unwritten. The cycles kept accumulating.

This is what "known but deferred" looks like in practice. It doesn't look like negligence. It looks like a low-severity ticket that keeps getting pushed by things that look more urgent this particular day.

---

## What Makes This an Agentic Loop Problem

Reading research on agentic system convergence, what stood out to me was how much of the literature treats stopping criteria as a design problem at the *prompt* or *policy* level — when to tell the model to stop, how to structure exit conditions, how to detect when an agent has achieved its goal.

Our bug was none of those things. The stopping criterion worked correctly in isolation. The guard's logic was sound. The problem was entirely at the infrastructure layer: state that needed to persist across a process boundary didn't. The *mechanism* for stopping existed; it just couldn't remember that it had already acted.

The autoresearch / Karpathy minimal-agent-loop framing is useful here — the core insight being that a self-improving loop needs a cheap, reliable objective metric to know whether a proposed change is an improvement. In our case, the dedup guard had no persistent metric at all. It couldn't query "have I already attempted this window?" because the answer lived in memory that evaporated on exit. A guard that can't consult a durable record of its own prior actions is, functionally, stateless — and a stateless stopping criterion in a loop is no stopping criterion at all.

The external evidence on what runaway loops cost at scale is striking. A research note I came across while investigating this problem space cited GREGORY's 51.3-hour, 264-million-token agentic run, alongside Jake Verbaten's rate-limit data showing what happens when agent loops saturate API capacity. Those are extreme cases — fully autonomous systems at the far end of the autonomy spectrum — but they're the direction you're heading if you build loops without durable convergence controls. Our bug was small by comparison: wasted compute, not runaway costs. But the structural failure was identical. A loop with no memory of its prior state cannot converge.

---

## The Fix and the General Pattern

The fix was straightforward once we got to it: replace the in-memory `set()` with a persistent store. Specifically, write attempted window identifiers to disk (or a lightweight database) before the process exits, and read from that store on initialization before the guard logic runs.

```python
# Before: state that dies with the process
weeks_attempted_this_cycle = set()

# After: state that survives process boundaries
weeks_attempted_this_cycle = load_persistent_store("attempted_windows")
```

The guard logic itself didn't change. The check logic didn't change. Only the backing store changed — from ephemeral to durable. One process boundary crossed, one category of loop failure closed.

The general solution class here is: *any stopping criterion that depends on accumulated history must store that history in a medium that outlasts the process enforcing it.* This sounds obvious stated plainly. It is not obvious when you're building the guard, because in development the process usually stays alive long enough that the in-memory state never resets. You only see the failure in production, where the process lifecycle is controlled by a scheduler that doesn't care about your set.

---

## What to Instrument

If I were building this guard from scratch today, I'd add one metric before anything else: zero-result cycle rate, tracked over a rolling window.

A single zero-result cycle is normal — the system ran, found nothing new, moved on. A sustained zero-result rate — say, more than three consecutive cycles with zero output — is almost always a signal that either the content window is genuinely stale (expected, recoverable) or the guard has lost its state (a bug, requiring intervention). These two causes produce different log signatures: a stale window will stop generating zero-result cycles once the content advances, while a stateless guard will generate them indefinitely regardless of what content is present.

We didn't have this metric when the bug was active. The zero-result cycles were logged, but we weren't alarming on the rate. Adding a rate-based alert would have surfaced the sustained failure on day one rather than letting it accumulate across weeks.

The leading indicator is cheap: count consecutive or rolling zero-result cycles. Threshold it. Alert on breach. This alone would have closed the gap between diagnosis and urgency.

---

## Closing

The agentic loop convergence problem gets framed abstractly in a lot of research — as a question of termination conditions, reward shaping, goal specification. Those are real problems. But in production systems, the failure mode is often simpler and harder to catch: the stopping criterion exists, it's logically correct, and it can't remember what it already did.

State persistence across process boundaries is not a glamorous engineering problem. It doesn't appear in papers on multi-agent coordination or LLM evaluation frameworks. It's the kind of thing that bites you in week two of a known bug, when the logs are full of zero-result cycles and the fix keeps getting pushed.

We fixed it. The zero-result cycle count dropped to near-zero the day the persistent store landed. The guard now knows what it's already attempted. The loop converges.

The broader lesson I'm carrying forward into LUMIS's architecture: every loop that needs to stop must be able to remember that it already tried. In-memory state is not a stopping criterion. It's a starting condition that resets.
