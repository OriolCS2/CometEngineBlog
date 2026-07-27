---
title: Getting From A to B
description: Grid A* with jump point search, navmeshes, agents, and per-agent rules about what counts as walkable
date: 2027-04-28 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Navigation
    - Tilemap
---

Pathfinding is one of those systems where the algorithm is the easy part. A* fits on a napkin. What takes the time is everything around it: where the graph comes from, what happens when the world changes, and the fact that not every agent agrees about what is walkable.

2.8 added the whole system, and it has two halves because 2D worlds come in two shapes.

![Two systems](two.png)

## Grids, when your world is cells

`CometEngine::AStarGrid` is a uniform grid of cells, each solid, free, or free-but-expensive.

If your world is a [tilemap]({{< ref "/post/Tilemaps1" >}}) — and in 2D it very often is — the grid already exists, and `Navigation::FillGridFromTilemap` populates an `AStarGrid` directly from a tilemap's occupied cells. One call between "I have a level" and "I have a pathfinding graph", with no separate authoring step and nothing to keep in sync.

![Grid options](gridopts.png)

The knobs matter more than they look.

**Diagonal modes** — four of them. "Never" for grid-locked movement. "Always" for free movement. And two variants that refuse to cut corners: no diagonal movement through a gap between two solid cells, or no diagonals near obstacles at all. That distinction is the difference between an agent that walks through a wall corner and one that does not, and it is a checkbox rather than a bug you hunt.

**Heuristics** — Euclidean, Manhattan, Octile, Chebyshev. Match the heuristic to the movement you allow. Manhattan with diagonal movement enabled overestimates, and an overestimating heuristic makes A* stop being optimal — it will find *a* path, not the shortest. This is the single most common way to get subtly bad paths.

**Jump Point Search** is the optimisation that matters on open grids. Rather than expanding every cell, JPS skips along straight runs and only considers cells where the path could meaningfully turn. On a large open map it is dramatically faster for identical results. On a maze it helps much less, because there are turns everywhere.

`CometEngine::AStar` is the same algorithm on an arbitrary weighted point-and-edge graph, for when your world is rooms and doors rather than cells.

## Navmeshes, when your world is not cells

![The navmesh side](navmesh.png)

For open, irregular spaces a grid is the wrong shape — you either make cells small and pay for thousands of them, or make them large and lose precision.

The `NavigationServer` bakes **walkable polygons** instead, with an agent-radius inset so an agent never plans a path its own body cannot fit through. That inset is the thing most people forget when writing their own: a path along a wall is only valid if you are a point.

Four behaviours: **NavigationRegion** defines walkable area, **NavigationAgent** follows paths and tracks a target, **NavigationLink** connects places the mesh cannot — a jump, a ladder, a teleporter — and **NavigationObstacle** carves holes with a circle, box or polygon.

![Two baked regions with obstacles and a link](baked.png)

Two regions, baked. The triangulation is the mesh the server actually searches; the circles punched out of it are obstacles; and the pair of green markers with a line between them is a NavigationLink bridging the gap the mesh cannot cross.

Obstacles are how you handle a door that closes or a crate that gets pushed, without rebaking the whole mesh.

![An agent's computed path](agent-path.png)

Select an agent while the game runs and you get the path it is currently following, corner by corner. Note that it does not hug the obstacle — the agent-radius inset keeps the whole body clear of it.

## Agents that disagree

![Callbacks](callbacks.png)

This is the part I am most pleased with, because it solves a problem most pathfinding systems handle badly.

Your world has a flying enemy that ignores walls, a heavy enemy that cannot cross the bridge, and a normal one. Three different notions of traversable, over the same world.

The usual answers are all bad: three separate graphs (memory, and three things to keep in sync), or one graph plus filtering after the fact (which produces wrong paths, not filtered ones).

Comet lets each query supply script callbacks: `SetComputeCostFunction` for what a step actually costs, `SetEstimateCostFunction` for the heuristic, and `SetFilterNeighborFunction` for whether a neighbour exists at all for *this* agent.

One grid. The flyer's filter accepts solid cells. The heavy unit's cost function returns infinity on the bridge. Nothing is duplicated.

## What this costs

Pathfinding is the system most likely to spike a frame, because a path request is not incremental — it either completes or it does not.

Practical advice: **do not path every agent every frame.** Stagger requests, and reuse a path until the world or the target moves meaningfully. **Prefer grids with JPS** for open maps. **Keep navmeshes coarse**; more polygons is more search.

And the one people miss: **a failed path is expensive.** If the target is unreachable, A* explores the entire reachable set before concluding that. An enemy repeatedly pathing to a player behind a locked door can cost more than the whole rest of your AI. Check reachability cheaply first, or cache the failure.

## What is missing

No flow fields, so a hundred agents heading to the same place each compute their own path rather than sharing one.

No local avoidance in the navigation system itself — agents follow paths but do not steer around each other. RVO is in the dependency list and this is the gap I would close next.

No dynamic navmesh rebaking at runtime; obstacles carve, but changing the geometry means a rebake.

---

Next Wednesday: the other end of the loop. Input — actions instead of key codes, the generated wrapper, controllers including the strange buttons, and touch and pinch on mobile.

*Comments and questions welcome ;)*
