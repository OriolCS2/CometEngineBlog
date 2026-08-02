---
title: The Year I Deleted Every Shader
description: 2.8 removed GLSL authoring entirely and replaced it with HLSL. Here is why that was worth it
date: 2026-10-28 03:00:00+0000
image: cover.png
categories:
    - Rendering
tags:
    - HLSL
    - Shaders
    - Vulkan
    - OpenGL
---

There is a line in the 2.8 changelog that I suspect cost some people an evening:

> **GLSL Deprecation:** Replaced the GLSL shader authoring workflow entirely with the HLSL pipeline. Existing GLSL shaders must be rewritten in HLSL.

No deprecation period, no compatibility layer. Every custom shader in every project had to be rewritten.

I want to explain why I think that was the right call, because "we changed shader languages" sounds like a matter of taste and it was not.

## The problem

Comet targets Windows, Linux, Android and the web, and it renders through [either OpenGL or Vulkan]({{< ref "/post/TwoBackends" >}}). Put those together and you get this:

![The four shader targets Comet has to feed](problem.png)

Desktop OpenGL wants GLSL 430. OpenGL ES on Android wants GLSL ES 300, with different precision rules and a smaller feature set. Vulkan does not want GLSL at all, it wants SPIR-V, which is a binary intermediate format. WebGL wants GLSL ES too, but an older dialect with fewer capabilities again.

So if you author in GLSL, which GLSL?

Under the old system the answer was desktop GLSL, plus a growing pile of `#ifdef` for the platforms where that was not true, plus a set of hand-maintained variants for the ones where the differences were too large to patch over. Every new shader feature meant checking it against four targets. Every platform bug meant working out which of the four variants had drifted.

That is not really a language problem. It is a problem of not having one authoring source of truth, and it gets worse with every platform you add.

## The fix

Pick one language that nobody's runtime speaks natively, and compile it to all of them.

![The HLSL to SPIR-V to GLSL pipeline](pipeline.png)

You write HLSL. **glslang** compiles it to SPIR-V. Vulkan takes the SPIR-V directly. For the OpenGL family, **SPIR-V Cross** translates it into whatever flavour of GLSL that target needs.

One source file, four outputs. All of it is generated at build time and not at runtime, so the player's machine never compiles anything.

## Why HLSL

I had three reasons, and none of them is "HLSL is a nicer language".

**The toolchain exists and it is excellent.** glslang and SPIR-V Cross are mature, widely used, and maintained by people who care a great deal about correctness. Going from HLSL to SPIR-V to GLSL is a well trodden path. Going from GLSL to SPIR-V to another GLSL is possible but far less common, and you hit rough edges nobody else has hit before.

**It is the industry default for this job.** Anyone who has written shaders for a modern engine has written something HLSL shaped. For a small engine that is a real advantage, because I do not have to teach the language, only the specifics of Comet.

**It is more explicit.** HLSL's explicit semantics and register bindings map onto Vulkan's descriptor model much more directly than GLSL's implicit conventions do, so there is less guessing in the middle of the pipeline.

## What a shader looks like now

![A Comet HLSL shader open in the editor](shader.png)

`#pragma vertex` and `#pragma fragment` name your entry points. Both stages are fully yours, and that was a real gain over the old system, where vertex stage customisation was limited and vertex animation meant working around the engine instead of with it.

You include a Comet header that provides the standard structures and helpers, you declare your parameters as plain globals, and you write two functions. Parameters declared at the top surface automatically as material properties, so they appear in the material inspector and you can set them from AngelScript with `Material.SetFloat` and friends.

There are template shaders under `Create → Shader` for the common starting points, including a post-process one, which is [three weeks from now]({{< ref "/post/PostProcessing" >}}).

## Was the hard break necessary?

I have thought about this a lot and I think yes, although not comfortably.

The alternative was supporting both: keep GLSL authoring working, add HLSL next to it, and deprecate GLSL over a few releases. That is friendlier to everybody. It is also two shader front-ends, two sets of bugs, two sets of platform quirks, and every new feature has to be built twice or marked as HLSL only. For a one-person engine I would have been paying that cost forever, and not only during a transition.

The facts that made it easier, stated honestly: Comet 2.x was only a few months old, the number of projects with custom GLSL shaders was small, and 2.0 had already broken everything anyway by [removing C#]({{< ref "/post/GoodbyeCSharp1" >}}). If there was ever a moment to do it, that was the moment.

If Comet had a large user base with years of shaders behind it, I do not think I would have had the right to make that call.

## Why this matters less than it sounds

The thing that made the decision much easier for me is that most people will never write a shader by hand.

2.8 shipped HLSL authoring and the **Shader Graph** in the same release, and that was not a coincidence. If you can build a dissolve, a hologram or a water surface by wiring nodes together with a live preview, the number of times you need to open a text file and write shader code drops enormously.

The graph compiles down the same pipeline and produces the same kind of output the HLSL front-end does. It is not a slower path or a limited subset of it.

So what 2.8 really did to shaders is this: writing them got harder for the small number of people who write them by hand, and it got a lot easier for everyone else.

---

Next Wednesday, that Shader Graph. The canvas, the node palette, the live preview, and a dissolve effect built from nothing, one node at a time.

*Comments and questions welcome ;)*
