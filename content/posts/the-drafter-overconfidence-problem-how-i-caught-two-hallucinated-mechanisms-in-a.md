---
title: 'The Drafter Overconfidence Problem: How I Caught Two Hallucinated Mechanisms
  in a Single Week of Agent-Written PRDs'
date: '2026-08-23T00:08:01.060997-04:00'
tags:
- lumis
- engineering
description: "A concrete case study from building LUMIS: two instances in one week\
  \ where the PRD-drafting agent confidently described non-existent technical mechanisms\u2026"
---

There's a correction block sitting in one of my research notes that I find myself coming back to. It reads, roughly: *the PostToolUse+compact hook matcher described above does not exist — verified against the official Claude Code hooks reference*. What makes it worth examining isn't the error itself. It's that the error was detailed, specific, and structurally indistinguishable from something correct. The drafter hadn't said "hooks might support this" or "consider whether compact events are catchable." It had described a concrete mechanism, named it, and implied it was ready to implement.

That note is where I started auditing my drafting pipeline more carefully. Within the same week, I found a second one.

---

## What the Failure Mode Actually Looks Like

The standard mental model of LLM hallucination is vagueness: hedged claims, made-up citations with approximate-sounding titles, plausible-but-unverifiable statistics. That model trains you to watch for the wrong signal.

What I was seeing in LUMIS's PRD-drafting agent output was the opposite: high specificity, appropriate technical register, references that *sounded* like they came from someone who had read the actual API documentation. The first incident involved a PostToolUse+compact hook matcher — a mechanism described as though it were a real hook event type in the Claude Code hooks reference that you could pattern-match against to catch session compaction events. The drafter laid out the matching logic, described the expected payload shape, and positioned it as the foundational mechanism for a context-restoration flow.

It doesn't exist. The research note's own correction memo documents this explicitly, verified against the Claude Code hooks reference directly. There is no compact event catchable via PostToolUse hooks — at least not as described. The mechanism was fabricated in the precise technical idiom of the thing it was pretending to be.

This matters because the failure mode defeats the usual review heuristic. When something is vague, reviewers probe it. When something is specific and confident, reviewers tend to anchor on the specificity as evidence of correctness. The drafter's confidence wasn't a bug in the prose — it was a feature working against the reader.

---

## Two Instances, One Week, One Pattern

The second incident that week involved what I'll call the prd-recon fork-mode claim — a description of a behavioral mode in the PRD reconnaissance agent that, again, was described specifically and confidently, and again did not correspond to anything in the actual codebase or agent configuration.

I'm not going to spend time on the details of the second mechanism because the details are less important than the shape: same week, same drafter, same failure structure. The agent described a fork-mode for the recon step that would spin up parallel exploration paths under certain input conditions. This was presented not as a proposal but as a description of how the system *currently* works, in implementation-plan language.

When two instances hit in one week with the same structure, you're no longer looking at a one-off. You're looking at a category. The drafter overconfidence problem isn't "the model made a mistake" — it's "the model generates plausibly-detailed descriptions of non-existent mechanisms as a routine output pattern, and the output format gives you no signal that this is happening."

---

## Why This Is Hard to Catch in Practice

The uncomfortable part of documenting this is that I didn't catch either instance during the drafting session itself. Both got caught during review — in the first case, because I happened to be cross-referencing the Claude Code hooks documentation while working on a related design question, and noticed the mechanism wasn't there. In the second, because I'd been sensitized by the first and started specifically interrogating behavioral claims in the same draft.

Neither detection was systematic. Both were lucky.

The fundamental problem is that the agent's output doesn't degrade gracefully at the boundary between "things it knows" and "things it's confabulating." A human engineer writing a PRD will typically mark uncertainty with hedges or TODO flags — "assuming this hook type exists, verify before implementation" or "need to confirm fork-mode is supported." The drafter doesn't do this. It writes the uncertain parts in exactly the same register as the parts it's confident about.

This means the signal you'd want — confidence calibration — isn't present. The only reliable verification path is external: check the claim against the actual spec, the actual codebase, the actual API reference. Not against the model's own reasoning chain, which will happily rationalize the fabricated mechanism if you ask it to defend the draft.

---

## What Got Codified, and What Didn't

After the second incident, I added a constraint block to `prd-drafter.md` — the prompt document that governs the PRD-drafting agent's behavior. The mitigation is essentially a requirement that any mechanism claim in a PRD draft be flagged with a verification status: either "confirmed against [specific source]" or "unverified — requires ground-truth check before implementation."

This is a partial fix. It moves the problem from invisible to visible: a PRD that says "unverified mechanism — check before shipping" is better than one that says "mechanism X works as follows" when mechanism X doesn't exist. But it relies on the drafter actually applying the flag consistently, which is its own LLM reliability problem.

What I don't yet have is a systematic pre-review checklist that runs before a drafted PRD moves to implementation. The shape of what that would need to cover is becoming clearer: for every mechanism claim in a PRD, there should be a verification step that resolves to a ground-truth source — a file path, a documentation section, a live test result. Not a reasoning trace. Not a "the model confirmed this when asked." An external anchor.

The contrast case is useful here. A separate design session that week involved a session_id-continuity assumption in the context-restore hook design — and that one got verified live, against actual observed agent behavior. The PRD that came out of that session is trustworthy in a way the PostToolUse+compact draft wasn't, and the difference is traceable: there's a verification artifact that exists outside the model's output.

---

## Treating Drafts as Hypotheses

The builder takeaway I keep coming back to is simple but requires a real adjustment in workflow posture: LLM-generated implementation plans are hypotheses about the system, not descriptions of it.

A PRD draft is not a spec you run. It's a structured prediction about what *would* need to be true for the described feature to work. Every mechanism claim is a hypothesis. Some of those hypotheses are correct. Some are confabulated with full confidence and zero basis. The draft format doesn't tell you which is which.

The discipline this requires — treating every mechanism claim as needing external verification before it becomes implementation work — is friction. It slows down the cycle between "idea" and "code." But the alternative is shipping code against a spec that describes a non-existent API, which is expensive in a different way.

I'm building toward a pre-implementation checklist that makes this verification step explicit and trackable, rather than something I do inconsistently based on how recently I've been burned. One week, two incidents, one documented pattern. That's enough evidence to build the check — not just remember to do it.
