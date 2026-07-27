---
title: "A Hundred Thousand Particles (Part 2): Moving It to the GPU"
description: Three compute kernels, an indirect draw, and a CPU that never learns how many particles are alive
date: 2026-12-02 18:00:00+0000
image: cover.png
categories:
    - Performance
tags:
    - Particles
    - Compute
    - Vulkan
    - Shaders
---

[Last Wednesday]({{< ref "/post/Particles1" >}}) ended at 100,000 particles in 2.7 ms on the CPU. That is a good number. It is also 2.7 ms of a CPU doing arithmetic on arrays, which is precisely the work a GPU exists to do.

So 2.8 added a **Simulation Mode** dropdown. Set it to GPU and the entire simulation moves.

![The GPU inspector](gpu-inspector.png)

## Compute shaders, briefly

If you have not written one: a compute shader is a program that runs on the GPU but does not draw anything. No vertices, no pixels — just "run this function a hundred thousand times in parallel, here is a buffer, do maths on it".

That is an almost perfect description of a particle update. Every particle does the same arithmetic to different data, and no particle depends on any other. It is the textbook case.

Comet exposes compute through the render hardware abstraction, gated behind `IRenderHardware::SupportsCompute()`. It is available on Vulkan and on desktop OpenGL 4.3+. It is **not** available on OpenGL ES or on the web, and that constraint shapes everything below.

## Three kernels

![The kernels](kernels.png)

**Emit** spawns new particles, taking indices off a free list held on the GPU.

**Update** runs every property module — lifetime, velocity, forces, limit velocity, size, colour and rotation over lifetime and by speed, texture sheet animation. All of it, in one dispatch.

**Compact** is the clever one. After an update, some particles died. The survivors need to be gathered into a contiguous alive list, and the naive way — each thread atomically appending itself — produces a list in whatever order the atomics happened to land. That order is essentially random and it changes every frame.

Comet instead compacts with a **prefix sum**, which preserves order. The alive list stays sorted oldest-first, deterministically.

That one decision buys two things. **Force Respawn** mode can recycle the genuinely oldest particle, because it knows which one that is. And both CHUNK sort modes become free — sorting front-to-back or back-to-front is just walking the alive list forwards or backwards.

**Draw** is a single `DrawIndirect`. The vertex shader reads the particle pool and the alive list and expands each entry into a quad on the fly.

## What that last part means

The draw arguments — including *how many particles to draw* — live in a GPU buffer written by the compute kernels. The CPU issues one indirect draw call and never learns the count.

No particle data is read back. Nothing crosses the bus in the wrong direction. The CPU's entire per-frame involvement with a GPU-simulating system is: dispatch three kernels, issue one indirect draw.

This is why the counts get absurd. The budget is **256,000 particles per system and 2 million in total**, and those are budget numbers rather than performance cliffs.

## Real numbers, from writing this post

I set a system to GPU mode with a 50,000 particle maximum while preparing these images, and asked the editor what it was actually doing:

![Measured](measured.png)

```
requestedMode        = gpu
simulatingOnGpu      = true
backendSupportsCompute = true
gpuCapacity          = 50000
gpuBytes             = 4876888
```

50,000 particles in **4.88 MB** — about 97 bytes each, against 169 bytes on the CPU path. The GPU layout is tighter because several CPU-side bookkeeping fields have no equivalent when the pool manages itself.

That readout comes from the `particle_get_status` tool, and the same information is shown live in the Inspector, which matters for the next section.

## The rule: GPU is a request

![Fallback](fallback.png)

Here is the design decision the whole feature rests on. **Setting Simulation Mode to GPU does not mean the system runs on the GPU.** It means you would like it to.

A system silently keeps simulating on the CPU when:

- the backend has no compute support — OpenGL ES, WebGL, older desktop GL
- the compute kernels were not baked into this build
- the global GPU particle budget is already spent
- the system uses an authoring feature the kernels cannot reproduce *faithfully*

That last one is the important word. Not "cannot reproduce at all" — cannot reproduce **identically**. If a GPU path would produce a slightly different result from the CPU path, Comet takes the CPU path, because a particle effect that looks different on a phone than on a desktop is a bug you will find at the worst possible moment.

Three things always stay on the CPU:

- **Individual render mode**, which depth-sorts every particle against the other renderers in its layer. That needs positions back on the CPU each frame, which defeats the entire point. Use Chunk, which now sorts on the GPU for free.
- **Sprite lists whose frames span more than one texture.**
- **UI particles.**

And the system tells you. The Inspector reports where it is *really* simulating, not what you asked for. That feedback is what makes the fallback acceptable rather than mysterious.

## Was it worth it?

For most 2D games, honestly: no. If your biggest effect is two thousand particles, the CPU path handles it in well under a millisecond and the GPU path adds nothing.

It was worth it for two reasons. Ambient effects at counts that were previously impossible — rain across a whole level, a snowfield, a hundred thousand drifting embers — are now free enough to just leave running. And building it forced compute and indirect drawing into the render hardware abstraction properly, which is infrastructure everything else can now use.

---

Next Wednesday: how you find out any of this. The profiler — flame graphs, deep script profiling down to individual branches, auto-stop on a frame spike, and attaching the editor's profiler to a game running as a standalone build.

*Comments and questions welcome ;)*
