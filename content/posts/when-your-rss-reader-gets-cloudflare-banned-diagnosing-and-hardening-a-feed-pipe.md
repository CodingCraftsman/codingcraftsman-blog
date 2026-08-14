---
title: 'When Your RSS Reader Gets Cloudflare-Banned: Diagnosing and Hardening a Feed
  Pipeline Against Bot Detection'
date: '2026-08-13T22:25:41.901497-04:00'
tags:
- lumis
- engineering
description: "A concrete incident report: LUMIS's feed poller accumulated 5+ consecutive\
  \ failures against Julia Evans' blog before the root cause \u2014 Cloudflare silently\u2026"
---

The failure looked like nothing. No HTTP 500, no timeout exception, no DNS resolution error — just empty content coming back from a feed that had been working fine. Julia Evans' blog had been a reliable source in LUMIS's feed pipeline, and then one day it wasn't, and the logs gave me almost nothing to work with.

That's the thing about Cloudflare blocking your bot: it doesn't always slam a door in your face. Sometimes it just stops letting anything through and waits to see if you notice.

---

## How It Presented

The poller had accumulated five consecutive failures against `jvns.ca` before I flagged it during a weekly reflection. The RSS queue had grown to 69 pending entries — not catastrophic, but a signal that something upstream was quietly degrading. The failures weren't loud. There was no exception stack in the logs, no HTTP error code to grep for. The fetch was completing; it was just returning empty content where a feed document should have been.

This is a particularly nasty failure mode because it defeats the obvious monitoring approach. If you're alerting on HTTP errors or network exceptions, a Cloudflare soft-block walks right past your instrumentation. The request succeeds in the transport sense. The response body is just... not what you asked for.

By the time I looked closely, the feed had been silently broken for at least two days — `2026-07-27` is when I first flagged it; `2026-07-29` is when I had a root cause.

---

## The Diagnostic Path

I started where you'd expect: ruling out the obvious candidates.

**DNS and SSL** were fine. The domain resolved correctly, TLS negotiation completed without complaint. Fetching the feed URL manually from a browser worked perfectly — Evans' Atom feed loaded immediately, full document.

That browser-versus-poller asymmetry is the tell. When something works in a browser and fails programmatically, and there's no obvious network or format issue, you start looking at request-level differences: headers, User-Agent strings, rate limiting, IP reputation.

I pulled the actual HTTP request the poller was making. The User-Agent string was the problem: `feed_poller/1.0` or something functionally equivalent — a string that announces, unambiguously, "I am automated software." Cloudflare's bot detection doesn't need to be sophisticated to catch that. It just needs to check whether your UA looks like a browser, and when it doesn't, it can choose to return a response that appears to succeed at the HTTP level while delivering an empty or stripped body instead of the actual content.

The fix was straightforward: change the User-Agent in `feed_poller.py` to something browser-shaped. Not spoofed in any aggressive sense — just a realistic browser UA rather than a self-identifying bot string. One global change, not a per-feed workaround.

The reason I didn't want a per-feed override deserves some explanation. A per-feed fix for `jvns.ca` would have solved this specific incident. But Evans' site is not unusual — she uses a CDN, as most sites do. The same Cloudflare bot detection that blocked LUMIS's poller is running on thousands of other domains in the feed list. A per-feed approach means I'm playing whack-a-mole: the next blocked feed requires another override, and another. A global browser-shaped UA addresses the underlying issue: the poller was presenting itself as a bot to every CDN it touched.

---

## The Bonus Bug

Here's where the incident got more interesting.

While I had `normalize_entry()` open to investigate the empty-content behavior, I did a close re-read of how the parser handled Atom feeds. That's when I found it: the timestamp normalization code was looking for `<published>` tags and falling back to nothing when that tag was absent.

Atom feeds use `<updated>`, not `<published>`, as the primary date element. RSS 2.0 uses `<pubDate>`. They're semantically similar but structurally different, and if you're parsing both formats with shared normalization logic, you have to handle both tag names. The code wasn't. For any Atom feed that didn't also include an optional `<published>` element — which is most of them — the timestamp was being silently dropped. Entries were ingesting with null timestamps.

I wouldn't have caught this if the Cloudflare investigation hadn't forced me to read the parser carefully. The null timestamps weren't causing visible failures — entries were still processing, just with missing date metadata. That's the kind of bug that sits in production indefinitely because nothing breaks loudly enough to trigger investigation.

The fix was adding the `<updated>` fallback into `normalize_entry()`. Two lines. But finding it required a failing feed and an hour of careful log-reading that I wouldn't have done otherwise.

---

## What Feed Reliability Actually Requires

This incident clarified something I'd been vague about: "feed reliability" has at least three distinct layers, and I'd only been instrumenting one of them.

**Transport reliability** is what I was monitoring — did the HTTP request complete? But as this incident demonstrated, transport success doesn't mean content success. You need to be checking that the response body is actually a parseable feed document, that it has entries, that those entries have the fields you expect. A Cloudflare-intercepted response can be transport-successful and content-empty.

**Content integrity** is harder to monitor but more meaningful. Content-hash deduplication — storing a hash of the fetched feed document and comparing it on subsequent fetches — would have caught the Cloudflare block within one polling cycle. If the hash changes from a valid feed document to an empty or stub response, that's a signal worth alerting on. If the hash *stops changing* across multiple cycles for a feed that should be updating, that's a different signal: the fetch is succeeding but returning stale or cached content.

**Per-feed failure budgets** are the operational layer. The poller needs to track consecutive failures per feed and escalate — not just log — when a feed crosses a threshold. Five consecutive empty fetches against a known-active feed should produce a visible alert, not just increment a counter that gets reviewed whenever I happen to look at the queue depth.

There's also a case for a feed-health heartbeat that's architecturally separate from the poller itself. The poller can't easily observe its own failure modes — if it's blocked at the UA level, it doesn't know it's being blocked. An independent health-check process that periodically validates a sample of feeds using different infrastructure (different IP, different UA) would catch classes of failures that the poller is constitutionally unable to self-diagnose.

None of this is exotic. It's the standard reliability engineering pattern applied to a domain — feed polling — that tends to get built quickly and then treated as solved infrastructure.

---

## What I Took Away

The immediate fixes were the UA change and the `<updated>` fallback. Both shipped on `2026-07-29`. The jvns.ca feed started returning content immediately once the UA was updated, which confirmed the diagnosis.

The longer-term lesson is about the gap between "the fetch completed" and "the pipeline is working." Those are different things, and I'd conflated them. A feed poller that can be silently blocked by CDN bot detection, that drops timestamps from Atom feeds without complaint, and that has no per-feed failure escalation isn't a reliable system — it's a system that works until it doesn't, and then fails quietly.

The RSS queue growing to 69 entries was the signal that something was wrong. That's too indirect. The infrastructure should have told me two days earlier that it had a broken feed, and it should have told me what was broken. That's the gap I'm closing.

---

*LUMIS is a personal knowledge-infrastructure system I'm building to handle content ingestion, synthesis, and retrieval. Earlier posts in this series cover the article-selection pipeline and the vector-store architecture. The feed poller is one component in a larger ingestion layer.*
