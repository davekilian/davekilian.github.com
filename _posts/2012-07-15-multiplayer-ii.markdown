---
layout: post
title: Multiplayer II &ndash; Client / Server Architecture
author: Dave
---

Welcome back! It's time to expand our single-player game into multiplayer game
on a basic client/server architecture. Shall we dive straight in?

## "The Server is the Man"

Most, if not all multiplayer shooters use client/server architectures. In a
client/server architecture, each player controls his own client, and all players
are connected to the central server. The server runs the game logic and
broadcasts game state to the clients. The clients render the state and
collect player input. The server uses the input to modify the 
simulation.

This model is simple, elegant, and works wonderfully in a low-latency
environment. Each client only needs to talk to the server, which reduces
bandwidth. Replication is easy because all clients just mirror the server
state. As the title of this section implies, the server is the authority
over the game state. This makes it harder for a single client to cheat.

Unfortunately, this model crumbles when latency is high. Over the Internet,
a game designed like this would be unplayable. We'll fix this by making a
client that constantly guesses what the server is about to do. For now, 
though, we'll pretend we don't need to worry about latency. We'll 
make the client simply mirror the server's game state. Then we'll add guessing
(called _prediction_) in a later post.

---

## Expanding the Original Game

TODO we released source for a simple single-player FPS last time. Now
     we're going to start expanding it into a multiplayer system.
     we'll need to build the basic client, server, and a replication
     system that keeps them in sync.

## Replication

TODO mention game object systems, extend to entity/attribute/scene

TODO we'll assume for now that every entity can `pack()` and `unpack()`
     itself to/from a string. This isn't very compact but alleviates some
     cross-platform issues and makes it simpler to debug. Not appropriate
     for a production system but fine for this tutorial.

## Server-Side Simulation

TODO each tick the server updates the simulation, just like the game did.
     except now, we need to broadcast the result to the clients instead of
     simply rendering the scene

TODO sending the whole world every frame would take too much bandwidth.
     instead we're going to only send the attributes that changed, and
     send broadcasts at a lower rate than the tickrate. 

## Client-Side Interpolation

TODO the client must render at the tickrate, even though it's receiving
     deltas at a lower rate. Additionally, some deltas might get lost.

TODO our solution is to linearly interpolate between deltas. 

TODO client makes the assumption everything moves at a constant velocity 
     in between deltas

TODO finally note that we don't have to assume constant velocity. We could have
     instead assumed e.g. constant acceleration.

TODO wrap up

Also should intersperse code, maybe?

