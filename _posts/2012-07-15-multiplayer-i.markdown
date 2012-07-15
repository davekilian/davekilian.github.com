---
layout: post
title: Multiplayer I &ndash; Introduction
subtitle: Building a robust networking system for online multiplayer
author: Dave
---

Ever wanted to write a multiplayer video game?

Writing a multiplayer game that works well over a LAN isn't too hard. Packets
moving through a LAN tend to get from A to B quickly. It's
usually enough to replicate the state of your game world between two networked
machines.

Writing a multiplayer game that works well over the Internet is a different
story. Packet loss is common. Latency is high, and changes without warning.
The anonymity of online play encourages some players to cheat. 
Dealing with these conditions requires a bit more finesse. 

This series of blog posts is a nuts-and-bolts introduction to building 
an Internet-ready first-person shooter. The game will be playable over typical
broadband connections, and we'll take some steps to mitigate cheating.

These posts are based on what I learned while building online multiplayer 
for [nullZERO](http://fracture-studios.com/404). I drew heavily from
publishings about [Valve's Source Engine](http://source.valvesoftware.com/) 
and [id Software's QuakeWorld](http://en.wikipedia.org/wiki/QuakeWorld). 

## Audience

This series is aimed at readers who have experience developing video games.
Readers should know enough to make their own single-player FPS without
relying heavily on tutorials or game engines.

Readers should be able to read C++. Each post in this series has C++ example
code, built on the [Qt Framework](http://qt.nokia.com). Code snippets 
throughout the series will also be written in C++. Most of the content, 
however, can be applied to your programming language of choice.

Game networking is shenanigans. You should be prepared to get stuck a couple 
times while following the tutorial. When you do, take a break and come back
later!

## Outside Reading

Before starting this tutorial series, I highly recommend you read Glenn 
Fiedler's 
[Networking for Game Programmers](http://gafferongames.com/networking-for-game-programmers) 
series. In particular, I won't be covering packet transport layers because
Glenn does it so well. He explains client/server gameplay at a high level;
this series covers the same with more technical detail.

The following is a list of documents I consulted when developing multiplayer
for nullZERO. You may also find them useful:

* [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
* [Latency Compensating Methods in Client/Server In-Game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)
* [NPC Lag Compensation](https://developer.valvesoftware.com/wiki/NPC_Lag_Compensation)

---

TODO let's move the rest to the next post. Finish out with some skeleton
code for a first-person FPS and an implementation of the custom UDP protocol.

---

## "The Server is the Man"

Most, if not all multiplayer shooters use client/server architectures. In a 
client/server architecture, each player has his own client, and all players 
are conencted to the central server. The server runs the game logic and 
broadcasts the results to the clients. The clients render the results and 
collect player input. The server uses the player input to modify the 
simulation.

This model is wonderful. The client doesn't need to understand anything about
the server's simulation logic. Since the client is not authoritative about the
game's state, it's impossible for the client to cheat. Clients save on
bandwidth because they only need to talk to the server. Replication is easy
because all clients just mirror the server's state. 

As described, this model would work great on a LAN. 
Unfortunately, factoring in the latency and unreliability of an Internet
connection uncovers serious responsiveness problems. To mitigate these we'll
end up making the model more complicated. But for now, we'll assume
the client is a dumb terminal that just renders the server's simulation.

## What is a Simulation?

We'll use the term _simulation_ to mean a sequence of _events_ that happen
over time. "Time" in terms of the simulation is not the same as real-world
clock time. For example, we could choose to run a simulation at half speed,
which means 1s of simulation time elapses every 2s of real-world time.

To avoid ambiguity, we will always refer to virtual simulation time as
_simulation time_ or _sim-time_, and real-world physical time as _wall time_. 

With this, we can more precisely state the jobs of the server and client:

* The server generates simulation events, as dictated by the simulation 
  logic and client inputs. The simulation time always matches the server's
  wall time perfectly.

* The client renders the simulation. The simulation time being rendered
  is always a small, constant amount earlier than the client's wall time.

Note that wall time may be different for the client and server, but the
simulation time will always match (since the client and server are
dealing with the same simulation).

## Example Code

TODO

Now that we've got some of the basics out of the way, we can start building
our system. Head on over to [Networking II: Client/Server Architectures](TODO)
when you're ready!

