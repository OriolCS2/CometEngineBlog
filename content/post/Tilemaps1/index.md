---
title: "Tilemaps (Part 1): Grids, Palettes and Painting"
description: Building a world one brush stroke at a time, and the collider merging that makes it walkable
date: 2027-03-03 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Tilemap
    - Editor
---

A tilemap is the oldest idea in 2D games and still the best one. Instead of placing four thousand sprite entities, you paint cells on a grid, and the renderer treats the whole thing as one object.

The payoff is not just authoring speed. A tilemap of two thousand tiles from one tileset [batches into a handful of draw calls]({{< ref "/post/HowCometDrawsAFrame" >}}), where two thousand separate sprite entities would not.

![A tilemap in the scene view](in-engine.png)

That is three tilemap layers — ground, scattered detail, buildings — under one grid. Every cell in it is two numbers and a tile index.

## The grid

![Grid types](grids.png)

Everything starts with a **Grid** entity, which decides the cell shape: square, isometric or hexagonal. Under it sit one or more **Tilemap** entities, each one a layer.

That split matters more than it looks. The grid owns the coordinate system; the tilemaps own the content. Change the cell size on the grid and every layer moves together, because they were never storing world positions in the first place — they store cell coordinates.

## The Tile Palette

The Tile Palette panel is where you build your brush set and where you paint from.

![The Tile Palette panel](palette.png)

You fill a palette by dragging in a texture, a set of sprites, or existing tile assets. From 2.1 all three show a **placement preview** at the mouse position, so you can see what you are about to paint before you commit — which sounds obvious and was genuinely missing.

The palette is itself a tilemap. That is why it looks like a level laid out flat: you arrange your brushes spatially, in the shape they will be used in, instead of scrolling a list of thumbnails.

![The tools](tools.png)

The tools are what you would expect: brush, box, fill, erase, and pick. Pick is the one worth forming a habit around — sampling a tile straight off the map is far faster than hunting for it in the palette.

2.3 added a `+` button that creates a new Tilemap inside the currently active Grid without leaving the panel. Small, and it removes a trip to the Hierarchy every time you want a new layer.

## Layers

![Layers](layers.png)

A level is almost never one tilemap. The usual arrangement is background for parallax and sky, terrain for the layer you actually walk on, and foreground for the things that draw over the player.

The problem with layers is that painting becomes ambiguous — which one is your stroke going into? Comet's answer is **focus modes**: the Tile Palette can focus on the active tilemap, the whole grid, or everything, and tilemaps that are not focused render dimmed.

I would put this in the small set of features that decide whether a tool is usable. Painting into the wrong layer and not noticing for ten minutes is a genuinely bad experience, and dimming everything else removes it entirely.

## Colliders, and the seam problem

![Colliders](collider.png)

Add a **Tilemap Collider** and your painted terrain becomes solid. The naive implementation of that gives one collider box per tile, and it produces a specific, infuriating bug: a character running along a perfectly flat tiled floor catches and stutters on the boundaries between tiles.

That happens because floating-point positions on shared edges are not exactly equal, so the physics engine occasionally believes there is a lip between two boxes.

Comet merges adjacent colliders into single shapes, so a run of twenty floor tiles is one collider with two ends and no interior seams. That fix landed in 2.3 and it is the difference between a tilemap you can build a platformer on and one you cannot.

`HasColliderAt` lets a script ask whether a specific cell is solid, which is the cheap way to do grounded checks and wall detection without raycasting.

## Two bugs worth knowing about

Both are fixed, and both illustrate how tilemaps are really a rendering optimisation wearing a level editor's clothes.

**Tiles disappearing after a stroke.** Until 2.8, tiles you painted could vanish once their tilemap stopped being the active paint target. The tiles were there — the culling tree had not been rebuilt after the stroke, so the renderer sincerely believed there was nothing in that region to draw.

**Tiles drawn twice.** Tilemaps in `Individual` rendering mode could skip or duplicate tiles when a scene was rendered by more than one camera in the same frame, because the per-frame state was not per-camera.

Neither was reported as "the tilemap is broken". Both were reported as "sometimes the level looks wrong", which is the hardest kind of bug to act on and the reason the [stats overlay]({{< ref "/post/HowCometDrawsAFrame" >}}) is worth checking when something looks off.

## What this costs

A tilemap is memory-cheap — a cell is an index, not an object — and draw-call-cheap when the tiles share a tileset.

Where it gets expensive is **colliders on very large maps**, because merging runs over the painted cells, and **many layers with many cameras**, because each combination is its own render pass. On a level big enough to notice, the answer is usually fewer, larger tilemaps rather than many small ones.

---

Next Wednesday, the part that stops tilemap painting being tedious: tiles that pick their own sprite. Rule tiles, autotile bitmasks, randomness seeded by position so a level looks varied and never changes, and reskinning a whole ruleset without redoing the rules.

*Comments and questions welcome ;)*
