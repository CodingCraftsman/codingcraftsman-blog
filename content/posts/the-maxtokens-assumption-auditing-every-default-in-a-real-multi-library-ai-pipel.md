---
title: 'The Max_Tokens Assumption: Auditing Every Default in a Real Multi-Library
  AI Pipeline'
date: '2026-08-28T23:18:13.048618-04:00'
tags:
- lumis
- engineering
description: "After discovering that pydantic-ai's default 4096-token cap had silently\
  \ truncated outputs across a live research pipeline, the author conducts a\u2026"
---

There's a particular kind of production bug that doesn't announce itself. It doesn't throw an exception. It doesn't trigger an alert. It just quietly returns less than you asked for, and the system accepts the answer, logs a success, and moves on.

We found one of those bugs retrospectively, buried in 442 scanned artifacts. Twenty-three-plus research outputs had been silently truncated — complete enough to look finished, incomplete enough to matter. The culprit was a single undocumented default: pydantic-ai's 4096-token cap on `max_tokens`, applied globally, never surfaced in a warning, never visible in the output format. The outputs weren't marked as truncated. They were just shorter than they should have been.

That discovery kicked off something I'd been meaning to do properly for a while: a full audit of every default in the inference stack.

---

## How Silent Truncation Works in a Layered System

The frustrating thing about this class of bug is that it's structurally predictable — and structurally invisible. Every library in an agentic pipeline introduces its own assumptions about output length. They don't coordinate with each other. They don't coordinate with the model's actual context window. They just each apply their own cap, independently, and whichever is smallest wins.

In practice, this means you have at minimum three places a response can be silently shortened:

**Library defaults.** The framework you're using to structure inference calls — in our case, pydantic-ai — may set a `max_tokens` default that's conservative by modern standards. 4096 tokens was a reasonable limit when GPT-3 was the reference model. It's a problematic default today, when models routinely support 32K, 128K, or more. But the default persists, and unless you explicitly override it at the call site, it silently governs every request.

**Model-API defaults.** The model provider itself may have default behavior when `max_tokens` is not specified — or when it is specified but set below the model's actual capability. Different providers handle this differently. Some return whatever they generate up to the limit. Some use the limit as a hard stop mid-sentence. None of them, in my experience, consistently surface this in a way that's easy to instrument without explicitly requesting finish-reason metadata.

**Proxy and orchestration layer caps.** If your pipeline routes through any intermediate layer — an internal API gateway, a caching proxy, a rate-limiting wrapper — that layer may impose its own response-size constraints. These are often the least documented and the hardest to discover, because they live outside the model client's visibility entirely.

These three don't add — they multiply. The effective output limit for any given call is the minimum across all three. And if you haven't explicitly set limits at each layer, you have no idea what that minimum actually is.

---

## Running the Audit

Once we confirmed the pydantic-ai truncation was real, I went through the pipeline layer by layer with a simple question for each component: what is the effective `max_tokens` limit this component applies, and is that limit documented anywhere visible to the call site?

For pydantic-ai, the answer was 4096 — set internally, not surfaced in standard configuration, not overridable through the model parameters we'd been passing. For the underlying API client, the answer varied by endpoint and by whether we'd been explicit. For the orchestration logic that chains research jobs, there was no explicit limit at all — which meant it inherited whatever the library below it assumed.

The audit produced something I hadn't expected: a taxonomy of ignorance. We didn't just have unknown limits — we had limits we'd been confidently wrong about. In a couple of places, I'd assumed we were getting full model output because we hadn't set a cap, not realizing that the absence of an explicit setting meant a library default was silently active.

The fix at the call site is straightforward: set `max_tokens` explicitly on every inference call, sized to what the model actually supports and what the use case actually needs. For a research synthesis job that might produce several thousand words of structured output, 4096 tokens is not enough. Setting it explicitly to something appropriate — and documenting why — takes about thirty seconds and eliminates an entire class of silent failure.

The harder lesson is that explicit settings at the call site are necessary but not sufficient.

---

## The Write-Time Assertion Guard

The more durable defense is a guard at the storage layer. The logic is simple: before any inference output gets written to the vault, assert that its length is within expected bounds. If the output is suspiciously short relative to what the task should produce — below a minimum token threshold, say, or below a percentage of the model's stated limit — the write fails with a diagnostic error rather than silently persisting a corrupted artifact.

This pattern catches truncation regardless of which layer introduced it. It doesn't matter if the shortfall came from a library default, an API limit, or a proxy cap. If the output is too short, it doesn't get written as complete.

The tradeoff is that you need to establish reasonable bounds per task type — a one-sentence completion and a full research synthesis have very different expected output lengths, and a single global threshold won't serve both. In practice this means tagging inference calls with their output class and maintaining a small lookup table of minimum acceptable lengths per class. It's a modest amount of overhead and it's paid back immediately the first time it catches something.

---

## The Retrospective Cost

Here's what made this incident more than an engineering inconvenience: twenty-three-plus truncated outputs weren't sitting inert in the vault waiting to be corrected. They had already been used.

Research notes referenced them. Beacon drafts had been structured around their conclusions. Downstream decisions — about what to investigate further, about what was already understood, about what could be treated as settled — had been made on the basis of outputs that were missing their endings.

Reprocessing the artifacts is straightforward. Tracing their influence on everything that read them is not. Some of that influence is recoverable through explicit dependency tracking. Some of it has already propagated into reasoning that's hard to unwind. The cost of a silent truncation bug isn't just the corrupted artifact — it's every downstream artifact that treated the corrupted output as ground truth.

This is why detection latency matters so much. The pydantic-ai default had been active for long enough that the affected outputs had already been incorporated into the broader knowledge base before we found the problem. A write-time guard would have surfaced this at the moment of failure rather than weeks later.

---

## What the Instrumentation Should Look Like

The instrumentation fix is the one I wish I'd shipped earlier. Every inference call site should log two things explicitly: `finish_reason` and token counts. Most model APIs return these in the response metadata. Most pipeline code ignores them.

`finish_reason` is the key signal. A response that terminated because the model finished naturally returns a different reason code than one that hit a length limit — typically something like `stop` versus `length`. If you're not logging finish reason, you have no way to detect in real time that a response was cut off. You find out weeks later, if you find out at all.

Token count logging gives you the secondary signal: if a response is consistently returning at exactly the limit you've set — or exactly the library default you didn't know was active — that pattern is detectable before you audit 442 artifacts looking for it.

Neither of these is expensive to instrument. Both should have been there from the beginning.

---

## The Broader Point

The max_tokens incident is a specific instance of a general problem: in a multi-library agentic system, defaults are a form of implicit contract that no one has signed. Each library ships with assumptions that made sense at some point, and those assumptions persist until something breaks visibly enough that someone goes looking.

The defense isn't to read every library's source code before using it, though that helps. The defense is to treat every default as unknown until explicitly verified, set output-length contracts explicitly at every call site, and add a storage-layer guard that catches what the call sites miss.

That's where we are now. The audit is done. The call sites are explicit. The write-time guard is running. And the next truncation — when it happens, because it will — will fail loudly at the moment of failure rather than quietly at the moment of consequence.
