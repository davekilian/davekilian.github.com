---
layout: post
title: Unity Programmer Quickstart
author: Dave
---

Recently I had the chance to take a closer look at the editor and game engine [Unity](http://unity3d.com) for some personal projects.
Unity is different from traditional game engines.
As is typical, Unity comes bundled with an editor; atypically, Unity's balance of power favors the editor over the engine.
Unity is designed for artists and designers first, programmers second.

As a programmer, I honestly think this is a great idea.
Unfortunately, this attitude is also reflected in Unity's documentation, which is clearly written for designers and artists, not developers.
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
You can't easily make your game types inherit from each other.
This is leaving you with only a handful of options, and they're all bad:

* You can just outright duplicate code between game classes, but this makes bugfixes very expensive
* You can create 'uber-objects' with conditional behavior controlled by flags, but this makes code hard to understand and tweak
* You can may be able to construct inheritance chains, but this is hard to refactor.

The entity/component pattern addresses some of these shortcomings without losing the object-oriented structure for our game code.
The main idea is to build each entity in the game world out of smaller self-contained units called components.
Each component tracks some game state and performs logic on that state as the game runs.
An entity is a generic container which groups components together.
Each entity represents one object in the game world.

This addresses two major shortcomings from the strategy above:

1. **Components implement a behavior once and share it across multiple types of game entities.**
You can add or remove components from an entity and see an instant effect, without needing to refactor anything.
When designed well, components make changes and bugfixes less likely to cause undesired side effects elsewhere.

2. **Components simply entity interaction.**
When components handle interaction between entities, they can examine each entity without worrying about what 'type' of entity it is.
A component in one entity can talk directly to another instance of that component in a different entity.

As always though, there are no silver bullets.
Components systems have their downsides too.
A well-designed entity/component system needs to strike a good balance for component size:

* If your components start to get too specific, you can't share them between entities, and they're hard to understand.

* If your components get too broad, you can share them easily, but the interaction between them becomes hard to understand.

Generally, you want components to be as general as possible while also minimizing how much they interact.

Anyways, that's the gist.
If you're interested, there's much more in-depth discussion just a few queries away online.
So without further ado, let's go back to talking about Unity.

## Unity in a nutshell

Unity provides an editor and a game engine, which are tightly integrated with each other.
You use the editor to build an initial game state (a Scene), and then compile one or more Scenes into a game.
In essence, the editor builds your game by first compiling your code, and then packaging the binaries with your game art/assets and Unity's own game engine binaries.
When you run the resulting game, the program starts Unity's game engine, which loads the Scene and runs the game's main loop.

As previously mentioned, Unity implements an entity/component system.
The editor is a drag-and-drop GUI which aids in building a scene based on entities.
Each Scene in Unity is a hierarchy of entities (which Unity calls GameObjects).
Each GameObject can be assigned one or more Components which define the entity's state and behaviors.

Unity provides a stock component called Transform, which defines a position, orientation and scale.
Unity makes the Transform component mandatory for every GameObject, which means every GameObject has a position, orientation and scale.
The editor makes use of this information to render the GameObjects in a Scene, even if some GameObjects don't have any geoemtry.
It also tends to come in handy when you implement your own components.

Unity's game engine infrastructure (Scene, GameObject, the main loop, etc) are closed source and are not extensible.
Your only option for implementing game logic is to implement custom Components called Scripts.

## Scripts

Unity calls all the game code you write 'scripts.'
You can 'script' in two languages: JavaScript and C# (via the [Mono project](http://mono-project.org)).

Despite the mildly derogatory name ("what are ya, some kinda script kid?"), scripts can be arbitrarily complex.
For example, you could implement a pathfinding system in C#, compile the whole thing into your game, and call into it at runtime to route some AI characters through a level.
As previously mentioned, though, the only way to call into your code in the game is to expose it as a Component of some GameObject.
In this example, you would need to expose your pathfinding system as a pathfinding component as follows:

* Create a pathfinding component class which implements the Unity interface MonoBehavior, which is the base interface for all Unity Components
* Implement the pathfinding component class by calling into your pathfinding implementation
* In the editor, create a new GameObject or choose an existing GameObject to add the pathfinding behavior to
* Make an instance of the component on the GameObject

This is the basic pattern for hooking up your own game logic to the scene.
In practice you may actually need to define multiple components which call into your pathfinding code differently (e.g. you might need one component for the AI logic which uses pathfinding, and another for defining the paths themselves).

Components (i.e. script classes which implement MonoBehavior) can publish properties to be exposed in the editor to the user.
Unity makes this pretty simple: using reflection, Unity finds any public fields/properties on the object and displays an editor control for that property.
In addition to setting basic types like strings and numbers, you can also use the editor to set up references to other objects in the scene.

## Scenes and GameObjects

The Unity editor is ultimately a tool which helps you build a Scene out of GameObjects.

A Scene is just a hierarchy of GameObjects; each GameObject in the scene can optionally 'own' one or more child GameObjects.
Unity calls this notion "parenting" a GameObject.

Although GameObjects themselves don't do anything with this hierarchy, Components attached to the GameObjects have the option of applying themselves recursively.
For example, Unity's Transform component is defined as being relative to the parent GameObject's Transform.
This means any change to the position, orientation or scale of a parent GameObject implicitly applies to all its child GameObjects recursively.
When used correctly, this type of recursion can be a powerful tool for simplifying game logic.

In addition to using the GameObject hierarchy, Components can find other GameObjects in the scene in a couple different ways.
As previously mentioned, Component scripts can define GameObject references set up by the user in the editor, allowing the Component to statically refer to other GameObjects in the scene.
As time passes, these Components can change the references or pass them on as needed.

Another way Components can find GameObjects is via tagging.
Tags are strings defined on a GameObject.
Typically tags are set up manually in the editor by the user, but they can also be created or modified programmatically.
Components can query the scene to get the GameObject(s) which match a particular tag.

## The editor at the center of the universe

The last note I wanted to touch on is a bit broader than what we've covered so far: Unity's design philosphy is try to remove code from game development as much as possible.
De-emphasing code greatly simplifies life for designers, artists and new game developers, but it can also feel a little foreign to more seasoned developers who are used to controlling the entire flow of the program.
Even so, in order to gel with Unity and its ecosystem, it's important to be mindful of this philosophy.
It colors many aspects of Unity's design and ultimiately affects your designs for your systems.

The editor-centric philosophy tends to manifest itself in two ways:

1. When it's possible to remove custom code from a workflow, Unity will opt to do just that.
The solution is usually to integrate the customization aspect into the editor, and implement the runtime aspect in the game engine.
Unity provides editor extensibility points to allow you to follow a similar pattern with your own custom workflows (e.g. if you wanted to create custom animations for game cutscenes, you could build a custom UI in the editor to design the cutscene, and implement runtime components for playing the scene back).

2. When it's not possible to remove custom code from the equation, Unity tends to compensate by boxing the code into component behaviors, which the user can then invoke a la carte via the editor.
This way code itself can be exposed in the editor as a primitive.

In short, in Unity the editor is the center of the universe.

## Next steps

From personal experience, I highly recommend going to Unity's documentation page, opening the manual, and then skimming through it in an evening or two.
Note that the manual, being a manual, covers the material in a depth-first traversal, which is inconvenient when you're starting out and you're looking for overall concepts and breadth.
That said, I find it much easier to skip unnecessary details in text docs than it is to skip through video tutorials, which are popular in the Unity landscape.

Other than that, the best way to learn is to just start prototyping whatever it was that brought you to Unity in the first place.
Happy coding!

---

TODO

One note to add if it's not already up there somewhere: I strongly believe Unity also has a far-flung dream to commoditize programming in gamedev.
Unity does a lot of work to take the code and box it into little pieces that designers pick up as part of their game.
By adding components to the store, they're continuing on the path to having a universal library of 'game code' for most types of games.
