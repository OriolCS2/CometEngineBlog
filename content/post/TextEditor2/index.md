---
title: "The Editor Has Its Own IDE (Part 2): Refactor and Debug"
description: Rename Symbol, Find All References, and breakpoints in a game that is already running
date: 2027-01-20 18:00:00+0000
image: cover.png
categories:
    - Scripting
tags:
    - Debugging
    - AngelScript
    - Editor
---

[Last Wednesday]({{< ref "/post/TextEditor1" >}}) ended with a claim. I did not build an editor inside the engine for convenience in the end, I built it because it makes tooling possible that an external editor could never give me. These are the three features that back that up.

## Find All References

![The refactoring tools](refactor.png)

Put the cursor on a symbol, press `Ctrl+Shift+F12` or right click it, and a **References** sidebar lists every usage in the project.

These are usages and not text matches. A local variable called `speed` in one method is not the same symbol as a member called `speed` in another class, and that difference matters a lot when what you are really asking is "what would break if I changed this?".

That question is the reason the feature exists. Before it, for any method used in a lot of places, my answer was "I do not know", and what happened in practice is that I did not change things I should have changed.

## Rename Symbol

The natural next step, and the one that changed how I work.

Rename a symbol and it is renamed everywhere it really is that symbol. It is not a find and replace. A `Fire()` method on your weapon class is not touched when you rename a `Fire()` on your particle emitter, and a string containing the word "fire" is left alone.

Bad names survive because renaming them is risky. Once the risk is gone, names keep getting better instead of staying frozen at whatever I thought on the first day. Half the identifiers in the engine's AngelScript layer are better than they were, purely because fixing one stopped being a small project.

## A real debugger

![The debugger stopped on a breakpoint](debugger.png)

2.7 added breakpoints and debugging for AngelScript in the native editor.

Click the gutter and you get a breakpoint. The game halts on that line. From there you can step over, step into, step out, look at the call stack, inspect locals and members, and hover a variable while paused to see its value.

## Debugging a build

This is the part that makes the whole thing more than a convenience.

The debugger attaches over loopback TCP to a **standalone development build**. Not the game running inside the editor, the actual exported game running as its own process.

You export with `Development Build` enabled, run it, attach, and set breakpoints in a shipped binary. The [profiler]({{< ref "/post/Profiler" >}}) attaches over the same channel at the same time.

"It only happens in the build" is one of the worst sentences in game development, because historically it meant my tools stopped existing exactly when I needed them. Everything works in the editor, in the build it does not, and all I had was a log file. Remote attach turns that into a normal debugging session.

## What I learned building it

A debugger is mostly bookkeeping. AngelScript gives you the hooks, line callbacks, stack inspection and variable enumeration. The work is everything around them: mapping bytecode positions back to source lines through a file that may have been edited since, keeping the UI responsive while the game is halted, and deciding what happens when a breakpoint is hit inside a callback the engine is iterating over.

Halting a game is a strange state to be in. The game is paused, but the editor has to keep drawing, keep responding and keep letting you inspect things. Comet's editor and game share a process, so pausing the game but not the editor took care that a debugger in a separate process would not need.

The bugs were all edge cases. Every debugger bug I fixed was some variation of "this works unless the thing is inside a namespace, or inside an array, or inherited from a parent, or declared as mixin". The happy path was working within a week, and the long tail took months.

![The console](console.png)

## What is missing

![The features that are not there yet](missing.png)

There are no conditional breakpoints. Break when `health < 0` would save me real time and it is not there yet.

There are no watch expressions. You can inspect the variables in scope, but you cannot type an arbitrary expression and evaluate it.

There is no edit and continue. If you change a script while paused you have to stop, recompile and replay. Given how fast the [compile loop]({{< ref "/post/WritingScripts" >}}) is, this bothers me less than it might.

And there is no visual script debugging from here. Debugging a graph is its own thing, and it is [two weeks away]({{< ref "/post/VisualScripting1" >}}).

---

Next Wednesday: the other way to write gameplay logic in Comet. Visual Scripting, where the node graph compiles directly into AngelScript bytecode and runs at exactly the speed of hand-written code, and where every node is generated from the engine's own API instead of written by hand.

*Comments and questions welcome ;)*
