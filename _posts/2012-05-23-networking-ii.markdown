---
layout: post
title: Networking II
subtitle: Multiplayer with smart servers and dumb clients
author: Dave
---

Welcome back! In this post we'll expand our single-player game into a
basic client/server architecture. The client will be able to render 
the outcomes of the server-side simulation, but will not yet be able
to collect inputs to send to the server.

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

