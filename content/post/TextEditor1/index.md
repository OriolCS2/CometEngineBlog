---
title: "The Editor Has Its Own IDE (Part 1)"
description: I wrote a code editor inside a game engine. Here is why, and what it does all day
date: 2027-01-13 03:00:00+0000
image: cover.png
categories:
    - Scripting
tags:
    - Editor
    - AngelScript
---

Comet 2.6 added a text editor inside the engine editor. 2.7 added breakpoints and debugging to it. 2.7.1 and 2.7.2 are almost entirely lists of fixes to it.

Writing a code editor is a strange thing to do when you are one person making a game engine, and the first question is a fair one. Why not just use VS Code?

## Why not just use VS Code

![The editor open next to the scene view](why.png)

You still can. VS Code is a supported external editor, and so is Antigravity since 2.2. If that is what you prefer, nothing here takes it away from you.

But there were reasons to write my own anyway.

Nobody was going to write an AngelScript language server. Autocomplete, go to definition and find references all come from language servers in a modern editor, and language servers exist for languages with large communities. AngelScript embedded in one specific engine, with that engine's specific API surface and its generated bindings, is never going to get one. If an editor is going to understand Comet scripts, I have to be the one who writes it.

Then there is alt-tab. Going out to another window to change one line and coming back happens something like thirty times an hour, and what it costs me is not really the seconds, it is the concentration.

## What is in it

![The main features of the text editor](features.png)

**Syntax highlighting** for AngelScript, HLSL, JSON, XML and Markdown. From 2.8, Markdown files also render a live preview behind a button in the top right, which is how I read the engine's own docs without leaving the editor.

**Autocomplete** that knows your project, not just the words already on screen. It knows your classes, your methods, their parameters and the whole engine API, and it knows which `using namespace` directives are in scope. That last part matters a lot, because when it gets namespaces wrong the suggestions are useless.

**A command palette** for jumping to files and running actions without leaving the keyboard.

**Find in Project**, which searches every script.

**Ctrl+click** on a symbol to jump to it.

**Hover tooltips** showing signatures and types.

**Scope lines**, the vertical guides that connect the braces of a block. They sound like a very small thing and they are the feature I would miss most.

**Auto-formatting** that understands AngelScript, including the annoying parts of it. Preprocessor directives like `#if` and `#else` used to make everything after them over-indent, and I fixed that in 2.8. A comment on the line above a `{` used to indent the brace wrongly, fixed in 2.7.2.

![A script shown in the inspector](script.png)

## The Code Workspace

![The workspace options](workspace.png)

The editor is a panel, so it docks like any other panel. But writing code inside a game editor layout is cramped. You want the file in front of you, and at that moment you do not need the Hierarchy.

So there is a **Code Workspace**, a layout that gives most of the window to the editor. You toggle into it, write, and toggle out.

There is also a preference to make the text editor always live in its own OS window. That is how I use it, the engine on one monitor and the code on the other, both fullscreen. No alt-tab, because they really are two windows.

## The parts that were harder than expected

Highlighting was easy and understanding was not. Colouring keywords is a lexer and an afternoon of work. Knowing that `player` on line 40 is a local of type `Entity` declared on line 12, inside a method whose class is in a namespace, needs a symbol table, scope tracking and incremental re-parsing on every keystroke.

The bug list tells this better than I can. Local variables not appearing in autocomplete when the method had more than one parameter. Member variables coloured wrongly when their class was inside a namespace. Classes declared `mixin` not being recognised as classes. Autocomplete failing when calling a method on a parent class.

Fonts matter too. The whole "Changed" section of 2.7.1 is one line, the text editor font was changed to a more readable one. That was a real complaint, from me, about my own editor, after a week of using it properly.

## Was it worth it?

For the engine, yes, although not for the reason I expected.

I built it for convenience. What it actually gave me was the ability to add tooling that understands Comet specifically. Once the editor has a symbol model of your project, things like Find All References, Rename Symbol and breakpoints become possible.

That is what next week is about.

---

Next Wednesday: the two features that made the built-in editor stop being a toy. Find All References, Rename Symbol across a whole project, and a real AngelScript debugger with breakpoints, stepping and variable inspection, including in a game running as a standalone build.

*Comments and questions welcome ;)*
