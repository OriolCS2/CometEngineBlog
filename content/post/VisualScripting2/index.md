---
title: "Visual Scripting (Part 2): Watching a Graph Think"
description: Nodes light up as execution flows through them, pins show live values, and you can stop on one
date: 2027-02-03 18:00:00+0000
image: cover.png
categories:
    - Scripting
tags:
    - Visual Scripting
    - Debugging
    - Editor
---

[Last Wednesday]({{< ref "/post/VisualScripting1" >}}) made the case that a Comet visual script is not slower than a written one, because it compiles to the same AngelScript bytecode.

That is a reason to be willing to use graphs at all. This post is about why I sometimes prefer them, and it comes down to one thing a text file can never do.

## The graph runs in front of you

![A graph being debugged live](live.png)

![The graph, at rest](graph.png)

That is the graph sitting still. Everything below is about what it looks like when it is not.

Enter play mode with a graph open and it stops being a diagram. The nodes on the executing path light up as control flow reaches them. It is not a replay and it is not a log, it is the actual path taken this frame.

Hover any pin and you see the value passing through it right now, the number that is in the wire at this moment, not the one you logged three frames ago.

And you can put a **breakpoint on a node**. The game halts there exactly like a line breakpoint in the [code editor]({{< ref "/post/TextEditor2" >}}), and it uses the same debugger underneath, because the graph compiled into the same bytecode that debugger already understands.

## The bug this kills

![Before and after](why-it-helps.png)

"Live debugging" is a vague thing to say, so let me be specific about the kind of problem it solves.

You have an enemy that should chase the player when they get close. It does not. The chase branch never runs.

Without live debugging you start guessing. Add a log to the distance check. Replay. Read the console. The distance looks fine. Add a log to the comparison. Replay. Now the comparison looks fine too, which means the problem is somewhere you have not instrumented, so you instrument more and replay again.

With it, you watch the branch not light up, you hover the condition pin, and you see that the comparison is reading `0` because the player reference was never assigned. That takes about four seconds, and you never touched the graph.

Logging is not slow in itself, but it only ever shows me the thing I already suspected, while the lit up path shows me everything at once.

## Why this is easier in a graph than in text

A text debugger can show you a call stack and the locals in scope. That is a huge amount of information and it is presented to you as a list.

A graph presents the same information as the shape you drew. You do not have to rebuild the control flow in your head, because the control flow is the picture, and the picture is lit up.

That advantage goes away as soon as the graph is big enough that you cannot see all of it at once, which brings me to the limitations.

## What it still cannot do

![The current limits](limits.png)

There are no conditional breakpoints. Break when `health < 0` is not there, in graphs or in text. It is the single feature I would add next to both.

Large graphs are hard. Past about forty nodes, following a wire across the canvas is genuinely difficult, and lighting up the path helps less than you would hope, because the path leaves the screen. Sub-graphs are the answer and I do not think they are a complete one.

There is no diff. A graph is a file, but not one you can read in a pull request. For a solo project that costs me nothing. For a team it is a real problem, and it is the reason I would still write shared, long lived systems as text.

## So when do I actually use one?

Being concrete, after a year of having both.

I use graphs for things I will tune more than I will write, so enemy behaviour, encounter pacing, anything with numbers a designer wants to fiddle with. Being able to watch it run while I change it is worth more there than anywhere else.

I use graphs for one-off scripted moments, the door that needs three switches, the cutscene trigger. They will never be reused, they are easier to read as a picture six months later, and they do not deserve a class.

I use text for anything with real algorithms, anything I want to review as a diff, and anything that will be called from a hundred places.

And because both compile to the same bytecode, a graph calling into a script and a script calling into a graph are both just function calls. That is the part I am most pleased with, the choice is per system and not per project.

---

Next Wednesday: the editor is scriptable in the same language as your game. Custom windows, custom inspectors, your own menu entries, settings pages that ship in the build, and the generic node-graph framework you can use to build your own visual tools.

*Comments and questions welcome ;)*
