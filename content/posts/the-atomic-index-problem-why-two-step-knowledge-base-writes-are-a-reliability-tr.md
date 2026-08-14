---
title: 'The Atomic INDEX Problem: Why Two-Step Knowledge-Base Writes Are a Reliability
  Trap'
date: '2026-08-13T22:18:00.364083-04:00'
tags:
- lumis
- engineering
description: "A concrete case study from building LUMIS: the pattern of writing a\
  \ research artifact and then separately updating an INDEX file is a well-known\u2026"
---

I've broken LUMIS's knowledge base the same way three times now. Each time, the failure looked different on the surface — a session that hit context exhaustion mid-task, a `/compact` event that truncated the agent's working memory, a clean session boundary where I just closed the terminal. Underneath, the failure was identical: a research artifact got written, the INDEX didn't get updated, and the gap was invisible until something downstream tried to find that artifact and couldn't.

This is not an LLM problem. It's not a prompt problem. It's a systems-engineering problem with a formal name and a structural fix.

---

## The Pattern and Why It Keeps Appearing

The two-step write-then-index pattern looks reasonable when you first sketch it. An agent completes a research task, writes a markdown file to `research/prism/`, then updates a central INDEX file with the new entry. Two operations, both succeeding in the happy path, everything consistent.

The problem is that "two operations" is already the failure mode. In filesystem and database literature, this is the classic non-atomic update problem: any time you need two writes to stay in sync, you've created a window in which they're out of sync. If anything interrupts between them — a crash, a power loss, a process kill — you're left with partial state. One operation succeeded, one didn't, and the system has no way to detect the inconsistency on its own.

The reason this keeps reappearing in LLM-assisted workflows specifically is that agents are particularly good at creating this window. A language model generating a long research artifact is doing expensive, latency-heavy work. By the time it finishes writing the artifact and turns its attention to the INDEX update, it's consumed a significant fraction of its context budget. Context exhaustion, `/compact` events, and session boundaries are all more likely after a substantial write than before one. The expensive operation — the one that creates the artifact — succeeds. The cheap operation — the one that maintains consistency — gets dropped.

And because the INDEX is a summary structure, not the primary data, the failure is silent. The artifact exists on disk. No error was raised. The system looks healthy. It's only when you ask "what research artifacts do I have on topic X?" that you discover the index is stale.

---

## How the Failures Actually Manifest

In practice, I've seen three distinct failure modes, all producing the same silent inconsistency.

**Session boundary failures** are the most common. An agent completes an artifact write near the end of a conversation. The user closes the terminal or starts a new session. The next session has no memory of the incomplete write, no obligation to finish it, and no signal that anything is wrong. The artifact accumulates in the filesystem; the INDEX stays stale indefinitely.

**Context exhaustion failures** happen mid-session but are equally invisible. A long research task fills the context window. When the agent is forced to truncate or summarize its working memory, pending housekeeping tasks — including INDEX updates — are candidates for dropping. The agent may not even register that it failed to complete the second step.

**/compact event failures** are the subtlest. Claude Code's `/compact` command compresses conversation history to free context. If a write is in progress or an INDEX update is queued when `/compact` fires, the compressed history may not preserve the pending obligation. The agent continues from the compacted state with no awareness of the dropped task.

All three produce the same artifact: a research file on disk with no INDEX entry. Grep can find it; the INDEX-aware query layer cannot. The inconsistency grows silently over time.

---

## The Fix Is Structural, Not Behavioral

The obvious first response is to make the agent more diligent. Add a CLAUDE.md rule: "Always update the INDEX after writing a research artifact." Add a reminder at the end of every session. Add a verification step that checks the INDEX after each write.

I tried this. It helps at the margins and doesn't solve the problem.

The reason behavioral fixes are insufficient is that they all operate at the prompt layer. They make the agent *more likely* to perform the INDEX update; they don't make it *impossible* to skip it. Every failure mode I described above — session boundary, context exhaustion, `/compact` — can interrupt execution between two well-intentioned prompt-layer steps just as easily as between two careless ones. Reliability at the architectural level requires structural enforcement, not behavioral convention.

The structural fix is to make artifact creation and INDEX mutation a single transactional unit at the tool layer. Instead of two tool calls — `write_file(artifact)` then `write_file(index_entry)` — there's one: `write_artifact_with_index(artifact, metadata)`. The tool implementation performs both writes and does not return success until both have completed. From the agent's perspective, there is no two-step operation. There is one operation that either succeeds completely or fails explicitly.

---

## Implementation Tradeoffs

Three approaches are worth considering, each with different tradeoffs.

**A dedicated atomic tool** is the cleanest. You write a custom MCP tool — `create_research_artifact` — that takes the artifact content and metadata as parameters, writes both the artifact file and the INDEX entry in a single function call, and exposes this as a single tool. The agent can't call them separately because they don't exist separately. The enforcement is absolute.

The tradeoff is that you've now pushed the INDEX update logic into your tool layer, where it has to stay synchronized with whatever your INDEX format actually is. Every time the INDEX schema changes, the tool has to change too.

**A wrapper script** is less elegant but faster to iterate on. The agent calls `write_file` normally, but a post-tool-use hook intercepts every write to the `research/prism/` directory and automatically updates the INDEX. This is the approach Claude Code's hook system is designed to support: a `PostToolUse` hook fires after each `write_file` call, checks whether the written path matches the research artifact pattern, and runs the INDEX update script if it does.

The advantage is that the agent doesn't need to know anything about this. Its behavior doesn't change. The consistency guarantee is enforced below the level it operates at. The disadvantage is that hook failures are harder to surface — if the INDEX update script itself fails, the agent may not see the error.

**A pre-commit hook** is appropriate if your knowledge base is version-controlled. A git pre-commit hook can reject commits that include new artifact files without a corresponding INDEX update. This catches failures at a commit boundary rather than in real time, which means stale state can persist within a session. But it provides a hard backstop: nothing gets permanently committed with an inconsistent INDEX.

For LUMIS, the wrapper approach using `PostToolUse` hooks is where I landed — it requires no change to agent behavior and enforces consistency at the boundary where the failure actually occurs.

---

## The General Pattern

The specific case — research artifacts and an INDEX file — is an instance of a broader problem that appears anywhere you have a primary artifact and a secondary index that must stay in sync across process or session boundaries.

Database secondary indexes. Search engine document stores and their keyword indexes. A codebase and its auto-generated API documentation. A message queue and its offset tracking. Everywhere the same structure appears: two things that must agree, maintained by two separate writes, with a window between them where they don't.

The systems engineering solution is always the same: collapse the two writes into one operation, enforce the invariant at the layer where the writes happen, and don't rely on the caller to maintain consistency through convention. For human programmers, this is the argument for transactions. For LLM agents operating across session boundaries with imperfect context, it's even more important — because the agent can't remember what it was doing before the session ended, and it can't be relied upon to check.

The knowledge base is only as useful as its index. If the index can go stale silently, the knowledge base is unreliable in proportion to how often partial writes occur. In a system that writes frequently and operates across many session boundaries, that proportion is not negligible.

The fix is boring: make it structurally impossible to write one without the other.
