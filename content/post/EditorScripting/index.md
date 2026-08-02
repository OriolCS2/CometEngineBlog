---
title: An Editor That Edits Itself
description: Custom windows, custom inspectors, scriptable settings pages, and building your own node-graph tools
date: 2027-02-10 03:00:00+0000
image: cover.png
categories:
    - Editor
tags:
    - AngelScript
    - Editor
    - Node Graphs
---

Sooner or later everyone using an engine needs a tool that the person who wrote the engine did not think of. That has happened to me plenty of times with Comet, and my answer was to make the editor scriptable in **the same language you write your game in**. It is not a plugin API and it is not a separate SDK. It is AngelScript, in your project, next to your gameplay code.

![What you can reach from an editor script](surface.png)

## A window in about thirty lines

![An editor window script](window-code.png)

You inherit from `EditorWindow`, implement its draw callback and use the `CometEditor::GUI` API. Buttons, fields, trees, tables, tab bars, all of it is there. Add `[MainMenuItemWindow]` and the window appears in the menu bar. It docks like any built-in panel, because as far as the editor is concerned it is one.

![The same script open in the editor](window-script.png)

That is the Code workspace, and I want to point at the attribute on the class, because `[MainMenuItemWindow]` is the whole registration. There is no manifest, no install step and no restart. You save the file and the entry is in the menu bar.

The `GUI` namespace has grown a lot over the last few releases: `Selectable` with a custom size, tooltips with flags, `SeparatorText`, clipboard access, `CalcTextSize`, `SetNextItemFullWidth`, and a `ListClipper` for when your window has to list ten thousand things and you would rather it did not draw all of them.

## Replacing an inspector

`[CustomInspector]` on a class deriving from `EditorBehaviour` takes over the inspector for a type completely, and then you draw whatever you want.

This is the escape hatch for the one real limitation of [attribute-driven inspectors]({{< ref "/post/WritingScripts" >}}). Attributes describe fields, and sometimes what you need is not a field. It is a picture, or a curve editor, or a grid of buttons, or a preview of the thing the data describes.

The rule I hold myself to is to reach for attributes first, and only write a custom inspector when the default arrangement is actively misleading. A custom inspector is code I have to maintain forever, and a `[TreeNode]` is not.

## Settings pages that ship

![Settings](settings-page.png)

2.8 added something I use constantly. You inherit from `ProjectSetting` and you get a real page in Project Settings, and **that data is packed into the game build**, so you can read it at runtime.

That closes a gap I used to fill with a hand-rolled config asset in every project. Difficulty curves, feature flags, the URL your game talks to. They belong in Project Settings, they should be editable by someone who is not you, and they should ship with the game.

`EditorPreference` is the same idea for editor-only settings, per machine. Putting either of them in an `Editor/` folder makes them editor-only automatically, so they never reach a build.

Underneath both there is `EditorPrefs`, a plain key/value store with a **global** scope shared across every project on the machine and a **local** scope stored in the project. Global for how I like my tools, local for how this project is set up.

## Driving the editor from script

Two namespaces let an editor script act instead of only drawing.

`CometEditor::Selection` reads and writes what is selected, both scene entities and Project panel resources, and it exposes a change event so a tool can react to whatever you are looking at.

`CometEditor::EditorApplication` controls play mode: enter, exit, pause and step, plus events for entering and leaving. That is what lets a tool run a scene, sample something and then stop.

With those two you can write things like "select every enemy with no patrol path", or "step one frame and dump the physics state".

## Build your own visual tool

![GraphNode](graphnode.png)

I think this is the most powerful thing in the post and the one people miss.

Comet's node-graph editing is not hard-wired to [Visual Scripting]({{< ref "/post/VisualScripting1" >}}). Underneath there is a generic **GraphNode** system. You inherit `GraphNode` for the graph and its execution, you inherit `Node` for each kind of box, and you attach a `GraphUpdater` behaviour to run it.

Visual Scripting is one consumer of that system, and nothing stops you writing another.

Dialogue trees are the obvious one. A `Line` node, a `Choice` node, a `SetFlag` node, and now your writer edits conversations in a graph instead of in a spreadsheet. Quest chains, ability trees, boss phase machines and procedural generation rules are all the same shape.

What I like about this is that the tool ends up being yours. It speaks in your game's nouns instead of the engine's, and 2.2 added variables with proper inputs and outputs to `GraphNode`, so those graphs can pass data around and not only sequence things.

## Where this leads

The MCP server, which gets [its own post in July]({{< ref "/post/McpServer" >}}), is built on the same principle from the other side. The editor's operations are exposed as a tool layer that does not care who is calling it, so an AI assistant, an external client and an in-editor panel all drive the same commands.

Editor scripting is that same idea applied to the person using the engine. The tools you build are made of the same parts as the built-in ones.

---

Next Wednesday: the escape hatch below scripting entirely. Loading a real `.dll`, `.so` or `.dylib` from AngelScript, a portable FFI across x86-64 and arm64, and how the build knows which binaries to ship.

*Comments and questions welcome ;)*
