---
title: "About"
---

I've been writing software for more than 25 years. In that time I've worked across more languages, frameworks, and domains than I can easily count — from C and C++ to Java, Python, .NET, and everything in between. DevOps, CI/CD, cloud infrastructure — if it was part of the stack, I've probably had my hands in it at some point.

But here's the thing: almost all of it was built for someone else.

This is the first project I've built entirely for myself — and it's the largest personal project I've ever taken on. LUMIS started as a simple idea: what if I gave an AI assistant a real job and saw what happened? It's grown into something I genuinely look forward to working on every day.

Why "LUMIS," you ask? I'm a big Iron Man fan, and I wanted something with a bit of JARVIS's personality — not just a tool, but something that feels present. After kicking around a few other names (LYRA and AIDEN both still have a soft spot in my heart — maybe you'll meet them someday), I landed on LUMIS.

> **LUMIS: Learned Understanding and Memory with Intuitive Support**
>
> The name comes from *lumen*, the root word for light — plus a small wink at "Lumos" from Harry Potter. Light does two things: it shows you what's already there, and it helps you find your way when you can't see the path yet. That's the job I built this to do.

I'm also someone who can't leave well enough alone. I like to pull things apart, understand how they work at the seams, and put them back together better than I found them. That instinct drives everything you'll read here.

AI is moving fast. Faster than most of us expected. I decided I'd rather be someone building with it than someone watching from the sidelines wondering what they missed. This blog is my way of building in public — sharing what works, what doesn't, and what I'm still figuring out.

If you're somewhere on that same journey, you're in the right place.

## How LUMIS Is Built

A lot of what I write here mentions the tools I used to build LUMIS. If some of those names don't mean anything to you, that's fine — here's the short, plain-language version of what's under the hood.

- **Python** does most of the heavy lifting. It's the language behind the "brain" that thinks and makes decisions, the background jobs that run on a schedule, the connections to my email and calendar, and the engine behind the web dashboard I use to keep an eye on everything.
- **SvelteKit** is what the dashboard *looks* like — the buttons, pages, and screens I actually click on. It's a modern toolkit for building web pages that feel fast and responsive.
- **SQLite** is a tiny, self-contained database that keeps track of the system's state — what's been done, what's pending, what to remember. And LUMIS's actual memory lives in plain text files (Markdown) that I can open and read in any editor, including a note-taking app called Obsidian. Nothing is locked away in some cloud service I can't reach.
- **Claude** is the AI at the center of it all. LUMIS talks to Claude — Anthropic's AI model — through a couple of official toolkits (the Claude Agent SDK, plus lifecycle "hooks" in Claude Code and OpenCode) that let the AI read notes, draft replies, and take actions in a controlled way. A small AI model running on my own machine handles the simpler, high-volume jobs to keep costs down, and Claude handles the real thinking.
- **A handful of shell and Python scripts** fill in the gaps around the AI — small bits of automation that glue the pieces together and handle the plumbing.

And this blog itself? It runs on **Hugo**, a tool that turns plain text files into a finished website, styled with a theme called **PaperMod**. Every time I publish, **GitHub Actions** (an automation service) rebuilds the site and posts it to **GitHub Pages** (free web hosting) — so writing a post is really just saving a text file and letting the robots do the rest.

None of this is magic. It's a stack of ordinary, mostly free tools, wired together with a lot of curiosity. If you're building something of your own, I hope seeing the seams makes it feel a little more within reach.
