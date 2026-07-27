---
title: "Animation (Part 1): The Timeline"
description: Recording, keyframes, events that call your code, and animating your own script fields
date: 2027-03-24 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Animation
    - Editor
---

Animation in a 2D engine is two different problems wearing one name. This week is the first: producing a clip. Next week is deciding which clip should be playing.

## Not just sprite swapping

![What you can animate](what.png)

The obvious thing to animate is which sprite is showing, and Comet does that. But the Animation Timeline animates **properties**, and the list is longer than people expect: transform position, rotation and scale; renderer colour and flip; Canvas Group alpha; Image properties; font size.

And — the one that changes how you use it — **fields on your own scripts**. A `bool`, `int`, `float`, `Vector2`, `Vector3` or `Color` declared in an AngelScript behaviour can be keyframed like anything else.

That turns the animation system into a general-purpose "change these values over time" tool. A screen shake is an animation. A damage flash is an animation. A door that unlocks is an animation on a script's `isLocked` field. You do not write a coroutine for any of them.

## Recording

![A sprite animation in the timeline](timeline-real.png)

That is a seven-frame run cycle. One property row — `Entity.Sprite Renderer.Sprite` — with a keyframe every ten frames, and the track draws the actual sprite at each key so you can read the cycle without playing it.

![The timeline](timeline.png)

Arm **Record** and every change you make writes a keyframe at the current frame. Move the entity, get a position key. Change a colour, get a colour key.

The one thing to know: recording captures *what you changed*, not everything. 0.9.2 fixed a bug where recording a frame and modifying one axis of a multi-value field added keys for every axis; now it keys only the value you actually touched. That matters because extra keys are not harmless — they pin values you wanted free to be interpolated by other clips.

Between keys, the curve is yours to shape. The default is smooth, which is right for movement and wrong for anything that should snap.

## Events

An **animation event** calls a method at a specific frame.

This is how animation and gameplay meet, and getting it right is the difference between a game that feels responsive and one that does not. The footstep sound plays on the frame the foot lands, not on a timer. The attack's hitbox turns on at frame 4 and off at frame 7, defined by the artist on the animation, not by a programmer guessing at durations.

2.0 added a `+` button that adds an event at the currently selected frame, and made right-clicking an event show options for *that* event rather than only "remove all events at this frame" — which was, as written, a small trap.

## Editing at speed

2.3 rewrote multi-selection, and it is the change that made the timeline pleasant.

Click selects a keyframe or event. Ctrl-click toggles individual ones. Clicking the global row selects everything at that frame. Selected keys and events **drag together as a group**, preserving their relative spacing, with ghost previews showing where they will land. `Supr` deletes the whole selection.

Retiming an animation — "this whole sequence should take 20% longer" — went from a per-key chore to a single drag.

Sub-resources can also be assigned to several keyframes at once rather than one at a time, which matters when you are pointing thirty frames at thirty sprites.

## When you do not need any of this

![Lightweight options](lightweight.png)

Most animated things in a game are not state machines. They are a loop.

2.8 added **AnimatedSprite** — a behaviour holding a reorderable list of sprites, played frame by frame. A coin spinning, a torch flickering, a pickup bobbing. No Animation asset, no controller, no timeline.

And **SingleAnimation**, which plays one `Animation` asset without needing an `AnimatorController` at all.

Both exist because the honest answer to "how do I animate this coin" used to be "create an Animation, create an AnimatorController, assign it, add one state", and three of those four steps were ceremony.

Reach for the Animator when you have *states and decisions*. That is next week.

## Two bugs, one lesson

**Single-frame animations looped forever** instead of finishing, right up to 2.2.1. The completion check assumed at least two frames existed, so a one-frame "animation" — which people genuinely make, as a way of setting a pose — never reported that it had ended, and anything waiting on it waited forever.

**Previewing left values applied.** Selecting an entity with an Animator or Single Animation and scrubbing the timeline previewed the animation, which is correct — but deselecting left those previewed values permanently applied to the scene entity. You would preview a death animation, click away, and your character was now lying down in the saved scene.

Both are the same lesson, and it is the one this whole blog keeps arriving at: the edge case is not the bug, *the edge case is where the assumption you never wrote down turns out to be wrong.* "Animations have more than one frame" and "previewing is temporary" were both assumptions nobody had stated.

---

Next Wednesday: which clip should be playing. The Animator — states, sub-machines, parameters driven from gameplay, transitions with conditions and exit time, controller overrides for a second character, and root motion.

*Comments and questions welcome ;)*
