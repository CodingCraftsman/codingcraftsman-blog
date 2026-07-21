---
title: 'The Per-File Lint Tax: Why I Moved to Session-Final Passes and Saved Thousands
  of Tokens'
date: '2026-07-20T19:19:59.085132-04:00'
tags:
- lumis
- engineering
description: "A specific, measurable inefficiency in real agentic coding sessions\
  \ \u2014 the triple-invocation lint cycle (check \u2192 auto-fix \u2192 re-check)\
  \ run after every\u2026"
---

Here's the session log entry that started this investigation: Bash call count for a routine module-writing session, twelve files modified, came out at 47. It should have been closer to 20. The delta — 27 extra tool invocations — traced back entirely to linting.

I'd instrumented LUMIS's agentic sessions well enough by that point to see the pattern clearly once I looked for it. Every file write triggered the same three-step sequence: `ruff check` on the just-modified file, `ruff --fix` on it, `ruff check` again to confirm the fix landed. Multiply by twelve files, and you've got 36 lint invocations in a session that genuinely needed one pass at the end. The remaining 11 Bash calls were the actual work.

This post is an autopsy of that pattern — where it comes from, what it costs, and what replaced it.

---

## What the Triple-Invocation Cycle Actually Looks Like

The sequence is mechanical enough that it's easy to miss while it's happening:

1. Write or edit a file
2. `ruff check path/to/file.py` — surface any issues
3. `ruff check --fix path/to/file.py` — auto-remediate what can be auto-remediated
4. `ruff check path/to/file.py` — confirm clean

That's three tool calls per file, every time, regardless of whether the file has issues. In a session with *N* modified files, you're paying 3N lint invocations. In the twelve-file session above: 36. In a heavier refactoring session I pulled from the logs — 23 files touched — it was 69 lint calls before I caught it.

The invocation count matters for a reason that isn't obvious if you think of tool calls as free: in an agentic session, every tool call and its output persists in the conversation history. The context window accumulates not just the lint results but the scaffolding around each call — the tool invocation itself, any stdout, the response. Ruff output for a clean file is terse, maybe 50-100 tokens. But 69 calls at even 80 tokens each is 5,520 tokens of lint scaffolding, appearing before any of the actual code review or integration work happens. In a session that might have a 100K token budget, that's non-trivial — and it's entirely waste, because the session-final pass would have caught any real issues anyway.

---

## Why This Pattern Exists

I want to be honest about something: this anti-pattern isn't irrational. It emerges from a reasonable engineering instinct — you want to know immediately if the file you just wrote is clean, because catching an issue close to the point of introduction is cheaper than catching it later when context has shifted.

That instinct is correct for humans working interactively. When you're in a flow state editing a file in an IDE, lint-on-save gives you a tight feedback loop with minimal cost. The keypress is cheap; the feedback is immediate; you fix it while the file is still open and in your head.

The mistake is importing that instinct unchanged into an agentic session, where the cost structure is fundamentally different. The "keypress" equivalent — a Bash tool call — isn't free. It consumes tokens, inflates the conversation history, and compounds as the session grows. The feedback loop that feels tight at the human level is actually deferred through the conversation turn cycle, which means you're paying the cost without getting the immediacy benefit that justifies it.

The underlying anxiety is: *what if there's a lint issue in this file and I don't catch it before I move on to the next one?* The answer, in a session with a proper final gate, is: *you'll catch it at the gate, with full session context, and fix it then.* The per-file pass doesn't actually reduce the number of times you fix issues — it just adds verification passes that come back clean 80% of the time.

---

## The Compounding Problem: Import One-Liners

While I was investigating the lint pattern, I found a parallel one that follows the same structure for a different reason.

After writing a new module, the session logs showed a recurring single-line Python invocation: `python -c "from lumis.some.module import SomeClass; SomeClass()"` — a quick instantiation check to confirm the module was importable without obvious errors. Per file. Every time.

This is the same anxiety in a different form. *Did I introduce a circular import? Did the `__init__.py` wire up correctly?* These are legitimate questions, but they're also questions that `pytest` answers at session end, more thoroughly, with actual assertions rather than "didn't crash on import." The per-file one-liner is a local proxy for a test that already exists — it's just being run early, at higher token cost, with less coverage.

Both anti-patterns share a root cause: **local feedback anxiety in the absence of trust in the integrated test cycle**. If you believe the session-final gate is fast, comprehensive, and will actually catch problems, you don't need the per-file proxies. If you don't believe that — if the gate is slow, or flaky, or incomplete — the per-file checks feel necessary because they're the only feedback you trust.

The fix for the anxiety, then, is partly workflow (consolidate to session-final passes) and partly quality gate (make the gate trustworthy enough to rely on).

---

## What the Session-Final Gate Looks Like

The replacement pattern got formalized in LUMIS's `execute.md` after the Token Efficiency Sprint completed. The gate runs as `make qa` or `make qa-quick` depending on whether the session has modified tests:

```
ruff format .
ruff check . --fix
pyright
bandit -r lumis/ -ll
```

Sequence matters here. `ruff format` first — normalize whitespace and formatting so the subsequent `ruff check` is looking at semantically stable code. `ruff check --fix` auto-remediates what it can, then exits non-zero if anything remains that requires human attention. `pyright` runs after ruff because type errors are more expensive to surface and fix, and you want the code clean before you start reading pyright output. `bandit` runs last because security findings are high-signal and you don't want them buried under style noise.

The whole sequence runs in 15-25 seconds on the LUMIS codebase at its current size. That's fast enough that waiting until session end feels like no penalty at all — which is the threshold you need for the behavioral change to stick. If it took two minutes, the per-file proxies would creep back in because the session-final gate would feel too heavy to invoke speculatively.

`make qa-quick` drops pyright from the sequence for sessions where type coverage hasn't changed — module additions without interface changes, documentation updates, configuration tweaks. The distinction keeps the fast path fast.

---

## What Didn't Work Immediately

The first version of the consolidated gate had an ordering bug: `ruff check --fix` ran before `ruff format`, which meant the formatter occasionally re-introduced style issues that the check had just marked clean. The outputs contradicted each other in a way that was confusing to read and required a second pass. Swapping the order — format, then check — resolved it, and in retrospect the correct ordering is obvious, but it took one bad session to surface it.

The more persistent issue is that the session-final pattern requires the agent (or developer) to actually trust the gate and not reach for per-file verification out of habit. The instrumentation data from the week after implementing the pattern showed per-file invocations dropping but not disappearing immediately — they fell from 36 per session to about 12 before reaching near-zero around day five. Behavioral patterns in agentic workflows have inertia, even when the replacement pattern is strictly better on paper.

---

## The Broader Point

This investigation started as "why is my Bash call count so high" and ended as a lesson about cost structure asymmetry. The same feedback-loop intuitions that are adaptive in interactive development become expensive habits in agentic sessions, because tool calls aren't free — they're context, they're tokens, they're history that every subsequent turn has to carry.

The session-final gate pattern isn't a novel idea. Pre-commit hooks, CI gates, merge requirements — these are all versions of the same consolidation. What's different in agentic workflows is that the cost of *not* consolidating is measured in something more concrete than developer annoyance: it's measurable token overhead, quantifiable context window pressure, and real dollars at scale if you're running many sessions.

Knowing that changes how aggressively you should work to get the gate right. A fast, trustworthy session-final gate isn't a nicety — it's the thing that makes the entire agentic workflow economical.
