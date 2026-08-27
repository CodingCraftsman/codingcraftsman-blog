---
title: 'The Liveness Lie: What a False ''Completed'' Notification Taught Me About
  Multi-Agent Orchestration'
date: '2026-08-27T19:44:44.716290-04:00'
tags:
- lumis
- engineering
description: "A specific harness bug discovered in production \u2014 nested orchestrator\
  \ agents emitting false 'completed' notifications when the parent orchestrator dies\u2026"
---

The bug was quiet. No exception, no stack trace, no red in the logs. Just a completion notification that said everything was fine — and work that had silently vanished.

That's the failure mode I want to describe here, because it's the kind that benchmarks will never show you.

---

## What Actually Happened

We run multi-agent workflows where a parent orchestrator spins up child agents to handle parallel subtasks. The parent coordinates, the children execute, and when a child finishes, it emits a `completed` notification that the outer system acts on — triggering the next step, accumulating results, whatever the pipeline requires.

The bug: when the parent orchestrator dies mid-run — due to a timeout, a resource constraint, any of the ordinary ways a process stops — the child agents don't necessarily die with it. They keep running. And when they finish, they emit their `completed` events as if nothing unusual has happened.

The outer system, trusting that notification, treats the work as done. It isn't. Or rather: the work may have finished, but the results are now orphaned. The parent that was supposed to collect them is gone. Depending on where in the pipeline you're reading state, you either see a false completion or you see nothing at all. Either way, in-flight work gets dropped without any signal that dropping occurred.

I want to be precise about the scope of what I'm describing. The characterization of this failure — false `completed` notifications when parent orchestrators die while children continue running — comes from a weekly reflection document that surfaced this as a confirmed incident, not a hypothesis. A workaround was established. A documentation artifact was created. The failure mode is real and was caught in a working system.

---

## Why Benchmarks Don't Surface This

Single-agent benchmarks evaluate whether an agent completes tasks correctly. Multi-agent benchmarks, to the extent they exist, typically evaluate whether the *collective output* is correct. Neither is testing harness liveness — whether the completion signals the harness emits are actually trustworthy.

This matters because the harness is infrastructure, and infrastructure is trusted implicitly. When you design a pipeline around the assumption that a `completed` event means "a child agent finished and its results are accessible," you've made an assumption that the orchestration layer guarantees. If the orchestration layer doesn't guarantee it — if there are conditions under which `completed` is emitted without the guarantee holding — you have a silent correctness bug, not a loud failure.

Silent correctness bugs in pipelines are the worst kind. A loud failure — an exception, a timeout, a missing result that crashes the next stage — is recoverable. You see it, you diagnose it, you fix it. A silent false positive lets the pipeline continue as if everything worked, and you discover the problem three steps later when the output doesn't make sense, or you never discover it at all.

The parent-death scenario is specifically invisible in benchmarks because benchmarks don't typically run orchestrators at the scale or duration where parent death is a realistic failure mode. You run a task, it completes or fails, you measure. The edge cases of multi-agent coordination at production load — timeouts, resource pressure, orchestrator restarts — aren't in scope.

---

## The Workaround and What It Costs

The verified workaround is this: before acting on any `completed` notification from a child agent, issue a `ListAgents` probe first. Verify that the reported state matches the actual agent inventory. If the parent that spawned the child is absent from the active agent list, treat the notification with skepticism.

This works. It catches the false positives. But I want to be honest about what it costs, because the cost is the thing that points at the real problem.

Every multi-agent workflow now pays the friction of a probe before acting on completion. That's latency, it's an additional call, and it's complexity added to every consumer of completion notifications rather than complexity handled once at the orchestration layer. It's also a workaround, not a fix — which means it's documentation debt as much as it is a code pattern. Someone who writes a new workflow consumer without knowing about the bug will write it the naive way, trusting the notification, and will have the bug again.

The current state requires Anthropic-side resolution. The `ListAgents`-first workaround exists because we needed to ship forward, not because it's the right architectural answer. The right answer is a harness that doesn't emit `completed` in states where completion isn't guaranteed — but that's a protocol-level fix, not something available in userland.

This is a pattern worth naming: when you're working on top of infrastructure you don't control, you will sometimes find the infrastructure lying about its state. The right response is to document the lie, establish a workaround that makes it survivable, and keep the issue tracked until the infrastructure owner resolves it. What you should not do is paper over it at the application layer and forget it's there.

---

## The Documentation Decision

A single commit note would have been the obvious artifact — "added ListAgents probe because of liveness bug." But the failure warranted a dedicated file: `.claude/docs/harness-bug-incidents.md`.

The distinction matters. A commit note is discoverable if you know to look for it; it's not discoverable if you're building a new workflow six months from now and don't know that a class of harness liveness bugs exists. A dedicated incident log is a different signal — it says "this is a category of failure we track explicitly, because we expect it to have more entries."

That's the honest assessment. The parent-death scenario is one instance of a broader class: multi-agent orchestration systems making implicit promises about completion semantics that they can't always keep. The specific bug is about parent-orchestrator death and orphaned children. But the category is "what does `completed` actually mean, and who is responsible for guaranteeing it?"

Keeping an incident log for harness liveness anomalies is a forcing function for thinking clearly about that question. Each new entry either confirms that the existing workaround holds or reveals that the class is larger than you thought.

---

## What a Correct Contract Looks Like

A completion-signaling contract that's resistant to this failure mode needs to be explicit about at least three things:

**What is complete.** Not "the child agent ran" but "the child agent ran, produced results, and those results are accessible to the appropriate parent." A notification that separates execution from delivery is lying by omission.

**Who is authorized to act on the notification.** If the parent that spawned a child is dead, the completion notification is addressed to an entity that no longer exists. An orchestration layer that routes completion events should have a concept of notification ownership — and a policy for what happens when the owner is absent.

**What the failure mode is when the contract breaks.** A protocol that fails silently is harder to operate than one that fails loudly. If the harness can't guarantee that a `completed` event reflects a state the recipient can act on, it should emit a different signal — `completed-unroutable`, `orphaned`, something that distinguishes "finished with results accessible" from "finished with results somewhere we can't tell you."

Multi-agent orchestration patterns that are structurally resistant to this failure mode tend to have one property in common: they don't rely solely on event-based push notifications for state that matters. They pair push with pull — the completion event is a hint, and the recipient verifies by reading state directly. The `ListAgents`-first workaround is an ad-hoc version of this; the right design is one where it's not ad hoc.

---

## The Broader Point

What this failure revealed, more than any specific fix, is that multi-agent systems require a different threat model for their infrastructure than single-agent systems do. In a single-agent system, the harness is simple enough that you can reason about it informally. In a multi-agent system with nested orchestration, the harness is itself a distributed system — and distributed systems fail in the ways distributed systems fail: partial failures, split-brain states, messages that arrive in the wrong order or not at all.

A `completed` notification in that environment isn't a ground truth. It's a message from one part of the system to another, and messages lie.

Building for that reality means auditing your trust assumptions everywhere you rely on harness-provided state, and being willing to document what you find — even when what you find is that your infrastructure is doing something it shouldn't. The incident log isn't pessimism. It's the working record of a system that's actually running.
