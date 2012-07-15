---
layout: post
subtitle: How to build a robust networking system for multiplayer shooters.
author: Dave
---

Welcome to game networking! In this series of posts, we'll build a simple 
multiplayer first-person shooter that we can play over the Internet.
Along the way we'll talk about all the lies networked games tell
their players. Our system will be playable over (reasonably) slow or unreliable 
connections, and we'll take some basic steps to mitigate cheating.

The system we'll build is based on the networked multiplayer system we built
for [nullZERO](http://fracture-studios.com/404). It draws heavily from 
documents published by the people who made [Valve's Source Engine](TODO)
and [id Software's QuakeWorld](TODO). 

Pages we found particularly useful include:

* TODO
* Links from the nullzero client-server doc

## Prerequisites

This series is aimed at readers who have some experience creating 3D video games.
Specifically, you should be able to write the skeleton for a single-player
first-person shooter without much difficulty.

Additionally, each post in this series is accompanied by example code, written
in C++ using the [Qt Framework](http://qt.nokia.com/). Although not strictly
necessary, being able to read the example code will be helpful if you get stuck.

Ready to start? Before moving on to the next part of this tutorial, please take
some time to read Glenn Fielder's excellent series
[Networking for Game Programmers](http://gafferongames.com/networking-for-game-programmers/).
The rest of this tutorial will assume you've already done so.

## Review: Smart Servers and Dumb Clients

In a client/server architecture, several players each interface with a client
connected to the central server. The server runs all simulation logic and
broadcasts the results to the clients. The clients simply render the results
and collect the players' inputs. The server uses these player inputs to modify
the simulation.

Ideally, this would mean the client doesn't need to know any of the simulation
logic. Unfortunately, this is not the case: to work smoothly over
an imperfect connection, the client will need to run simulation logic on
player input before the input reaches the server.

In future posts, we'll talk about expanding the client this way, but for
now we'll assume the client is a dumb terminal that just renders the 
server's simulation.

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

