---
title: "Animation (Part 2): State Machines"
description: States, parameters, exit time, controller overrides and root motion
date: 2027-03-31 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Animation
    - Editor
---

[Last Wednesday]({{< ref "/post/Animation1" >}}) produced clips. This week decides which one plays, and that turns out to be a design problem rather than an animation one.

## Why not just if-statements

You can absolutely write `if (grounded && speed > 0) PlayRun();` and for a character with three animations it works fine.

It stops working at about six. The failure is not that the code gets long — it is that transitions become implicit. "Can I go from attack to jump?" is answered by whatever order your `if`s happen to be in, and nobody, including you in three months, can read the file and say what the legal transitions are.

An Animator makes the transitions the *data*. What can follow what is a thing you can look at.

![The parts](machine.png)

## Parameters are the seam

The Animator does not read your game. It reads **parameters** — floats, ints, bools and triggers — that your gameplay code sets.

That indirection is the whole design. Your movement script says `animator.SetFloat("speed", velocity.magnitude)` and knows nothing about which animation exists. The Animator decides that run starts at 0.1 and sprint at 6.0, and you can retune those without touching gameplay code.

Triggers are the special case: a bool that consumes itself when a transition uses it. 1.0 added `ResetTrigger()` by name or parameter ID, because a trigger set on a frame where no transition could consume it stays armed and fires later, which produces a genuinely confusing bug — an attack animation that plays half a second after the button, once.

## Exit time

![Exit time](exittime.png)

This is the setting that separates animation that feels good from animation that feels broken, and it is one checkbox.

Without exit time, a transition fires the instant its conditions are true. Press attack, then release the direction key on frame 2, and the attack animation cuts to idle two frames in. The swing never happens. The player pressed attack and nothing visible occurred.

Exit time says: this state must play for at least *this* fraction before it is allowed to leave. The swing completes. The input is still responsive because the *next* state is already queued; it just does not interrupt the frames that carry the meaning.

2.4 also changed the Animator to evaluate all conditions **immediately upon entering a new state** rather than waiting for the next frame. That removed a one-frame lag on chained transitions, which is exactly the kind of thing you feel and cannot name.

## Sub-machines

When a character has fifteen states the graph becomes unreadable, and the answer is grouping.

A sub-machine is a state that contains its own state machine. `Airborne` holds jump, fall and land; from outside it is one box with transitions in and out. The graph reads as five states instead of fifteen.

2.4 added a quality-of-life fix that matters more than it sounds: the Animator panel now **automatically switches to the visible state machine** when you select an entity whose current state lives in a different one. Before that, selecting a character mid-jump showed you a graph that did not contain the state it was in.

Also from 2.3: copy, paste and duplicate for states and machines (`Ctrl+C`, `Ctrl+V`, `Ctrl+D`), renaming parameters with the rename shortcut, and dragging a transition to empty background to create its target state on the spot. Building a graph stopped being a right-click marathon.

The state inspector also shows **a table of every transition arriving at that state**, which is the question you actually ask when debugging — not "where does this go" but "what gets me here".

## One machine, two characters

![Overrides](overrides.png)

An **AnimatorControllerOverride** takes an existing controller and swaps only the animation clips.

Your second character gets the same states, the same parameters, the same transitions and its own art. Fix the logic and both inherit the fix. This is the same idea as [RuleOverrideTile]({{< ref "/post/Tilemaps2" >}}) and [InstanciableEntity variants]({{< ref "/post/InstanciableEntities2" >}}) — the engine keeps arriving at "inherit the structure, override the leaves", which I think is the right instinct.

## Root motion

![Root motion](rootmotion.png)

With root motion off, the animation plays in place and your code moves the entity. Full control, and the classic risk of foot sliding when the movement speed and the animation disagree.

With it on, the animation's own motion drives the entity. The art and the movement match exactly, and you give up some directness.

It is per-Animator via `applyRootMotion`. 2.7 fixed the Animation Timeline not honouring that flag when previewing — so a root-motion clip previewed as if it were in place, which made it impossible to author against.

## The failure mode nobody warns you about

Not a bug — a design trap.

State machines grow. Every new ability adds a state and, worse, adds transitions *from* several existing states. The count of things you can create grows linearly; the count of transitions grows much faster.

The point at which to stop is when you find yourself adding "can interrupt" transitions from everything to everything. At that stage the machine is no longer describing legal sequences, and you would be better served by a small amount of code deciding intent and a much simpler machine playing it.

I do not have a clean rule for where that line is. I do know that every animator graph I have regretted was one I kept adding to past the point where I could see it all at once.

---

Next Wednesday: things falling over. Physics on Box2D 3 — bodies, colliders, joints, the collision matrix, raycasts, and three bugs that each taught me something about how the pieces fit.

*Comments and questions welcome ;)*
