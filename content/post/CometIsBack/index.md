---
title: Comet Engine Is Back — And It Grew Up
description: Fourteen months of silence, nine releases, and a promise about every Wednesday from now on
date: 2026-07-29 18:00:00+0000
image: cover.png
categories:
    - Devlog
tags:
    - Editor
    - Roadmap
    - AngelScript
    - Vulkan
---

The last time I posted here was May 2025. I had just decided to throw away the C# scripting system, I was three weeks into rewriting 300 script files in AngelScript, and I ended that post with a sentence I have thought about many times since:

> So I expect that just before the ending of 2025 I will be able to release a CometEngine_v2.0_Beta version.

I missed that by about four months. 2.0 shipped in April 2026.

But here is the thing nobody warned me about: once 2.0 was out, everything got faster. Nine releases landed between April and July. The blog stayed quiet the whole time because every evening I had went into the engine instead of into writing about the engine. That was probably the right call at the time. It is definitely the wrong call now, because there is a year of work in here that nobody has seen.

So this post is the catch-up. And at the end of it, a promise.

![Comet Engine today](editor-today.png)

## What actually happened

Let me put the numbers down first, because they surprised me when I collected them.

Since the last post: **nine releases**, 2.0 through 2.8. The engine went from "a 2D editor with C# scripting that runs on Windows and Linux" to something I have a hard time describing in one sentence. Which is itself a problem, and part of why this blog is starting again.

![Release timeline](releases.png)

The single biggest change is the one I was writing about last time. **C# is gone.** Completely. AngelScript is the only scripting language now, and 2.0 was a hard break — projects written for 1.x do not open. That was a genuinely uncomfortable decision and I will spend the next two Wednesdays on it, because "I rewrote 300 files" is not the interesting part. The interesting part is what I got in exchange, and what it cost.

The short version of the exchange: **the engine now runs in a browser and on a phone.** That was the entire reason for the migration, and it worked.

## The parts I did not see coming

If the story had ended at "AngelScript, Web, Android," this would be a shorter post. It did not.

![What landed](highlights.png)

Somewhere around 2.2 the pace changed. Things I had written down as "maybe in a few years" started landing in single releases. A **Shader Graph** with about 115 nodes, so you can build a dissolve effect without writing a line of HLSL. **Visual Scripting**, where the node graph compiles directly into AngelScript bytecode — not interpreted, not a slower path, the same bytecode a hand-written script produces. **2D dynamic shadows.** **A Vulkan backend**, loaded at runtime so machines without a Vulkan driver still boot on OpenGL.

And the one I am still slightly amazed by: the particle system now simulates **a hundred thousand particles in 2.7 milliseconds** on the CPU, or moves the whole thing onto the GPU in compute shaders if the backend supports it. The old particle system was one object per particle with a virtual call per property per particle. That rewrite is a whole post on its own, probably two.

There is also a full networking stack now, with ENet, WebSocket and WebRTC transports and high-level scene replication on top. There is a package manager with a marketplace. There is a profiler with flame graphs that can attach to a running build over TCP. There is an MCP server inside the editor so an AI assistant can drive it.

I am listing these like a changelog, which is exactly the thing I do not want this blog to be. So let me stop.

![The game view](game-view.png)

## Why the blog is starting again

Here is what I have learned from fourteen months of not writing.

Comet has become genuinely capable and almost entirely invisible. When someone asks me what it does, I either give them a two-word answer that undersells it, or I talk for twenty minutes and lose them. There is no version in between, and that is a writing problem, not an engine problem.

There is also a selfish reason. Every single time I have written about a system in this blog, I have found a bug in it. Not sometimes — every time. Explaining how something works to a stranger is the most reliable code review I have ever found. Fourteen months of not writing is fourteen months of not doing that review.

## The promise

**A post every Wednesday, for a year.**

Not a changelog. Not a tutorial. I want to explain what Comet actually is, one system at a time, in a way that is enjoyable to read whether or not you ever intend to use it. Lots of pictures. Real screenshots from the real editor, not marketing renders.

Some of them will be in parts, because "2D lighting" is not one post and pretending it is would make it a bad one. When a post is part of a series I will say so at the bottom and tell you what is coming the following Wednesday.

Roughly, the year looks like this: the scripting migration and the editor first, then the object model, then a long run through rendering — lighting, shadows, shaders, the Shader Graph, post-processing. Then particles and performance. Then scripting tools, the built-in code editor, visual scripting. Then tilemaps, animation, physics, UI, audio, navigation. Then the content pipeline and shipping to web and mobile. Then multiplayer. Then, at the end of it, a proper look back at what is now seven years of this project.

Next Wednesday: the migration. What a Comet script looked like in C#, what the mono bridge was actually doing, which parts of the port were mechanical, and the three things AngelScript flatly does not have that I had to design around.

*See you Wednesday. Feel free to share comments or questions ;)*
