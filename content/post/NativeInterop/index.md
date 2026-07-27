---
title: Talking to Native Code
description: Loading a real dynamic library from AngelScript, and shipping only the binary each platform needs
date: 2027-02-17 18:00:00+0000
image: cover.png
categories:
    - Scripting
tags:
    - Native Interop
    - C++
    - Android
---

Every scripting language ends up needing an escape hatch. Somebody wants to use a physics solver you did not write, or a compression library, or a piece of hardware that only ships a C SDK, and no amount of engine API is going to replace it.

2.8 added the hatch. AngelScript can load and call into a native `.dll`, `.so` or `.dylib`.

## The four pieces

![The API](api.png)

`NativeLibrary` loads the library. `NativeFunction` resolves an export and pins down its call signature. `NativeCall` invokes it. `NativeBuffer` and the `Marshal` helpers move raw bytes across the boundary without leaking or corrupting them.

![An example](example.png)

That is genuinely the whole surface. What I wanted was for calling a C function to be about as much ceremony as calling an engine function, and mostly it is. The one extra thing you have to do is declare the signature, because a `.dll` export is just an address and nothing in the file tells you it takes two floats.

## A portable FFI, not a per-platform hack

![The FFI](ffi.png)

The interesting engineering is underneath. Calling a C function whose signature you only learn at runtime means building a call frame by hand, and the rules for that change with the architecture: which registers hold which argument, how floats are passed, what the stack alignment has to be, what happens to structs.

Comet ships a portable foreign function interface covering **x86-64 and arm64** on Windows, Linux and Android. That combination is what makes it usable on mobile, and mobile is where a native library is most often the reason you needed one in the first place.

It is also the part I would treat with respect. A wrong signature does not throw a nice error. It corrupts a stack frame and crashes somewhere unrelated, which is exactly the class of bug the [migration away from C#]({{< ref "/post/GoodbyeCSharp1" >}}) was supposed to reduce. The FFI is a sharp tool by nature.

## Plugins are assets

![Plugins](plugins.png)

This is the part that keeps native libraries from turning into a build system nightmare.

Drop a native library into the project and Comet detects it as a **Plugin asset**. It gets an inspector with platform and architecture targeting, and those settings are inferred from folder conventions, so if you put an `arm64-v8a` build in a folder named for it the importer works out what it is.

Then at build time, only the plugins matching the destination platform and architecture are copied into the export. An Android arm64 build does not carry your Windows x64 DLL, and a web build carries none of them because it cannot load any.

Without that, every project using a native library grows a copy step that somebody has to maintain by hand, and every platform bug turns into "did the right binary get copied?". With it, the answer is data you can see in an inspector.

## When to reach for this

Honestly, rarely.

Good reasons: a mature library that would take months to reimplement, like a solver, a codec or a format parser. Vendor SDKs for hardware. Code that is genuinely hot, where the scripting layer is measurably the bottleneck and you have profiled it instead of assuming it.

Bad reasons: "native is faster" as a general belief. It is faster per instruction, but each call across the boundary costs something, so a native function called in a tight loop can easily end up slower overall than the AngelScript one it replaced. Measure the whole loop and not the function.

The one I see catching people is using this to avoid learning the engine API. If Comet already does the thing, the built-in path is the one that works on the web, gets debugged with the [script debugger]({{< ref "/post/TextEditor2" >}}), and does not need a separate build per architecture.

## The web

This comes up often enough that I want to say it plainly: there is no native interop on the web. A browser cannot load a `.dll`. If your game depends on a native library, it does not have a web build.

That is a real architectural decision and not an oversight, and it is the main reason I would think hard before making a native library load-bearing. Comet's whole platform story rests on the same project running everywhere, and a native plugin is the one feature that can break that promise.

If you need the library and the web as well, the answer is to compile it to WebAssembly and use it through the web build's own mechanisms. That is a different project, and one Comet does not do for you.

---

Next Wednesday: keeping what the player did. `SaveData` as a typed key/value store, binary saves with atomic writes, JSON when a human should be able to read it, and the compression, encryption and hashing that sit alongside them.

*Comments and questions welcome ;)*
