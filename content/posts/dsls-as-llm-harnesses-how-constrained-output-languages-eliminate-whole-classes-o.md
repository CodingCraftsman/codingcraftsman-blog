---
title: 'DSLs as LLM Harnesses: How Constrained Output Languages Eliminate Whole Classes
  of Agentic Bugs'
date: '2026-08-07T21:32:43.796470-04:00'
tags:
- lumis
- engineering
description: "Drawing on both Prism research and documented patterns from LUMIS's\
  \ own agentic sessions, this post argues that general-purpose languages are structurally\u2026"
---

Last quarter I watched a Claude Code session spend forty minutes debugging a Python script it had generated itself. The script was supposed to walk a directory tree and emit structured metadata for each file it found. The logic was correct in outline — the right algorithm, reasonable variable names, sensible structure. What it kept getting wrong was subtler: a loop counter that was initialized inside the function body but read outside it after early returns, a `zip()` that silently truncated when the two lists weren't the same length, a state dictionary that accumulated across calls because it was defined at module scope. Each bug was individually fixable. Together, they represented something more systematic: the failure modes that appear when you ask an LLM to generate multi-step procedural code in a general-purpose language.

The session eventually produced working code. But I kept thinking about the debugging loop — the way each fix revealed another problem, the way the errors were structurally invisible until runtime. That's not a capability problem. The model understood what it was trying to do. It's an architectural problem, and it points toward a different design strategy.

---

## Why General-Purpose Languages Are Structurally Bad Targets

There's a common framing of LLM code generation failures: the model isn't smart enough yet, more training will fix it, we're in an early period. That framing is wrong, or at least incomplete. The specific errors that plague LLM-generated multi-step code aren't random. They cluster around a handful of categories that the Prism research notes document fairly precisely.

**Variable scoping errors** are the most common. LLMs generate code token by token, and a variable introduced in one context gets referenced later as though that context is still active. Python's scoping rules are permissive enough that these errors don't surface at parse time — they wait for the specific execution path that exercises them.

**Off-by-one errors in iteration** are endemic. The model produces code that works for the representative example it's reasoning about, but the boundary conditions are wrong. This is particularly bad in agentic contexts where the loops often operate over dynamically-sized inputs.

**State management bugs** are the hardest to catch. A function that modifies shared state, a cache that doesn't get invalidated, a counter that doesn't reset — these bugs don't just fail, they produce outputs that look correct while quietly accumulating error. In an agent loop that's generating dozens of artifacts, this is especially painful.

What these three categories share is that they're all invisible until runtime, and in agentic systems, "runtime" often means "after the agent has made several downstream decisions based on the buggy output." The bug isn't just in the code — it's in everything the code touched.

The structural cause is straightforward: general-purpose languages have enormous valid output spaces. Python can do almost anything, which means the surface area for incorrectness is correspondingly vast. When you ask an LLM to generate arbitrary Python, you're asking it to navigate that entire space correctly, and the failure modes listed above are exactly what emerges when it doesn't.

---

## The DSL Strategy: Constrain the Output Space at Design Time

The alternative isn't to write better prompts or add more validation. It's to make the output space smaller by design.

A narrow DSL — or even just a strict schema — works by eliminating the degrees of freedom that generate the bugs. If your output language doesn't have mutable module-level state, state management bugs become syntactically impossible. If your iteration construct is declarative ("apply this operation to each item in this list") rather than imperative ("initialize i, check i < len(list), increment i"), off-by-one errors in loop mechanics go away. If your scoping rules are explicit in the schema, variable capture bugs can't be silently introduced.

The principle here is the same one Matt Williams describes in his "schema-first design" approach: define the output shape first, make it strict, and treat non-conforming outputs as explicitly unknown rather than silently wrong. The model either produces something that validates against the schema or it fails loudly. That loud failure is valuable — it's catchable, recoverable, and doesn't propagate downstream.

This isn't a new idea in programming language theory. Restricted computational models — regular expressions, context-free grammars, configuration languages with no Turing completeness — have always traded expressiveness for analyzability. The insight is that LLM code generation reactivates this tradeoff in a new context.

---

## What This Looks Like in LUMIS

Three output formats in the LUMIS stack function as narrow DSLs in exactly this sense.

**YAML pipeline definitions for systemd units.** New pipeline scaffolding in the system follows a fixed sequence: discover CLI registration patterns, write the pipeline module, author `.timer` and `.service` unit files, register them in setup. That sequence is invariant — every pipeline follows it. The YAML schema that drives generation captures that invariance explicitly. The model doesn't generate Python that decides what to do next; it fills slots in a structure that already knows what the sequence is. The systemd unit files themselves are near-trivially verifiable: `systemd-analyze verify` catches structural problems immediately.

**Skill metadata format for Claude Code.** Skill descriptions in the Claude Code integration follow a schema with explicit fields for capability, preconditions, and output contract. The model generates into those fields rather than producing free-form descriptions. This matters because the metadata is machine-read downstream — a vague or structurally malformed skill description causes silent failures at routing time. The schema doesn't make the descriptions better in a prose quality sense; it makes them *checkable*.

**PRD scaffolding templates.** Product requirement documents follow a template structure that functions as a fill-in-the-blank DSL. The template defines section types, required fields, and relationship markers between requirements. The model's job is to populate those structures, not to invent document architecture. The result is PRDs that can be mechanically checked for completeness — every required section present, every requirement linked to at least one acceptance criterion — rather than requiring human review of document structure.

In each case, the LLM is doing genuine intellectual work: figuring out what goes in the slots, reasoning about dependencies, writing content that's specific and correct. What it's *not* doing is deciding the output structure on the fly, which is where the class of bugs I described earlier enters.

---

## The Honest Tradeoff

DSLs require upfront design investment, and that investment has a real cost. Before you can generate YAML pipeline definitions, someone has to figure out what the schema is — which means having already built enough pipelines to know what the invariant structure looks like. The schema isn't discoverable from first principles; it emerges from experience with the domain.

This means DSLs are the wrong tool at the start of a project. Exploration phases, one-off scripts, tasks where the output shape is genuinely unknown — these are exactly the contexts where the flexibility of general-purpose code generation earns its cost. The debugging loop I described at the start is annoying, but it's the right way to figure out what you're actually building. Once you know the shape of what you're building, you can constrain it.

There's also a second failure mode worth naming: over-constraining. A schema that's too rigid forces the model to produce technically valid but semantically empty outputs — it fills the required fields with plausible text that doesn't actually carry information. The slot structure needs to match genuine structure in the problem domain; if it doesn't, you've just moved the badness from "structurally wrong" to "vacuously correct."

---

## The Broader Pattern

What the DSL strategy points toward is verification-driven design: not "make the model generate correct things" but "design output spaces where incorrectness is detectable." This is a different design philosophy than prompt engineering, and it works at a different level of the stack. Prompts tell the model what to do. Output schemas constrain what the model can produce.

In an agent loop that's running continuously and making dozens of generative decisions per session, the ability to verify outputs mechanically — against a schema, against a linter, against `systemd-analyze verify` — is what makes the loop trustworthy at scale. The alternative is a testing burden that grows linearly with the system's capability. That's not sustainable architecture.

The insight generalizes beyond code generation specifically. Any place where an LLM produces output that another system consumes is a candidate for schema-first design. The question to ask isn't "is the model capable of producing good output here?" It almost certainly is. The question is: "if it produces bad output, will I know immediately, or will I find out three steps later when something unrelated breaks?"

The answer to that question is an architectural choice, and narrow output grammars are one of the better tools available for getting it right.
