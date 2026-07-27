---
title: Shipping to the Web
description: Emscripten, ASYNCIFY, and an honest list of what a browser does not get
date: 2027-06-02 18:00:00+0000
image: cover.png
categories:
    - Platforms
tags:
    - Web
    - Emscripten
    - AngelScript
---

This is the platform that caused everything else in this blog's first year.

I could not export to the web with C# and Mono. That is why [2.0 removed C#]({{< ref "/post/GoodbyeCSharp1" >}}), which is why every project broke, which is why the scripting layer, the shader pipeline and the content system all look the way they do now.

![The platform list](platforms.png)

Web sits in that list alongside Windows, Linux and Android for one reason: the scripting layer compiles *into* the engine instead of shipping a runtime beside it.

## What a web build is

Comet compiles through **Emscripten** to WebAssembly. You get a `.wasm` module, a JavaScript loader, and your content packs. Serve them over HTTP and it runs.

**ASYNCIFY** is the piece that makes it possible at all. A native game loop is a `while` loop that blocks. A browser cannot have that — block the main thread and the tab freezes. ASYNCIFY transforms the code so it can unwind and resume around the browser's event loop, which is what lets engine code that was written as a blocking loop keep working.

The cost is real: ASYNCIFY adds size and overhead. It is the price of not rewriting the entire engine around callbacks.

## What the web does not get

![Missing](missing.png)

I want this list to be complete rather than flattering.

**No threads.** This is the big one and it colours everything. `THREADS_ENABLED` is not defined, so there is no [render thread]({{< ref "/post/TwoBackends" >}}) — rendering happens inline. [Particles simulate single-threaded]({{< ref "/post/Particles1" >}}). Async asset loading has no worker to load on.

**No Vulkan**, so no compute, so [GPU particles]({{< ref "/post/Particles2" >}}) fall back to the CPU. WebGL through GLES only.

**No native sockets.** No ENet, no raw TCP or UDP.

**No native plugins.** A browser cannot load a `.dll`, so [native interop]({{< ref "/post/NativeInterop" >}}) is unavailable, full stop.

## What it does instead

![What it gets](gets.png)

Every one of those has a fallback, and that is the design rule the whole engine follows: **the same project runs everywhere, sometimes slower.** Not "that feature is desktop-only".

Particles that would use compute use the CPU path and produce [bit-identical results]({{< ref "/post/Particles1" >}}), because the randomness is seeded per particle rather than per thread. Networking uses **WebSocket** and **WebRTC** through native browser backends. HTTP uses the browser's own `fetch()`. Content packs work unchanged, and remote content groups are arguably *more* natural on the web than anywhere else.

And 2.8 added the small thing that makes a web build actually playable on a phone: tapping an InputField on a touch device raises the **native on-screen keyboard**. Without it you have text fields nobody can type into, and the whole build feels broken for one missing hook.

## Size and load time

The thing nobody warns you about: a web build is a **download**, and the player is waiting.

The engine module is what it is, but content is where you have leverage — and it is exactly what the [content system]({{< ref "/post/Content1" >}}) is for. Local groups go in the main pack; everything else becomes remote groups fetched on demand. A game that starts from a small first download and streams the rest feels dramatically faster than one that ships everything up front, even when the total bytes are identical.

Texture compression settings are per platform for the same reason. The web wants smaller more than it wants pristine.

## Testing it

Two things worth knowing.

**You cannot open the HTML from disk.** Browsers block WASM over `file://`. Serve it over HTTP, even locally, or you will spend an hour debugging a security policy.

**Test in more than one browser.** WebGL and WebAudio behave differently enough that "works in Chrome" means less than you would like, particularly on Safari and particularly on iOS.

## Would I recommend it?

For jams, demos and anything you want people to try without installing: unreservedly. A link that plays is worth an enormous amount, and it is the single best reason to have gone through the C# migration.

For a large commercial game: think about it carefully. No threads means the CPU budget is genuinely smaller, and the download is a real barrier at scale.

Comet's position is that the choice should be yours rather than the engine's — which is the whole reason the fallbacks exist rather than a compile error.

---

Next Wednesday: the other platform that migration unlocked. Android — the NDK, four architectures, Vulkan on mobile and the GLES fallback, and what changes when the screen is a finger.

*Comments and questions welcome ;)*
