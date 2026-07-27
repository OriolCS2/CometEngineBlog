---
title: Everything Falls Down
description: Box2D 3, colliders, joints, the collision matrix, and three bugs that each taught me something
date: 2027-04-07 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Physics
    - Box2D
    - Editor
---

Comet's physics is **Box2D 3.x**. I did not write a physics engine and I am not going to — Box2D is twenty years of accumulated correctness about a genuinely hard problem, and the sensible thing is to wrap it well.

So this post is mostly about the wrapping: what Comet exposes, which decisions are yours, and where the sharp edges turned out to be.

## Bodies and shapes

![Body types](bodies.png)

Three body types, and picking correctly is most of your performance budget. **Static** never moves — walls and floors — and is the cheapest by a wide margin. **Kinematic** is moved by your code and ignores forces; moving platforms, doors, anything on rails. **Dynamic** is simulated.

The common mistake is making level geometry dynamic "just in case". A static body participates in far less of the solver's work.

![Collider shapes](shapes.png)

Shapes attach to bodies. Box and circle are fastest and cover most things. **Capsule** is the character shape — a box catches on tile corners in a way a capsule does not, and that alone fixes a class of platformer bug. Edge and polygon are for arbitrary outlines and cost more. And a [Tilemap Collider]({{< ref "/post/Tilemaps1" >}}) turns a whole painted layer into merged runs.

2.2 fixed capsule editing, where dragging one handle moved all of them — the sort of thing that makes an editor feel broken even when the runtime is fine.

## The collision matrix

![The matrix](matrix.png)

Layers plus a grid of which layers test against which. Player, Enemy, Pickup, Terrain, and a checkbox for each pair.

Everyone treats this as a gameplay tool — bullets should not hit the player who fired them. It is equally a **performance** tool. Every unchecked box is a set of pairs the broadphase never has to consider. On a level with three hundred pickups, "pickups do not collide with pickups" removes about forty-five thousand potential pairs.

## Triggers, and the ordering trap

A collider marked as a trigger reports overlaps without resolving them, and you get `OnTriggerEnter`, `OnTriggerStay`, `OnTriggerExit` — plus the collision equivalents for solid contacts.

Here is the trap, and it is worth stating loudly: **you are inside the physics step when those run.**

Disabling a collider inside `OnTriggerEnter` mutates the list the solver is iterating. That crashed the editor until 2.8. Comet now defends against it, and the general habit is still worth forming — if a callback needs to destroy or disable something, mark it and act at the end of the frame. [Deferred destruction]({{< ref "/post/EntitiesAndBehaviours" >}}) exists for exactly this reason.

2.2 fixed the related case of a collision or trigger callback firing on an entity whose script had gone missing.

## Raycasts

`Physics::RaycastClosest` and `RaycastAny`, with layer masks and distance bounds.

2.7.2 made both **return `null` when there is no hit**, rather than an object you had to inspect. That is a small API change with a real effect: a miss is now impossible to accidentally treat as a hit, because you have to check.

2.4 also fixed a default that had been wrong for a long time — the default `minDepth` for raycasts was positive when it should have been negative, so raycasts silently ignored anything behind the plane you would expect them to cover.

## Three bugs

![Three bugs](wars.png)

These are worth telling because none of them presented as a physics bug.

**The joint that connected a body to itself.** A specific [InstanciableEntity override]({{< ref "/post/InstanciableEntities2" >}}) configuration produced a joint whose two ends resolved to the same body. Box2D does not expect that. The world corrupted, and the crash arrived several frames later in unrelated code, so every stack trace pointed somewhere innocent. I spent a week convinced the integration was fundamentally broken. Fixed in 2.8, along with a crash when dragging a rigid body with the scene gizmos while a joint was attached.

**Negative scale.** Negative transform scale values were being lost during matrix decomposition, so a sprite flipped by scaling `x` to `-1` silently reverted. That broke sprite pivots, facing vectors and physics alignment together — and it looked like four separate bugs in four separate systems. One fix in 2.8 closed all of them.

**The tilemap collider seam.** Covered in the tilemap post, but it belongs here: per-tile collider boxes let a character catch on the boundary between two flat floor tiles, because shared edges at floating-point positions are not exactly equal.

The pattern across all three: physics bugs surface as *timing* and *geometry* weirdness far from the cause, because the solver's state is global and the corruption outlives the frame that caused it.

## What Comet does not do

No continuous collision for everything by default — fast small objects can tunnel unless you enable it where it matters. That is a Box2D setting exposed rather than a Comet decision, and it is a real cost/benefit choice.

No 3D. Obviously, but worth saying: Comet is 2D throughout, and physics is the system where people most often expect an escape hatch. There is not one.

No soft bodies, no cloth, no fluids. Box2D is a rigid body solver.

---

Next Wednesday: the other thing that touches everything, and the one people fight hardest. UI — canvases, rect transforms, anchors explained until they click, and the 2.8 snapping guides that made laying one out bearable.

*Comments and questions welcome ;)*
