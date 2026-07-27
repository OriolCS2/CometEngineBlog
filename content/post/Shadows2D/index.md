---
title: Shadows in Two Dimensions
description: Occluders, soft edges, per-light control, and the afternoon everything pointed the wrong way
date: 2026-10-21 18:00:00+0000
image: cover.png
categories:
    - Rendering
tags:
    - Shadows
    - Lighting
    - Tilemap
---

Last Wednesday before Halloween, so: shadows.

Comet's 2D shadows are real-time and dynamic. Move a light, the shadows move. Move an object, its shadow moves. Nothing is baked, which means nothing has to be rebaked when your level changes.

## How a 2D shadow gets decided

![How it works](how.png)

For each light that has shadows enabled, the engine gathers every **Shadow Caster** within its reach. Each occluder's edges are projected away from the light, forming regions the light cannot get into. Those regions are filled, the light is not applied inside them, and then the edge is filtered — either left crisp, or softened.

The important word there is **per light**. Each light does this independently. Two lights in a room produce two sets of shadows, overlapping where they should. It also means shadow cost scales with `lights × occluders`, which is worth remembering before you add the fortieth torch.

A light with `Cast Shadows` off skips the whole thing. Most lights in most scenes should have it off — a small glow on a coin does not need to occlude anything, and turning it off is free performance.

## Telling the engine what blocks light

An entity does not cast a shadow because it has a sprite. It casts a shadow because you put a **Shadow Caster** behaviour on it. That is deliberate: in a 2D scene most things are backgrounds, decals and effects, and if everything blocked light the result would be an unreadable mess.

![Occluders](occluders.png)

The caster can get its shape three ways.

**From the sprite.** It uses the sprite's silhouette. Zero setup, and right most of the time.

**From a custom polygon.** You draw the occluder by hand. More work, and the right answer whenever the visual silhouette is not the shape you want to block light — a character with a big hat, a tree whose canopy should not cast a solid slab.

**From the physics shape.** It reuses the collider. This is my default for level geometry, because the thing that stops you walking through it is exactly the thing that should stop light, and now they can never disagree.

There is a fourth: **Tilemap Shadow Caster**, which builds a single occluder set for a whole tilemap layer rather than one per tile. On a level with three thousand tiles that difference is not an optimisation, it is the difference between working and not.

## The controls

![Shadow settings](controls.png)

Shadow settings live on the **light**, not on the caster, which is the right way round — the same wall should be able to throw a hard shadow from a nearby lamp and a soft one from a distant window.

**Shadow Mode** is crisp or soft. Crisp is a hard edge, and it suits stylised and pixel art. Soft uses PCF — the edge is sampled several times and averaged, giving a gradient. Soft looks better in realistic scenes and costs more.

**Shadow Colour** does not have to be black. This is the single setting that most improves how shadows look, and it is the one people never touch. Real shadows are not absences of light, they are areas lit only by bounced light, which has colour. A dark blue shadow in a warm scene reads as *daylight* rather than as a hole cut in the image.

**Strength** is how much light gets blocked. Below 1.0, some light leaks through, which is often what you want.

**Softness** widens the PCF filter.

**Sorting layers** restrict which layers this light shadows at all. And on the other side, every renderer has its own `Receives Shadows` toggle — so a background can stay evenly lit while everything in front of it darkens. That combination is how you get shadows that support the art instead of fighting it.

## The afternoon everything was inside-out

Shadow Casters support a polygon **cull mode** — clockwise or counter-clockwise — which decides which side of an edge counts as facing the light.

If that sounds like an implementation detail that should not be exposed: I agree, and it is exposed anyway, because polygons authored in different tools wind in different directions and there is no reliable way to guess.

Getting it wrong does not produce no shadows. It produces *inverted* shadows: every area that should be lit is dark and every area that should be dark is lit. The scene is still fully rendered, still plausible-looking at a glance, and completely wrong.

I spent an afternoon on this convinced the projection maths was broken. I rewrote a working function twice. The fix was a dropdown.

## What this costs

Shadows are the most expensive thing in Comet's 2D lighting, by a wide margin.

The rough shape: cost scales with the number of shadow-casting **lights** multiplied by the number of **occluders** each one can reach, and soft shadows multiply that again by the PCF sample count.

Practical advice from having made this slow and then made it fast again:

- **Turn Cast Shadows off by default** and enable it only on lights where a shadow is doing work.
- **Use Tilemap Shadow Caster** rather than per-tile casters. Always.
- **Restrict lights to sorting layers.** A light that only touches the character layer only tests occluders on the character layer.
- **Prefer crisp over soft** unless the art actually needs the gradient. It is a large saving for a small visual difference in stylised work.

## What Comet does not do

No shadow-casting from *emissive* sprites — a glowing object does not automatically light its surroundings. If you want that, put a light on it.

No shadow bounce or global illumination. A shadow is a region a light does not reach; light does not accumulate off surfaces. That is a whole different class of renderer.

And shadows are per-light in screen space, so extremely large numbers of small shadow-casting lights are not the workload this is built for. Fewer, better-placed lights is both the faster answer and the better-looking one.

---

Next Wednesday: the release where I deleted every shader in every project. 2.8 removed GLSL authoring entirely and replaced it with HLSL, cross-compiled to whatever the target platform actually speaks. Why I did something that aggressive, and what a Comet shader looks like now.

*Comments and questions welcome ;)*
