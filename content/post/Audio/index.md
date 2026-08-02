---
title: Making Noise
description: SoLoud, the redesigned mixer, effects, and 3D audio in a game with no third dimension
date: 2026-12-16 03:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Audio
    - SoLoud
    - Editor
---

Audio is usually the last system anyone adds, and then you wish you had added it first. A game with no sound feels broken in a way that is hard to explain and very easy to notice. You press a button and nothing answers you, and the whole thing feels like a diagram of a game instead of a game.

Comet's audio runs on **SoLoud**, a small C++ audio library with a permissive licence. It does its job and it does not try to be a whole ecosystem, which is what I wanted.

## Playing something

You need two things. An **Audio Source** on the entity making the noise, and an **Audio Listener** on the entity hearing it, which is normally the camera.

Point the source at an Audio Clip, tick `Play On Awake`, and that is it. From script you have `Play()`, `Stop()`, `Pause()`, and `PlayOneShot` for sounds that should overlap instead of interrupting each other.

That last one matters more than it looks. A footstep that cuts off the previous footstep sounds wrong straight away, and changing one call fixes it.

## The mixer

![The Audio Mixer panel](mixer.png)

The Audio Mixer got a visual rework in 2.8. Only the visuals, the routing underneath is the same as before. It is the panel that turns a pile of individual sounds into something you can control.

![How sources route into groups](routing.png)

Sources route into **groups**, groups route into other groups, and everything ends at **Master**. If you put every footstep, impact and explosion into an `SFX` group, one slider controls all of them. That is what groups are for.

The part that goes beyond volume sliders is **layouts**. A layout is a saved snapshot of the whole mixer state, and you can switch between layouts at runtime. I use an `Underwater` layout that ducks the highs and pushes the reverb, a `Paused` layout that drops everything except the UI, and a `LowHealth` layout that thins out the music. Switching layout is one call, and it is how the audio reacts to what is happening in the game instead of just playing.

## Effects

Every group can have effects: echo, reverb, filters and a few others.

Reverb is the one that earns its cost. The same footstep clip played dry, and then played through a reverb with a large room size, sounds like two completely different places. You do not need a separate set of cave footsteps, you need one footstep and a room.

In 2.0 I tidied up the effect parameters quite a lot. `AudioEffectEcho` gained a `filter`, and reverb gained `width`, `damp`, `roomSize`, `freeze` and `wetMix`. I also removed a long list of parameters that did nothing useful, together with a few effects (Chorus, Compressor, Fader) that were not carrying their weight. I would rather have fewer knobs and have all of them do something.

## 3D audio in 2D

![A 3D audio source in a 2D scene](threed.png)

This sounds like a contradiction, and it is one of the features that gives back the most for the least work.

Mark an Audio Source as 3D and it gets a world position and a rolloff curve. The listener has a world position too. Comet works out the rest, taking the stereo panning from the horizontal offset and the volume from the distance.

A machine humming off to the left of the screen sounds like it is off to the left. Walk towards it and it gets louder. You do not write any of that.

The rolloff curve is worth tuning instead of leaving it at the default. Linear rolloff is rarely right, because real sound falls off fast when you are close and slowly when you are far away, and a curve shaped like that sounds much more natural.

Keep UI sounds and music in 2D. A menu click that pans because the camera happens to be off centre is a bug, and it is one that people find really disorienting.

## Import settings

![Per platform import settings for a clip](import.png)

Every clip has import settings, and they are set **per platform**, which matters more than it sounds.

**Preload** decodes the clip up front and keeps it in memory. That is right for short, frequent sounds: footsteps, hits, UI. No disk access at play time and no decode hitch.

**Stream** decodes as it plays. That is right for long, infrequent things: music, ambience, long dialogue.

Getting these the wrong way round is a classic. Preloading a five minute music track means decoding several minutes of audio into RAM before the main menu appears. Streaming a footstep means a disk seek every time the player takes a step.

The per platform part is the important one. A desktop build can afford to preload generously. A mobile or web build cannot. Being able to say "preload on desktop, stream on Android" for the same clip, without keeping two versions of the asset, is exactly the kind of thing that stops platform work from being miserable.

## Small things that took real time

The audio work that took me longest was not the interesting part.

**A volume slider on the clip preview.** When you click an audio asset, the inspector plays it. Until 2.0 it always played at full volume. Adding the slider was trivial, but it took me an embarrassingly long time to notice that it needed one, because my system volume was always already set for whatever I was working on.

**Removing a mixer group from a source.** You could assign a group. You could not remove it again. That shipped like that, and the fix is in the 2.0 fixed list under a very short line.

**Making the panel stop looking like a debug tool.** The 2.8 rework changed no functionality at all. It only stopped looking like something I had built for myself, which is real work even though it gives you nothing to put in a feature list.

## What Comet does not have

There is no procedural audio and no DSP graph. Effects are a fixed set on groups, not a node graph you compose.

There is no occlusion. A sound behind a wall is as loud as a sound in front of it. You can fake it by driving the volume from a raycast, but that is a script you write, not a feature you enable.

There is no built in music sequencing or adaptive layering. Layouts get you a good part of the way towards the same feeling, but crossfading stems in time with a beat is something you build yourself.

---

Next Wednesday, the week before Christmas, I will write the most visual post of the year: text. Font rendering, kerning, bitmap fonts and BBCode, with colours, waves, shakes, rainbows and a custom `[snow]` tag written for the occasion.

*Comments and questions welcome ;)*
