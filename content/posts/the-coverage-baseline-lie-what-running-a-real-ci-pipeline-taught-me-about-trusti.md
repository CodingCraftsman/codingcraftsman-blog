---
title: 'The Coverage Baseline Lie: What Running a Real CI Pipeline Taught Me About
  Trusting Your Own Numbers'
date: '2026-08-26T20:25:46.026888-04:00'
tags:
- lumis
- engineering
description: "A concrete case study in how a stale coverage artifact (coverage.xml)\
  \ reported 52.75% \u2014 causing a ticket to be filed and prioritization decisions\
  \ to be\u2026"
---

For several sessions, LUMIS's CI pipeline reported 52.75% test coverage. That number sat in a ticket, shaped prioritization decisions, and drove the creation of a task to close what looked like a meaningful gap. The actual full-suite coverage, when we finally ran it correctly, was 81.93%. A 29-point phantom gap — real enough to generate engineering work, invisible enough that nothing in our normal review process flagged it.

This is a write-up of how that happened, how we found it, and what we changed so the same class of failure doesn't quietly corrupt future decisions.

---

## How a Stale Artifact Becomes Ground Truth

The mechanism is almost embarrassingly simple in retrospect. At some point during active development, a `coverage.xml` artifact was generated from a partial test run — not the full suite, just whatever subset was being exercised at that moment. That artifact was committed to the repository. From that point forward, anything reading coverage numbers from the stored file was reading a lie.

The file reported 52.75%. Nothing about that number looked obviously wrong. It was plausible — we were in active development, modules were being added, test coverage was legitimately uneven. A number in the low fifties didn't trigger skepticism. It triggered a ticket.

What made this particularly invisible is the structure of how coverage artifacts get consumed. The CI pipeline had a gate: pass if coverage exceeds threshold. The threshold was 50%. The stored artifact reported 52.75%. Gate passed. No alert. No indication that the number being checked was stale or partial. From the pipeline's perspective, everything was green.

The partial-run number — what you'd get if you ran only the modules that were being touched in a given session — was actually 38.53%. So we had three numbers in the system simultaneously: 38.53% from fresh partial runs, 52.75% from the trusted stored artifact, and 81.93% as the true full-suite figure. None of them matched. None of them produced an obvious contradiction signal, because they were never being compared against each other.

---

## The Three-Number Problem

This is worth dwelling on, because the three-number situation isn't just an implementation accident — it's a structural property of any system that mixes stored artifacts with live runs.

The stored artifact (52.75%) was the authoritative number by convention. It lived in the repository, it had a filename that implied completeness (`coverage.xml`, not `coverage-partial.xml`), and it was what the gate read. The partial-run number (38.53%) was what you'd see if you ran coverage manually during a session focused on a specific module cluster. The full-suite number (81.93%) was what you'd get if you ran the entire test suite from scratch with a clean coverage database — which we apparently hadn't done in a while, or hadn't committed the resulting artifact.

Three numbers, three different measurement conditions, zero reconciliation mechanism. The pipeline gate was checking the stored artifact against a threshold, not checking whether the stored artifact was fresh. The manual runs during development weren't updating the committed artifact. And nobody was running the full suite and committing those results on a regular cadence.

The result was a priority queue that contained task #164 — a task to close the fictional 29-point coverage gap — sitting there consuming planning attention for work that was largely already done.

---

## Discovery and What It Actually Took

The discrepancy surfaced through a routine reflection pass, not through any automated alarm. Looking at the coverage numbers during a weekly review, something felt off about the gap between what we were seeing in active development and what the stored artifact claimed. That prompted running the full suite explicitly, which produced 81.93%, which immediately invalidated the stored number and the task built on top of it.

This is the uncomfortable part: the detection mechanism was human pattern recognition during a manual review, not anything the pipeline itself provided. A 29-point discrepancy between ground truth and trusted artifact, and the system's response was silence. No staleness check. No "this artifact is N days old." No comparison between fresh run and stored artifact. Just a gate that passed because 52.75 > 50.

The fix was straightforward once the problem was clearly framed. We raised the Makefile floor from 50% to 80%, which better reflects actual system state and makes future partial-run artifacts fail the gate rather than slip through. More importantly, we switched to a pipeline configuration that generates fresh coverage numbers on every CI run rather than reading from a stored artifact. The gate now runs the full suite and checks the output inline — the artifact, if generated at all, is a reporting artifact, not the thing being checked against the threshold.

Task #164 was closed as moot. Task #169 was filed as the correct follow-up: maintain coverage above 80% as new modules land, with the gate now actually enforcing that against real numbers.

---

## The Broader Pattern

Coverage is just the instance I have the clearest receipts for. The failure mode generalizes.

Any stored metric artifact in a CI pipeline is a silent drift risk. Benchmark scores committed after an optimization pass will keep reporting that performance even as subsequent changes erode it. Lint counts frozen in a baseline file will pass your gate while new violations accumulate in new files. Dependency audit outputs cached from the last time someone ran them will tell you your dependencies are clean while new CVEs are disclosed against packages you're actually shipping.

The common thread: you took a measurement at time T, stored the result, and now your pipeline is checking the stored result rather than taking a new measurement. The measurement is of the system as it existed at time T. The system has moved. The stored result hasn't.

The fix in each case is the same structural move: make the pipeline generate the number freshly on each run, check the fresh number against the threshold, and treat any stored artifact as a reporting output rather than an authoritative input. If fresh generation is too slow to run on every commit, that's a real constraint — but the response to that constraint should be explicit (run fresh on merge to main, run cached on PR builds, be explicit about which is which) rather than silent (run cached everywhere and pretend it's fresh).

---

## What This Changed in How I Think About Pipeline Outputs

Running through this left me more skeptical of any CI green that depends on reading a file rather than running a command. The distinction matters. "Run tests and check coverage" and "read coverage.xml and check the number" look the same in a passing pipeline. They are not the same.

The LUMIS pipeline now enforces the distinction structurally: the coverage gate runs the suite. The artifact is a side effect. If the run fails, the gate fails — not because a stored file has gone stale, but because the system under test doesn't meet the threshold right now, on this commit, with this code.

That's a harder standard to pass. It's also the only version of the standard that actually tells you something true.

The 52.75% number is gone from the codebase. The task it spawned is closed. The 81.93% baseline is now enforced by a gate that would catch its own drift. Small change to the Makefile. Large change to what "green" means.
