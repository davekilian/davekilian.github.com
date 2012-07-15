Roadmap
=======

## Concepts

* Basic FPS controls
* Client/server architecture
* Simulation terminology
* Client-side interpolation
* User input commands
* Prediction
* Lag compensation

## Posts

1. Post: networking is lies! Goal, target audience, prerequisites
   (e.g. gafferongames). How/why we simulate latency and packet loss.

   Code: A bare-bones game object system that implements a single-
   player first-person shooter. 

2. Post: client-server architecture, simulation terminology. 
   Server simulates, client is an interpolating "dumb terminal"

   Code: Entities are serialized. Server runs the simulation, and
   the client only interpolates. We add a special object (e.g. a 
   bouncing ball) to demonstrate interpolation is working. 

3. Post: Server receives client inputs, which are discretized. 
   Discuss why we send inputs and not the outcomes of inputs (e.g.
   'client is moving forward' rather than 'client moved here'). 

   Code: Server simulates using client inputs. Without prediction,
   client input is delayed.

4. Post: Client-side prediction, diverging simulations.
   "What" happened is more important than "when." 

   Code: client-side prediction. Render interpolated ghost to 
   demonstrate eventual resync. 

5. Post: When lag compensation is necessary, how it works. 

   Code: lag compensation for shooting. Players should be able
   to hit fast-moving objects if they did client-side
   
6. Post: extensions (smoothing prediction errors, etc)

   No code. 

