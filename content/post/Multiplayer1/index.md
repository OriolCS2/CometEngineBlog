---
title: "Multiplayer (Part 1): The Stack Underneath"
description: One peer abstraction over three transports, and the web constraint that shaped all of it
date: 2027-06-30 03:00:00+0000
image: cover.png
categories:
    - Networking
tags:
    - Networking
    - Web
---

Comet 1.x had sockets. `SocketClient`, `SocketServer`, some events. That was enough to send bytes between two machines, and not enough to build a game on.

In 2.8 I removed them and built a real stack.

![The layers of the networking stack](layers.png)

## The layers

At the bottom there is low level I/O: `NetSocket`, `IP`, `StreamPeer`, `PacketPeer`. Above that, `TCPServer` and `UDPServer`. Above those, one abstraction called **`MultiplayerPeer`**, and three transports that implement it.

I picked that shape for the same reason [rendering has IRenderHardware]({{< ref "/post/TwoBackends" >}}). The layer above should not have to know which transport it got.

## Three transports

![Choosing between ENet, WebSocket and WebRTC](transports.png)

**ENet** is the answer on desktop and mobile. It is UDP with optional reliability per message, client/server or mesh topologies, four compression schemes (FastLZ, zlib, Zstd, Range Coder) and DTLS encryption. If the platform allows it and the game is real-time, this is the transport I would use.

**WebSocket** (RFC 6455, `ws://` and `wss://`) works everywhere, browsers included. It runs over TCP, so you get head of line blocking: one delayed packet stalls everything queued behind it. That is fine for turn-based games and for lobbies, and it is a real problem for a shooter.

**WebRTC** is how you get UDP-like behaviour inside a browser, with native backends (`EMWSPeer`, `RTCPeerConnection`) for web builds. It needs more setup because it needs signalling, and it is the only real-time option on the web.

## Why the abstraction exists

All of this exists because a browser cannot open a raw socket.

If Comet only targeted desktop I would have used ENet on its own and the abstraction would have been a waste of time. But [web is a first-class target]({{< ref "/post/WebExport" >}}), so the same game code has to run over a transport that behaves quite differently. The only sane way I found to do that was to define the interface first and then implement it three times.

It is the same pattern as the [render backends]({{< ref "/post/TwoBackends" >}}) and the [shader pipeline]({{< ref "/post/HelloHLSL" >}}). When I support one awkward platform properly, it forces an abstraction that ends up making everything else cleaner. The web has probably been the most useful constraint in the history of this engine.

## What it looks like in a script

![A networked Behaviour with attributes on three of its fields](script.png)

There is no networking boilerplate in that file. It is an ordinary `CometBehaviour` with attributes on three of its fields. The layer above, which is [next Wednesday's post]({{< ref "/post/Multiplayer2" >}}), is what turns those attributes into packets.

## The rest of the networking module

![TLS, HTTP, UPnP and the crypto primitives](extras.png)

**TLS and DTLS** through `StreamPeerTLS` and `PacketPeerDTLS`, backed by mbedTLS, with certificate and key handling.

**`HTTPClient`**, which uses the browser's native `fetch()` on web and TCP/TLS everywhere else. Leaderboards, analytics and remote content, with the same code path on every platform.

**UPnP NAT port mapping** through miniupnpc. That is what gives a player-hosted server a chance of being reachable without the host having to configure a router.

**Crypto primitives**: `Crypto`, `CryptoKey`, `X509Certificate` and `HMACContext`, plus `StreamPeerGZIP` and Unix domain sockets.

## A few things I learned

Pick the transport from the platforms you are actually going to ship to. If you need the web then you need WebSocket or WebRTC, and that decides more about your netcode than any preference of yours.

Reliable delivery is not free. ENet lets you choose per message, and it is tempting to make everything reliable because it is easier to reason about. Position updates should be unreliable. A dropped one is replaced by the next one anyway, and forcing a retransmit only makes things worse.

Encrypt from the start. If you add DTLS at the end you will find out that your packet sizes no longer fit.

Test with real latency. Everything works on loopback, so testing there does not tell you much.

## Would I build this again?

This is a lot of engine for a feature most 2D games never use, and I thought quite hard about whether it deserved a whole release.

What convinced me is that most of it is not optional. `HTTPClient` is used by games with no multiplayer at all. TLS is needed the moment you talk to any service. The remote content groups in the [content system]({{< ref "/post/Content2" >}}) are built on top of this. The three transports are only the visible part of something I needed anyway.

---

Next Wednesday I will write about the layer that makes all of this invisible: RPCs with authority and reliability, synchronizers that replicate entity state, spawners that deal with late joiners, and interest management so a peer only hears about what it can see.

*Comments and questions welcome ;)*
