---
title: "Shader Graph (Part 2): Blackboard, Sub Graphs and Vertex Offset"
description: Turning a graph into a material somebody else can use — and moving the geometry, not just colouring it
date: 2026-11-11 18:00:00+0000
image: cover.png
categories:
    - Rendering
tags:
    - Shader Graph
    - Shaders
    - Editor
---

[Last Wednesday]({{< ref "/post/ShaderGraph1" >}}) built a dissolve. It worked, and it had one flaw that made it nearly useless: every value in it was hard-coded inside the graph. Want a slower dissolve, or a blue burning edge instead of orange? Open the graph and edit it.

That is not a shader. That is one specific effect.

## The Blackboard

![Properties](blackboard.png)

The **Properties** section is how a graph stops being a single effect and becomes a template.

Add a property, wire it in place of a constant, and it appears in the **material inspector**. Now one graph backs fifty materials that all behave differently. The dissolve gets a `_Progress` float, an `_EdgeColor`, an `_EdgeWidth` and a `_NoiseScale`, and suddenly it is a dissolve *system* — a fast blue one for teleports, a slow orange one for burning, a chunky one for stone.

![Property types](property-types.png)

Floats, vectors, colours, booleans, textures and matrices. Whatever you expose is also reachable from AngelScript through `Material.SetFloat`, `SetColor`, `SetTexture` and friends — which is the bit that makes this a gameplay feature rather than an authoring convenience. Animating `_Progress` from 0 to 1 over half a second in a script is how the dissolve actually gets used.

They can be animated by the Animator too, since a material property is just a value with a name.

## Graph Settings, revisited

The settings block is small and every switch in it matters.

**Target** decides what kind of shader this is — Sprite, UI, and (from 2.8) Post-Process, which is [next week]({{< ref "/post/PostProcessing" >}}).

**Lit (2D light map)** decides whether the result participates in [2D lighting]({{< ref "/post/Lighting1" >}}). Off, and your shader ignores every light in the scene. This is the switch people forget when their beautiful custom material comes out flat in a lit scene.

**Alpha Clip** enables discarding pixels, which the dissolve needs.

**Vertex Color Tint** multiplies by the per-vertex colour, so the sprite renderer's Color field still works. Turn it off and your material stops responding to tinting, which is occasionally what you want and usually a bug.

**Is Sub Graph** turns the whole thing into a reusable node. More on that below.

## Keywords

![Keywords](keywords.png)

A keyword compiles two versions of the shader — one with a branch included, one with it removed entirely. A material picks which variant it uses.

The value is that the disabled variant does not contain the code at all. There is no runtime branch and no cost. A glow you can toggle per material genuinely costs nothing on the materials that have it off.

The trap is combinatorial. Each keyword doubles the variant count. Three keywords is eight shaders to compile at build time; six is sixty-four. Use them for genuinely separate looks, not for things a float could express.

## Sub Graphs

![Sub graphs](subgraphs.png)

Once you have written a few graphs you notice you keep rebuilding the same cluster of nodes. A particular noise setup, a UV distortion, a colour remap.

Set **Is Sub Graph** and the graph becomes a node. Its exposed inputs become input pins, its outputs become output pins, and you drop it into other graphs like any built-in node. Fix a bug in the sub graph and every graph using it is fixed.

This is the feature that decides whether a node-based system stays usable past about twenty nodes, and it is worth reaching for earlier than feels necessary.

## Vertex Offset

![Vertex offset](vertex-offset.png)

Everything so far has been about colour. Vertex Offset is the input that moves the geometry.

The master node has an input that takes a `float3` added to each vertex position before it is transformed. It is a small hook with a disproportionate payoff, because it turns a shader into an animation system that runs entirely on the GPU:

- **A waving flag.** Take the vertex's X, feed it into a sine with time, offset Y. The quad ripples.
- **A floating pickup.** Sine of time into Y offset, no position input. It bobs. No script, no Animator, no CPU cost.
- **Grass in wind.** Same as the flag, but mask the offset by the vertex's Y so the base stays planted and only the top moves.
- **Heat haze or jelly.** Noise into the XY offset.

The thing to appreciate is that this is free. The vertices were being transformed anyway; adding an offset to that transform costs essentially nothing, and it happens on the GPU for every instance without the CPU knowing.

The limit: it moves *vertices*, and a sprite has four of them. You cannot bend a sprite smoothly along its length, because there is nothing in the middle to bend. For that you need geometry with more vertices — a tilemap, a line renderer, or a mesh built for it.

## The node library

About 115 nodes, grouped into families: **Artistic** (22 blend modes, filters, adjustments), **Channel**, **Input** (geometry, camera, time, object data), **Math**, **Procedural** (noise, shapes, patterns), **UV** and **Utility**.

Two in Utility are worth knowing about specifically. **Custom HLSL** lets you drop a snippet directly into the graph, and **Expression** evaluates a small formula inline. Both exist so that hitting the limits of the node vocabulary does not mean abandoning the graph and rewriting the whole shader by hand.

## What I would change

Two things bother me.

**Large graphs are hard to read.** Past forty or so nodes, following a wire across the canvas is genuinely difficult. Sub Graphs are the answer and I do not think they are enough — I would like some form of grouping or reroute nodes.

**No diffing.** A shader graph is a file, but not one you can review in a pull request in any meaningful way. Text shaders diff. Graphs do not, and I have no good answer for that yet.

---

Next Wednesday: taking the whole rendered frame and doing something to it. Post-process profiles as a stack, the `Post-Process` graph target with its Scene Color and Screen UV nodes, writing one in HLSL instead, and why a post-process pass costs exactly the same on an empty scene as on a full one.

*Comments and questions welcome ;)*
