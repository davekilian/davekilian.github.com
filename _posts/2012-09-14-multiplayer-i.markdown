---
layout: post
title: Multiplayer I &ndash; Introduction
author: Dave
draft: true
---

## TODO

We're currently working on a WebGL port of the nullZERO prototype from 
CS195-U. Could make sense to revamp the series to use the js port as the
example code.

---

Ever wanted to write a multiplayer video game?

Writing a multiplayer game that works well over a LAN isn't too hard. Packets
moving through a LAN tend to get from A to B quickly. It's
usually enough to replicate the state of your game world between two networked
machines.

Writing a multiplayer game that works well over the Internet is a different
story. Packet loss is common. Latency is high, and changes without warning.
Some players will inevitably tamper with the network to cheat. Dealing
with these conditions requires a bit more finesse than simply replicating 
state.

This series of blog posts is a nuts-and-bolts introduction to building 
an Internet-ready first-person shooter. The game will perform well over
broadband connections, and we'll take some steps to mitigate cheating.

These posts are based on what I learned while building online multiplayer 
for [nullZERO](http://fracture-studios.com/404). I drew heavily from
publishings about [Valve's Source Engine](http://source.valvesoftware.com/) 
and [id Software's QuakeWorld](http://en.wikipedia.org/wiki/QuakeWorld). 

## Audience

This series is aimed at readers who have experience developing video games.
If you can build a rudimentary FPS without relying heavily on sample code,
these posts are for you!

Readers should be able to read C++. The example code for each post, as well
as the inline code snippets, are written in C++. That said, you can apply
what you learn in any language you desire.

Game networking is shenanigans. You should be prepared to get stuck a couple 
times while following the tutorial. When you do, take a break and come back
later!

## Outside Reading

Before starting this tutorial series, I highly recommend you read Glenn 
Fiedler's 
[Networking for Game Programmers](http://gafferongames.com/networking-for-game-programmers) 
series. In particular, I won't be covering packet transport layers because
Glenn does it so well. He explains client/server gameplay at a high level;
this series covers the same in greater technical detail.

The following is a list of other documents I consulted when developing 
nullZERO. You may also find them useful:

* [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
* [Latency Compensating Methods in Client/Server In-Game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)
* [NPC Lag Compensation](https://developer.valvesoftware.com/wiki/NPC_Lag_Compensation)

## Client-Server in a Nutshell

TODO diagram

We'll build our game using a client-server architecture. I'll assume 
you're familiar with client/server systems in general. For games, a 
client-server system usually means:

* The server tracks the state of the game
* The server sends the state to the clients
* The clients locally render the server state to the player's screen
* The clients collect player input
* The clients send the player input to the server
* The server uses the input to affect the simulation

Posts further down the line will blur this division of labor, but this 
description will suffice for now.

## Terminology

Network programmers worry about time a lot. Each client has a separate 
timeline of the game's events, as does the server. As we'll see later, these 
timelines can sometimes diverge. To avoid ambiguity, these posts will use the 
following terms to mean specific things:

* A **simulation** is a series of **events** that occur over time. Time here
  doesn't necessarily correspond to the real world: for example, we could
  choose to run the simulation at half speed. Then 1 second of simultation
  time would elapse for every 2 seconds of real-world time. 

* **Virtual time** (or **v-time**) will always refer to the timeline of a
  simulation. 1 second of v-time elapsed in our previous example.

* **Real time** (or **clock time**) will always refer to the time elapsed in
  the real world. 2 seconds of clock time elapsed in our previous example.

* **Client time** refers to the **v-time** of the client's local version of
  the **simulation**.

* **Server time** refers to the **v-time** of the server's local version of
  the **simulation**. In many cases, **server time** is identical to **real
  time**. 

If you ever find a place in this series where it's not clear whether the time
being discussed is virtual or real, please
[file a bug](https://github.com/davekilian/davekilian.github.com/issues)
against the blog post!

## Example Code

Download: [Example Code I](404)

Each post in this series has attached sample code. It's written in C++ using
the cross-platform [Qt Framework](http://qt.nokia.com) for graphics, windowing,
networking, and the like.

The example code for this post is a game with basic FPS gameplay. We'll be
expanding this into a multiplayer game. 

One of the most common beginner mistakes in networking is to decide to build
your game first and add multiplayer later. Although you should usually leave
networking out of your initial prototypes, 'adding' multiplayer usually means
rewriting your core game logic. If you want your game to be multiplayer, 
don't put it off until the end!

In addition to the FPS skeleton, the example code also implements a UDP
protocol like the one Fielder describes in
[Networking for Game Programmers](http://gafferongames.com/networking-for-game-programmers).

## Onward

Ready to start? Head on over to
[Multiplayer II: Client / Server Architecture](/2012/07/15/multiplayer-ii.html)!

