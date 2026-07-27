---
title: Saving the Game
description: Three layers for three jobs, and why play mode reloads your save file every time
date: 2027-02-24 18:00:00+0000
image: cover.png
categories:
    - Systems
tags:
    - Save Data
    - Serialization
---

Saving is one of those systems that looks like an afternoon and turns into a fortnight. The writing part is easy. The parts that are not: where the file goes on four different platforms, what happens when the write is interrupted, and what your save file does when you ship an update that changes the data.

Comet gives you three layers. The skill is picking the right one.

![The three layers](layers.png)

## SaveData, for the boring 80%

![SaveData](savedata.png)

`CometEngine::SaveData` is a typed key/value store. `SetInt`, `GetFloat`, `SetString`, and typed accessors for every scripting primitive plus `Vector2`/`Vector3`, their integer variants, `Quaternion`, `Color` and `Rectf`. Scalar getters take a default, so reading a key that does not exist yet is not an error — it is the first run.

It writes JSON into the platform's persistent data directory, and *that* is most of the value. You do not think about where a save file lives on Android versus Windows versus the web; the engine knows.

Changes are held in memory and flushed on `Save()`, automatically when the game quits, and in the editor when you leave play mode.

**And here is the rule I like most.** Entering play mode **reloads the saved file**, so every play session starts exactly like a fresh game launch.

That sounds like a small detail. It removes an entire category of bug — the one where your save logic appears to work for an hour because the values are still sitting in memory from the last time you pressed play, and then fails the first time a real player launches the game cold. If it works in play mode in Comet, it worked from disk.

For settings, progress, unlocks, best times and "has the player seen the tutorial", this layer is the whole answer.

## Binary, for structured state

When you are saving an inventory, a world, or four hundred entities' worth of state, key/value stops being the right shape.

`BinarySave` and `BinaryLoad` give you typed `Set`/`Get` for all the primitives, strings and byte buffers, plus the engine math types. You write fields in order, you read them back in order.

The property that matters: **`Save` is atomic.** It writes to a temporary file and replaces the original only once the write succeeded. An interrupted save — power loss, a crash, a player force-quitting on mobile — leaves the previous save intact rather than a half-written file.

I care about this specifically because the [InstanciableEntity asset save]({{< ref "/post/InstanciableEntities2" >}}) did *not* do it until 2.8, and I destroyed one of my own assets finding out. Doing it for player saves from the start was the one lesson I actually applied in advance.

## JSON, for when a human should read it

`CometEngine::Json::JsonObject` handles the cases where the file is not really a save — it is data. Level definitions, balance tables, exported debug state. 2.2 widened it to cover `Vector2i`, `Vector3i`, `Color` and friends, so engine types round-trip without hand-marshalling.

Rule of thumb: if you would ever want to open it in a text editor, or diff it in git, use JSON. If it is only ever read by the game, use binary and get the size and speed.

## Compression, encryption, and being honest about it

![Protection](protect.png)

Three namespaces sit alongside the save layers.

`Compression` does ZLIB and GZIP at four levels, plus Base64. Worth it on large binary saves; pointless on a settings file.

`Encryption` does authenticated **AES-256-GCM**, with the key derived by PBKDF2-HMAC-SHA256 over a random salt. It has `Encrypt`, `Decrypt`, and — the one that matters — `TryDecrypt` with an explicit success flag, so you can tell a genuinely empty plaintext apart from an authentication failure. That distinction is the difference between "the save is empty" and "the save has been tampered with", and conflating them is a real bug.

`Crypto` gives you `Sha256`, `HmacSha256` and `SecureRandomBytes` from the OS entropy source.

Now the honest part. **Encrypting a local save file does not stop a determined player**, and it never will — the key has to be in the binary they are running. What it does is stop *casual* editing: the player who opens the JSON, changes `coins` to 999999, and then reports your economy as broken.

If the integrity of a value actually matters — a leaderboard, anything competitive — it has to be validated on a server. That is not a Comet limitation, it is arithmetic.

## Versioning, which nobody does until it hurts

Comet does not solve this for you and I want to be clear about it.

The moment you ship an update that changes your save structure, every existing save is from the old version. `SaveData` degrades gracefully because missing keys return their default. Binary saves do not — read the fields in the wrong order and you get garbage.

So: write a version number as the first field of every binary save, from day one, before you need it. It costs four bytes and it is the difference between migrating old saves and abandoning them.

---

Next Wednesday we start a month on building worlds. Tilemaps: grids, palettes, the painting tools, layering, and the collider merging that stops your character catching on invisible seams.

*Comments and questions welcome ;)*
