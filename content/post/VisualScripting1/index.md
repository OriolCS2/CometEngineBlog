---
title: "Visual Scripting (Part 1): Graphs That Compile to Bytecode"
description: Not interpreted. The node graph becomes AngelScript bytecode and runs at exactly the same speed
date: 2027-01-27 03:00:00+0000
image: cover.png
categories:
    - Scripting
tags:
    - Visual Scripting
    - AngelScript
    - Editor
---

`Create → Script → Visual Script` makes an asset that looks like every other node editor you have used, boxes, wires and a canvas.

What makes it worth a post is what happens when you press play.

![A visual script graph](graph.png)

That is a whole behaviour. It spins this entity by `spinSpeed` degrees a second and logs a line when the player presses E. The `spinSpeed` variable on the left is marked **Exposed**, so it shows up in the Inspector like a field on a written script, because that is exactly what it becomes.

## The claim

![How the graph turns into code](claim.png)

Most visual scripting systems work like this. The graph is data, and at execution time a runtime walks that data and interprets the nodes one at a time. It works, and it is slower than written code, usually by enough that "use visual scripting for prototyping and then rewrite it properly" ends up being the standard advice.

Comet's graphs are source rather than data. A visual script compiles into AngelScript bytecode, the same bytecode a hand-written `.as` file produces, and the running game cannot tell the difference, because there is no difference.

![The generated AngelScript](generated.png)

That is the file the graph above produced, ordinary readable AngelScript with its `[Serialize]` attribute and all. The header is not a joke, the file really is regenerated on every save, so the graph is the source of truth and the text is a build artefact you can read.

There is no graph runtime and nothing walks the nodes. At play time the graph does not exist, only the bytecode does.

That was the design goal before I implemented a single node, and it is the reason the feature exists at all. I did not want to build a visual scripting system that I would then have to tell people to rewrite later.

## Where the nodes come from

![The node palette](nodes.png)

The second design decision is the one that makes this maintainable by one person. I do not write the node library. Nodes are generated automatically from the engine's API, so every function, every type, every variable and every event that AngelScript can see becomes a node without anybody authoring it.

The consequence is that the node library cannot fall behind the engine. When I add a method to `Rigidbody`, a node for it exists. When 2.8 added the `Assets::` namespace, the whole namespace showed up as nodes. There is no wrapper layer to maintain, no list of "supported" API, and no gap between what you can do in text and what you can do in a graph.

Your own script classes appear too. Write a `Health` behaviour in AngelScript and its public methods and fields are nodes, so text scripts and graphs work with each other in both directions instead of living in separate worlds.

## Building something small

To make it concrete, an enemy that patrols between two points and chases you when you get close.

Start with an **Update** event node, which is the entry point. Wire in a **Distance** node comparing this entity's position to the player's. Feed that into a **Compare** against a threshold. Branch.

On the near side, a node that moves toward the player's position. On the far side, the patrol logic, a stored target, a move toward, a distance check and a flip.

That is maybe fifteen nodes and no typing beyond a couple of numbers and a name. It is not less work than writing it, and for me it is slightly more. I think that is worth saying.

## So who is it for

![When to use it](when.png)

There are three cases where I reach for a graph instead of a file.

**Event-driven logic.** Anything shaped like "when X happens, do these things in order" reads well as a graph, because the flow is the shape. A cutscene trigger, a door that needs three switches, a pickup that plays a sound and spawns particles and adds score.

**Behaviour someone else will tune.** A designer who is not going to open a code editor will happily rewire a graph. That is a real category of person even on a solo project, because six months from now I will not remember the code either, and a graph is legible at a glance in a way a file is not.

**One-off moments.** Scripted set pieces that will never be reused and do not deserve a class.

And then the cases where I do not use one: heavy maths, long algorithms, and anything I want to review as a diff. A graph is a file, but not one you can read in a pull request. That is the same complaint I have about [Shader Graph]({{< ref "/post/ShaderGraph2" >}}), and I do not have a good answer to it in either place.

## What was hard

Control flow. Data flow between nodes is straightforward, outputs connect to inputs and you evaluate them in dependency order. Execution flow is a different problem: which node runs next, what a branch means, how a loop terminates, what happens when an event fires in the middle of execution. Those two graphs are drawn on the same canvas and they have to stay understandable together.

Generating good code and not just correct code. A naive translation of a graph produces a temporary for every wire and a function call for every node. It works, and it is measurably slower, which would have broken the whole premise. Getting the emitted AngelScript into a shape the compiler optimises well took me much longer than getting it to emit anything at all.

Type coercion, again. It is exactly the problem [Shader Graph had]({{< ref "/post/ShaderGraph1" >}}). A user drags a wire between two pins and expects it to work. Too strict and it is exhausting, too loose and graphs silently do the wrong thing. Same trade-off, same compromise, and I am still not certain it is in the right place.

Deciding what is not a node. If every engine function is a node, the palette has thousands of entries. Search carries most of that weight, and I still think the palette needs better organisation than it has.

## What I think of it

Visual scripting in Comet is not faster to write than text if you are comfortable with text. It is not a shortcut and I would not tell anyone that it is.

What it does give me is something legible at a glance, tunable by people who do not write code, impossible to fall behind the engine API, and free at runtime, which is the part that took all the work.

---

Next Wednesday, the thing graphs do better than text: you can watch one think. Nodes lighting up as execution flows through them in play mode, hovering a pin to see the value passing through it right now, and breakpoints you set on a node.

*Comments and questions welcome ;)*
