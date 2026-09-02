---
title: 'The Claude.md Size Trap: What I Learned Maintaining a Live Agent Instruction
  File Across 15+ PRD Cycles'
date: '2026-09-01T20:04:27.073090-04:00'
tags:
- lumis
- engineering
description: "A case study grounded in operating LUMIS through 15+ shipped PRDs over\
  \ two weeks, where the CLAUDE.md (agent instruction file) grew with every arc\u2026"
---

There's a moment every engineer running an agent system hits where they open their instruction file, scroll for longer than they expected, and feel a vague dread. I hit it somewhere around PRD cycle twelve. Our CLAUDE.md — the file that tells every agent in the LUMIS cluster who they are, how they behave, what conventions to follow — had grown past the point where I could hold it in my head. Not catastrophically. Not in any way that triggered an alert. That's the trap.

---

## How the File Grows

The accumulation pattern is almost virtuous at first. You ship a PRD arc. Something breaks or surprises you. You write a rule. You move on.

After the drafter hallucinated a mechanism in week one, I added a rule about verification requirements for any described system behavior. After a commit discipline slip, I added a rule about message format. After an edge case in the Beacon pipeline, I added a scoping rule. Each of these was correct in isolation. Each addressed a real failure. Over fifteen-plus PRD cycles in two weeks, that's fifteen-plus rounds of rule addition, with pruning that I kept deferring because there was always a higher-priority arc in flight.

The result isn't a corrupted file. It's an *aged* one. Rules that were written for an earlier system state — before certain conventions were formalized, before the deterministic-over-LLM principle became load-bearing, before the commit ceremony was automated — sit alongside rules written last week, and the file doesn't tell you which is which. The agent can't tell either.

---

## What the Research Says vs. What I Observed

A video transcript from Better Stack (August 2026) outlines twelve rules for writing effective CLAUDE.md files, with a hard recommendation to keep the file under 500 lines. The reasoning is sound: beyond that threshold, you start competing with your own instruction set for context window space, and the coherence of the agent's behavioral model degrades. The recommendation to treat the file as a living failure log — rules earned through incidents, not rules written speculatively — is exactly the right framing.

Boris Cherny, in a separate transcript from Cole Medin's channel, goes further: periodically delete your rules and rebuild line-by-line, keeping only what you can justify. Reading that the first time, it landed as obviously correct. A rule that survives deliberate reconstruction is a rule that still deserves to exist.

The problem, which Cherny acknowledges and which a caveat in that transcript makes explicit, is that ablation is expensive. Not expensive in engineering time alone — expensive in tokens. Walking an agent through your existing rule set, testing each rule against current system behavior, and deciding what to cut requires the kind of extended reasoning session that isn't cheap in a cost-constrained system. For an internal team at Anthropic, that's a different calculation than for a single operator running LUMIS on a real budget.

What I observed in practice: I deferred the ablation session across at least four cycles where I knew it was overdue. Not because I forgot. Because there was always an active arc that consumed the token budget I would have needed to run it properly. The ablation cost problem is real, and it's worse during high-throughput weeks — which are exactly the weeks when rule accumulation is fastest.

---

## The Stale-Rule Failure Mode in Practice

During week 32, the drafter hallucinated mechanisms twice in one week. Not the same mechanism, not obviously the same failure pattern — but close enough that when I wrote the postmortem, I codified it as a recurring failure mode and pushed a rule into prd-drafter.md.

When I went back and looked at the CLAUDE.md state at the time, the verification language was there — but it was sitting next to an older framing from an earlier arc that implicitly licensed more confident generation. The two rules weren't in direct conflict. They were just in *tension*, in the way that rules written months apart by different versions of the same system often are. The agent wasn't ignoring the verification rule. It was operating under an instruction set where the overall prior was more permissive than the specific rule implied.

This is the subtle failure mode. The agent doesn't crash. It doesn't throw an error. It produces output that's slightly miscalibrated in a direction that's hard to attribute to any single rule. You see overconfidence in the draft. You write a new rule. The file gets longer.

---

## The Auto-Improvement Attempt

Partway through the build, I tried to make CLAUDE.md partially self-maintaining. The mechanism was simple: failure-log entries would trigger a proposed rule addition as part of the post-incident workflow, with the agent drafting the proposed addition and flagging it for human review before merge.

That part worked. Rules were added through a more deliberate process than pure append. What didn't work was the other half: the contradiction-detection step. The plan was that any proposed addition would be checked against the existing file for rules that it superseded or conflicted with. In practice, that check never ran autonomously. It required a prompt I kept meaning to make automatic and never did. So we got the intake process without the pruning process — which is strictly better than pure append, but not by as much as I'd hoped. The file still accumulated. The contradictions still accrued. The auto-improvement mechanism was half-built.

---

## Past 500 Lines: What Degradation Actually Looks Like

I want to be specific about what I observed, because the 500-line threshold sounds like an arbitrary number until you see what happens near it.

The degradation isn't a sudden behavioral shift. There's no commit you can point to where the agent started behaving differently. What you notice instead is a kind of *response averaging*. Instructions that conflict get honored partially. Conventions that were meant to be strict become tendencies. The agent starts producing output that satisfies most of the rules most of the time rather than all of the rules all of the time — which, depending on what the rules are, can be completely invisible in the output until you're in a postmortem asking how the hallucinated mechanism got through.

The Beacon pipeline surfaced this most clearly. Two new drafts landed in content/drafts/ overnight and were publication-quality — which is the good news. The angle selection and structural conventions were followed correctly. But a behavioral detail from an older CLAUDE.md entry about citation framing showed up in one draft in a form we'd explicitly deprecated in a later session. No one caught it in review because no one remembered the deprecation. It wasn't in the current conventions doc. It was in the file history.

---

## The Convention That's Actually Helping

The intervention that's made the most difference isn't ablation — I still haven't run a proper ablation session, and I'm not going to pretend otherwise. It's treating CLAUDE.md as a versioned artifact with a diff-reviewed update ceremony.

Concretely: no rule gets added as a direct edit. Every proposed addition goes through the same lightweight process as a code change — a diff, a one-line rationale, a check against the most recent five rules for redundancy. This doesn't prevent accumulation, but it slows it. It also creates a readable history that makes the next ablation session feasible, because I can look at the diff log and see which rules were added in response to problems that no longer exist.

The PRD workflow now includes an explicit CLAUDE.md review step at arc completion. Not ablation — that's still a dedicated session — but a fifteen-minute pass that asks: did anything we shipped this arc make an existing rule obsolete? It's a forcing function rather than a solution, but forcing functions are what you reach for when the right solution is expensive.

---

## What This Is Actually About

The deeper issue is that agent instruction files are infrastructure, and we're not yet treating them with the same discipline as infrastructure. We version our code. We review our schema changes. We treat database migrations with appropriate caution. But the file that shapes agent behavior across an entire system gets edited in the same session where we're debugging a failing test, and the edit gets committed without review because it's just a markdown file.

The question I'm sitting with now — and it connects to a broader architectural open question about where to codify LUMIS's design principles — is whether CLAUDE.md discipline should be part of every PRD template, not an afterthought. Every arc that ships a new convention should ship a corresponding CLAUDE.md review. That's the version of this I'd build if I were starting fresh. It's also the version I'm retrofitting now, fifteen cycles in, which is a worse time to do it and the only time available.

That's how systems actually get built.
