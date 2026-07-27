---
title: "Tilemaps (Part 2): Tiles That Think"
description: Rule tiles, autotile bitmasks, and randomness that looks random and never changes
date: 2027-03-10 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Tilemap
    - Editor
---

[Last Wednesday]({{< ref "/post/Tilemaps1" >}}) covered painting cells. This week is about not having to.

Here is the problem. You draw a grass tileset: a flat top, a left edge, a right edge, an inner corner, an outer corner, a single isolated block, and so on — around fifteen sprites for a convincing terrain. Then you paint a hill, and you have to pick the correct one of those fifteen for every single cell, by eye, forever.

Tiles that think do that for you.

![The tile types](types.png)

## RuleTile

The general case. A **RuleTile** holds a list of rules; each rule says "if the neighbours look like *this*, use *that* sprite". Rules are evaluated top to bottom and **first match wins**, which makes the ordering meaningful: put your specific cases above your general ones.

You paint one tile type. The tile works out which sprite to show from what is around it, and it re-evaluates when the neighbourhood changes — so drawing a hill just produces a correct hill.

2.5 added **RuleOverrideTile**, which takes an existing RuleTile's rules and swaps only the sprites.

![Reskinning](variants.png)

That is the feature that makes the whole system pay off across a project. Author terrain logic once; get grass, snow and desert by supplying three sprite sets. If you later fix a rule, all three inherit the fix.

There are **IsometricRuleTile** and **HexagonalRuleTile** variants too, because "which neighbours count as adjacent" is a different question on those grids.

## AutoTile, and what a bitmask really is

![The bitmask](bitmask.png)

**AutoTile** is the formalised version of the same idea, and it is worth understanding because the numbers look arbitrary until they do not.

Look at a cell's eight neighbours. Each one either is or is not the same tile. Eight yes/no answers is eight bits, so 256 possible neighbourhoods.

But most of those are visually identical. Whether a diagonal neighbour exists only matters if *both* of its adjacent orthogonal neighbours also exist — otherwise that corner is already an edge and the diagonal changes nothing. Collapse all the duplicates and 256 possibilities become **47 distinct cases**.

Hence the 47-tile layout. Draw 47 sprites and every possible terrain shape resolves itself. Comet also supports the simpler **2x2 / 16-tile** layout, which only considers four neighbours — fewer sprites to draw, blockier results, and completely fine for a lot of art styles.

## Random that stays put

![Seeded random](seeded.png)

**RandomTile** picks a random sprite from a set. **WeightedTile** does the same with per-sprite probabilities, so your rare cracked-stone variant shows up one time in twenty instead of one in five.

The important design decision is in how the randomness works. A naive implementation calls a random number generator when the tile is drawn — and then the ground shimmers as sprites change every frame, or the level looks different every time you load it.

Comet seeds the choice from the **tile's position**. Cell (14, 7) always produces the same "random" sprite, forever, on every machine. The map looks varied, never changes, and **nothing has to be stored** — the variation is a pure function of the coordinate, so a hundred thousand randomised tiles cost zero bytes of save data.

That is my favourite kind of solution: it makes the feature work correctly and makes it free at the same time.

## PipelineTile

**PipelineTile** checks only the four orthogonal neighbours and picks the sprite that connects correctly. Pipes, fences, wires, rope bridges, roads — anything where diagonals are meaningless and you need exactly the sixteen connection cases.

Simpler than a full autotile, and the right tool when the thing you are drawing is genuinely a network rather than a surface.

## The one that bit me

Removing a `TileAsset` that was currently in the active Tile Palette crashed the editor, right through to 2.1.

It is the obvious bug in hindsight — the palette held a pointer to something the project browser had just deleted — and it is the kind of thing that only shows up once someone is *iterating*, because you have to actually change your mind about a tile to hit it. Building a level from scratch never triggers it. Redesigning one does.

That is a small argument for using your own tools for real work rather than only for demos.

## Where the ceiling is

Rule evaluation happens when a tile's neighbourhood changes, not per frame, so a static level costs nothing at runtime. Painting is where you feel it: a large box-fill with a complex RuleTile re-evaluates every affected cell and their neighbours, and on a big stroke you will notice.

The other limit is authoring. A RuleTile with forty rules is genuinely hard to reason about, because "first match wins" means the behaviour depends on an ordering you cannot see all of at once. When a ruleset gets that big, the honest answer is usually two simpler tile types rather than one clever one.

---

Next Wednesday: the art that fills all those cells. The Sprite Editor — slicing, pivots and 9-slice borders — sprite atlases, and the reason all of it exists, which is draw calls.

*Comments and questions welcome ;)*
