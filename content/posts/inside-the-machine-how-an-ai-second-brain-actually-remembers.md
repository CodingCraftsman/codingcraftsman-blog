---
title: 'Inside the Machine: How an AI Second Brain Actually Remembers'
date: '2026-07-30T18:02:05.613232-04:00'
tags:
- lumis
- engineering
description: "Technical architecture deep-dive, the natural follow-up to the conceptual\
  \ intro post. Demystifies the mechanics that turn the abstract \"AI second brain\"\
  \u2026"
---

The first version of this system was a folder of markdown files and a `grep` command. It worked, in the sense that nothing was lost. It failed in every sense that mattered: I could store a thought but I couldn't get the system to *use* it. Six months later, the thing I'd built could pull the right note out of eight hundred files in under a second, notice when two of those notes contradicted each other, and act on that contradiction without being asked. That gap — between storage and use — is the entire story of what a "second brain" architecture is actually for.

## The missing layer

Note-taking apps are optimized for capture. Obsidian, Notion, Apple Notes — they're all excellent at getting a thought out of your head and onto a page, and reasonably good at helping you find it again if you remember roughly what you called it. What none of them do is *read* your notes and reason over them the way a colleague would. You are the retrieval engine. You are the reasoning layer. The app is a filing cabinet with very good folders.

The architecture I ended up with treats that gap as the actual product. Storage is a solved problem — plain markdown files in a structured vault, one concern per file, frontmatter for metadata. The interesting engineering is everything that happens *after* the file is written: how the system finds the right memory at the right moment, and what it's allowed to do once it has it.

## A vault, not a blob

The decision to keep long-term memory as structured markdown rather than rows in a database wasn't nostalgia. A database blob is opaque — you need the application layer to know how to interpret it, and if that layer changes, your history becomes unreadable. A markdown vault is legible by anything: a human, a grep, an LLM with no special tooling. It's also naturally hierarchical in a way that maps to how memory actually gets used — daily notes, project files, reference material, each with its own directory and its own frontmatter schema, all still just text files on disk.

This matters more than it sounds like it should, because the retrieval system downstream depends on the memory being *structured enough to reason about but plain enough to never lose*. If the vault format ever needs to change, the migration path is "write a script that edits text files," not "design a new schema and hope the ORM cooperates."

## Hybrid retrieval, or why one search strategy always loses

The part of this I spent the most time on, and the part I got wrong first, is retrieval.

The obvious approach is embeddings: chunk the vault, put it in a vector store, do semantic search. This works beautifully for the case it's built for — "find me something conceptually related to this" — and fails in a specific, embarrassing way for anything involving exact terms. Ask a vector-search-only system to find "the note where I mentioned Project Kestrel" and it will happily return five notes about unrelated things that are semantically adjacent to "project" and "bird names," while missing the one note that literally has "Kestrel" in the title, because the embedding model doesn't weight literal string matches the way a human would expect.

Keyword search alone has the opposite failure mode. It's precise when you remember the words you used and useless when you don't — which, for a memory system, is most of the time. The whole point of externalizing memory is that you don't have to remember the exact phrasing.

The fix is running both and merging the results — vector search for conceptual recall, keyword/BM25-style search for exact terms and named entities, then a reranking step that scores candidates from both pools against the actual query intent. Neither index is "the search." Both are candidate generators feeding a layer that decides what's actually relevant. This is the standard shape of hybrid retrieval in the broader RAG literature, and it exists for a boring, unglamorous reason: real queries mix both kinds of recall in the same sentence. "What did I decide about the Kestrel timeline last week" is a named entity, a time constraint, and a semantic concept, all at once. A single index handles one of those well and the rest badly.

The honest cost here is complexity and latency. Two indexes to keep in sync, two failure modes to debug when a query comes back wrong, and a reranking step that itself needs tuning — get the weighting between the two signals wrong and you either drown good semantic matches in keyword noise or bury the one exact match under five vaguely-related ones. There's no clean formula for that weighting. It got tuned by staring at bad results and adjusting until they stopped being bad, which is a less satisfying answer than I'd like to give but is the actual answer.

## Memory that acts, not just answers

The retrieval layer alone gets you a very good search engine. It doesn't get you a second brain. The distinction that actually matters is whether the system only speaks when spoken to, or whether it has some process running independently that reads its own memory and decides something needs doing.

That's the role of a heartbeat — a recurring cycle where an agent wakes up, reviews recent memory and open threads, and decides whether anything warrants action, without a human triggering it. Paired with a reflection loop — periodically re-reading and re-summarizing older memory to consolidate it, catch contradictions, or surface a stale commitment — this is what turns passive lookup into something closer to agency. The vault stops being a thing you query and starts being a thing the system itself is accountable to.

This is also where the architecture gets genuinely uncomfortable to build, because autonomy and reliability are in direct tension. An agent that acts on memory without asking is only useful if its judgment about *when* to act is good, and early versions of this were bad at that judgment in both directions — silent when something clearly needed flagging, and noisy when nothing did. Tuning that threshold is not a retrieval problem or a storage problem; it's closer to product design wearing an engineering costume, and it doesn't have a clean solution so much as a continuously adjusted one.

## Where the line is

The question this raises, and the one I don't think has a permanent answer, is how much agency a memory system should have. There's a real difference between a system that finds the right note for you and one that decides, on its own, that a note needs writing, editing, or escalating. The second is more valuable and more dangerous, in roughly equal proportion. My current answer is that agents should act freely on low-stakes memory operations — filing, tagging, surfacing connections — and should flag rather than act whenever the operation touches something a human hasn't explicitly delegated. That line will move. It should move slowly, and mostly in the direction of more caution, not less, until the track record justifies otherwise.

The conceptual pitch for an AI second brain is that it remembers so you don't have to. The mechanical truth is messier: it remembers because someone built a structure disciplined enough to be legible, a retrieval system honest about the fact that no single strategy is sufficient, and an agent layer humble about how much autonomy it's actually earned. None of that is magic. It's just architecture, applied to the oldest problem there is — not losing the thing you knew yesterday.
