---
layout: post
title: Unity Programmer Quickstart
author: Dave
draft: true
---

Recently I had the chance to take a closer look at the editor and game engine [Unity](http://unity3d.com) for a personal project.
Unlike traditional game engines, Unity tilts the balance of power in favor of its editor over the engine itself.
It's designed for artists and designers first, programmers second.

As a programmer, I think this is a great idea.
However, this attitude is also reflected in Unity's documentation, which is clearly written for designers and artists, not developers.
As a programmer, all the info you need *is* in Unity's manual, but the manual isn't structured to make it easy to to find.

Consider this guide the lost "overview for programmers" page for programmers getting started with Unity.
This post will focus on the big picture.
Once you have that, I still recommend thumbing through the manual and picking out the parts that are useful to you.

## Entity/component systems

At its core, Unity implements an [entity/component system](https://en.wikipedia.org/wiki/Entity_component_system).
In case that term is new to you, we'll recap here.

Imagine you're building a game.
You know the faster you can iterate on your game design, the faster you can experiment your way into fun and compelling gameplay.
To accomplish this, you want simple, flexible and maintainable code.
You don't want refactoring code or tracking down nasty bugs to get in the way of experimenting.

An obvious design for coding a game is to model each of the 'things' in your game as classes.
To build the game world, you just make the right instances of the right classes.

So you get started coding up a game this way.
However, part of the way through you start to notice a problem:
a bunch of your game 'things' are similar and share behaviors, but no two 'things' are exactly the same.
There isn't an easy way to share code between the 'things:' you have a handful of options, but they're all bad:

* You could just outright duplicate code between game classes, but this makes bugfixes very expensive
* You could create 'uber-objects' with conditional behavior controlled by flags, but this makes code hard to understand and tweak
* Even if you find yourself able to construct inheritance chains now, doing so locks in your design pretty much forever.

How can we improve this?
One key observation is usually the behaviors we want to share are mostly self-contained.
For example, the code which makes a 'thing' react to physics doesn't need to interact with the code which determines how the thing looks, or what happens when the player tries to talk to the thing.

So if all of these behaviors are roughly self-contained, could we implement them as standalone pieces?
The answer is yes!
This is the core underlying idea in the entity/component pattern.

Unsurprisingly, an entity/component system consists of two major pieces:

* **Components**, which are self-contained units for behavior.
  Each component not only encapsulates some game logic, but also remembers any related state.

* **Entities**, which are generic containers for components.
  There is one entity for each 'thing' in your game world.

One tricky bit is game state.
Entities don't have any game state on their own; instead, they just have components.
(You could literally make a simple entity with the declaration `class Entity : List<Component> { }`).
Instead, each component tracks the relevant state *for the parent entity*.
In other words, the component stores the state, but the state logically applies to the entity as a whole.
For example, a `PhysicsComponent` might keep track of the parent entity's velocity/mass/momentum/etc.

Why go through all this trouble?
The component system we just set up addresses two major shortcomings from our class-based approach:

1. **Components implement a behavior once and share it across multiple types of game entities.**
You can add or remove components from an entity and see an instant effect, without needing to refactor anything.
When designed well, components make changes and bugfixes less likely to cause undesired side effects elsewhere.

2. **Components simplify entity interaction.**
When components handle interaction between entities, they can examine each entity without worrying about what type of game 'thing' it is.
A component in one entity can talk directly to another instance of that component in a different entity.

As always though, there are no silver bullets.
Components systems have their downsides too.
A well-designed entity/component system needs to strike a good balance for component size:

* If your components start to get too specific, they get long, complicated and hard to understand.
  You can't share them between entities because the behaviors are too specific.

* If your components get too broad, you can share them easily, but the interaction between them becomes complicated and hard to understand.
  This might land you with some really nasty, hard to diagnose bugs.

Generally, you want components to be as general as possible while also minimizing how much they interact.

Anyways, that's the gist.
If you're interested, there's much more in-depth discussion just a few queries away online.
So without further ado, let's go back to talking about Unity.

## Unity in a nutshell

As we just finished discussing, Unity is at its core an entity/component system.
This assumption underlies both the game engine and the game editor.
It's also how the editor and the engine integrate with each other.

To make a game, you open the Unity editor and create a *Scene*.
A Scene is just an initial game state created from entities, their components, and the initial state variables for each component.
Then you compile one or more of these Scenes into a full game.
Internally, Unity compiles the game as follows:

1. All your components are compiled into one or more binaries
2. The Scene(s) you designed are serialized
3. Unity packages the compiled component code, your serialized scenes, your game assets, and Unity's own engine binaries into a single executable package

When you run the resulting game, the program starts Unity's game engine, which deserializes the first Scene and loads the Scene's art/assets.
Then the engine enters the game's main loop, which in turn runs the game logic for each component of each entity.

> **A note on terminology**
>
> So far, we've been using the term 'entity' extensively.
> However, Unity developers typically use the Unity-specific term 'game object' instead.
> In this context 'game object' means the same thing as 'entity.'
> You'll rarely see anyone in the Unity community use the term 'entity.'
>
> To fall in line with standards, we'll switch to the term 'GameObject' for the rest of this article :)

Unity provides a stock component called Transform, which defines a position, orientation and scale.
Unity makes the Transform component mandatory for every GameObject, which means every GameObject has a position, orientation and scale.
The editor makes use of this information to render the GameObjects in a Scene, even if some GameObjects don't have any geoemtry.
It also tends to come in handy when you implement your own components.

Unity's game engine infrastructure (Scene, GameObject, the main loop, etc) are closed source and are not extensible.
Your only option for implementing game logic is to implement custom components, called Scripts.

## Scripts

Unity calls all the game code you write 'scripts.'
You can 'script' in two languages: JavaScript and C# (via the [Mono project](http://mono-project.org)).

The name 'script' may imply scripts are expected to be simple/trivial bits of code to add behavior to a game.
However, there's nothing stopping you from implementing complex systems in a 'script.'
For example, you could implement a pathfinding system in C#, compile the whole thing into your game, and call into it at runtime to route some AI characters through a level.

As previously mentioned, though, the only way to call into your code in the game is to expose it as a Component of some GameObject.
In this pathfinding example, you would need to expose your pathfinding system as a pathfinding component as follows:

* Create a pathfinding component class which implements the Unity interface `MonoBehavior` (the base interface for all Unity Components)
* Implement the pathfinding component class by calling into your pathfinding implementation
* In the editor, create a new GameObject or choose an existing GameObject to add the pathfinding behavior to
* Make an instance of the component on the GameObject

This is the basic pattern for hooking up your own game logic to the scene.
In practice you may actually need to define multiple components which call into your pathfinding code differently (e.g. you might need one component for the AI logic which uses pathfinding, and another to define paths).

Components (i.e. script classes which implement MonoBehavior) can publish properties to be exposed in the editor to the user.
Unity makes this pretty simple: using reflection, Unity finds any public fields/properties on the object and displays an editor control for that property.
In addition to setting basic types like strings and numbers, you can also use the editor to set up references to other objects in the scene.

## Scenes and GameObjects

The Unity editor is ultimately a tool which helps you build a Scene out of GameObjects.

A Scene is just a hierarchy of GameObjects; each GameObject in the scene can optionally 'own' one or more child GameObjects.
Unity calls this notion "parenting" a GameObject.
In the broader industry, this pattern is usually referred to as building a *scene graph*.

Although GameObjects themselves don't do anything with this hierarchy, Components attached to the GameObjects have the option of applying themselves recursively.
For example, Unity's Transform component is defined as being relative to the parent GameObject's Transform.
This means any change to the position, orientation or scale of a parent GameObject implicitly applies to all its child GameObjects recursively.
When used correctly, this type of recursion can be a powerful tool for binding entities together, or building big entities out of smaller sub-entities.

In addition the hierarchy, Components can find other GameObjects in the scene in a couple different ways.
As previously mentioned, Component scripts can define GameObject references set up by the user in the editor, allowing the Component to statically refer to other GameObjects at runtime.
As time passes, these Components can change the references or pass them on as needed.

Another way Components can find GameObjects is via tagging.
Tags are strings defined on a GameObject.
Typically tags are set up manually in the editor by the user, but they can also be created or modified programmatically.
Components can query the scene to get the GameObject(s) which match a particular tag.

## The editor at the center of the universe

Now that we've conquered the basic concepts, I wanted to end on the same note I started out on:
As a programmer, Unity cares about artists and designers more than it cares about yyou.
Its design philosophy is trying to remove code from game development as much as possible.
Unity wants to take code and box it up into components, which game designers can add to their toolbox like any other asset.

De-emphasing code greatly simplifies life for designers, artists and new game developers, but if you're more seasoned and used to controlling the entire program flow, this can feel a little foreign.
To gel with Unity and its ecosystem, it's worth being mindful of this philosophy, since this will color many aspects of Unity's design and ultimiately affects your designs for your systems.

The editor-centric philosophy tends to manifest itself in two ways:

1. If it's possible to hide code from a designer-centric workflow, Unity will always opt to do that.
   The solution is often to extend the editor with some customization UI, then implement the runtime aspect in the game engine.
   Unity provides extensibility in the editor to allow you to follow a similar pattern with your own custom workflows.
   For example, if you wanted to create a custom system for animating cutscenes, you can extend the editor UI to help build cutscenes, and write runtime components to play the cutscene.

2. When it's not possible to remove custom code from the equation, Unity will try to box the code into component behaviors, which the user can then invoke a la carte via the editor.
  This way code itself can be exposed in the editor like any other asset.

In short, in Unity the editor is the center of the universe.

## Next steps

From personal experience, I highly recommend going to Unity's documentation page, opening the manual, and then skimming through it in an evening or two.
The manual, being a manual, covers the material in a depth-first traversal, which is inconvenient when you're starting out and you're looking for overall concepts and breadth.
That said, I find it much easier to skip unnecessary details in text docs than it is to skip through video tutorials, which are popular in the Unity landscape.

Other than that, the best way to learn is to just start prototyping whatever it was that brought you to Unity in the first place.
Happy coding!

