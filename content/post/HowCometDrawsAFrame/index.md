---
title: How Comet Draws a Frame
description: From four thousand sprites to twelve draw calls, and what quietly refuses to merge
date: 2026-09-23 03:00:00+0000
image: cover.png
categories:
    - Rendering
tags:
    - OpenGL
    - Vulkan
    - Performance
    - Sprites
---

Everything in a 2D scene is a textured rectangle. Every sprite, every tile, every letter of text, every particle. That sounds like it should make rendering easy, and in a way it does. Drawing one rectangle is trivial. Drawing four thousand of them without asking the GPU four thousand separate times is the part that takes work.

This post follows one frame, from "here is a scene" to "here are the pixels".

![The demo scene I use through this post](scene.png)

## Culling

![The camera viewport drawn in the Scene view](viewport.png)

That red rectangle in the Scene view is what the active camera can see. Everything outside it is thrown away before it reaches the GPU at all.

Comet keeps a **spatial index** of renderers, a structure that can answer "what overlaps this rectangle?" without testing every object in the scene. On a small scene this does not matter at all. On a level with fifteen thousand tiles where the camera is showing two hundred of them, it decides whether the level runs.

The property I care about is that the cost of culling grows with what is visible, not with what exists. That is why a big Comet level does not get slower as you keep adding more level further away.

There is a bug worth mentioning here, because it shows what happens when the index goes stale. Until 2.8, tiles you painted in the editor could vanish once their tilemap stopped being the active paint target. The tiles were there. The culling tree had not been rebuilt after the stroke, so the renderer believed there was nothing to draw.

## Sorting

2D rendering is mostly about order. There is no depth buffer sorting this out for you, so whatever is drawn last ends up on top.

Comet uses three levels:

1. **Sorting layer**. A named, project-wide list that you define in order. `Background`, `Terrain`, `Characters`, `Effects`, `UI`. This is the coarse control and it is the one you should be using.
2. **Order in layer**. An integer inside a layer, for "this specific thing goes in front of that specific thing".
3. **Depth**. Position along Z, which resolves anything still tied.

I prefer sorting layers over nudging Z values because a layer has a name. Six months later, `Characters` still means something to me. `z = -0.03` does not.

## Batching

![The rendering pipeline, from culling to submission](pipeline.png)

Comet walks the sorted list and merges consecutive renderers that can share a draw call into one. One vertex buffer, one texture bind, one submission.

Fifty coins with the same sprite become one draw call. A tilemap of two thousand tiles from one tileset becomes a handful. This is the step that decides whether a 2D renderer is fast or slow.

![The three things that break a batch](batch-breakers.png)

Three things break the merge:

- **A different material.** A different shader, or different parameters, means a different GPU state.
- **A different texture.** This is what sprite atlases are for. Pack twenty sprites onto one page and they can all batch together. Atlases get their own post in March.
- **Something drawn between them in the sort order.** This one catches everyone. Two sprites using the same material will not merge if a third sprite with a different material sorts between them, because merging them would move that third sprite to the wrong side of something.

That last rule is why the practical advice is to keep things that look alike in the same sorting layer. You are not only organising the scene, you are giving the batcher something it can work with.

## Handing the work over

The batched result becomes a **command buffer**, a backend-agnostic list of "bind this, set that, draw this many". Nothing in it mentions OpenGL or Vulkan.

That buffer is packed into a **frame packet** and handed to the render thread, which turns it into real API calls while the game thread has already moved on to the next frame. On every platform except the web, which has no threads at all, those two things really are running at the same time.

That is the whole of next week's post, so I will stop there.

## The Stats panel

The Game panel has a **Stats** toggle. Turn it on and you get live rendering statistics: draw calls, batches, vertices, triangles and the frame time.

Draw calls are the number I watch when I am optimising, and I find them much more useful than frame rate, because frame rate is capped and noisy while draw calls are exact. Add an atlas and watch the count fall. Add one sprite with a different material into the middle of a batched group and watch it jump.

Some rough numbers from this project. The fifteen-entity scene above sits in single-digit draw calls. A test level with about four thousand sprites drawn from three atlases lands around twelve. A deliberately hostile version of the same level, where every sprite has its own material, gets you four thousand, and a frame time you can feel.

In all three cases it is the same scene, on the same GPU, producing the same pixels. The only thing that changed is whether the renderer was allowed to group the work.

## What Comet does not do

A few limits I should be honest about.

There is no automatic atlasing at build time. If you want sprites to batch, you put them in a Sprite Atlas yourself. I have gone back and forth on this and right now I think the explicit version is better, because automatic packing turns the batching behaviour of your game into something that falls out of an algorithm you cannot see.

There is no draw-call budget warning either. Nothing tells you that you have made things worse. The Stats panel will show you, if you look.

And sorting is per-camera, per-frame, on the CPU. For the object counts a 2D game realistically has, I think that is the right trade. It is also why the GPU particle path, which can have a hundred thousand particles in one system, does its sorting in a completely different way.

---

Next Wednesday: the same frame, drawn twice. Comet renders through an abstraction with an OpenGL backend and a Vulkan backend, picks between them at runtime, and hands the result to a render thread. I will talk about why I put Vulkan in a 2D engine, and what happens on the one platform that has no threads.

*Comments and questions welcome ;)*
