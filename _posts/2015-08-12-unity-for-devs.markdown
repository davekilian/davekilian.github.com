---
layout: post
title: Developer Quickstart for Unity
author: Dave
---

I recently took a closer look at using the editor and game engine [Unity](http://unity3d.com) for some personal projects.
Unity is a nice tool, but its target audience is primarily artists and designers, over software engineers.
Although this is probably for the best (:D), one unfortunate consequence is there aren't many great resources to help developers get started with Unity, even though there's tons of great stuff for the pure beginner and for other disciplines.

This guide is my attempt to distill the Unity manual into what the typical software developer needs to know to get started with Unity.
Some prior game programming experience is presumed.

## Entity-component systems

Unity is an implementation of an [entity-component system](https://en.wikipedia.org/wiki/Entity_component_system).
If you're not familiar with entity-component systems, here's a quick overview.

Imagine you're building a game.
As a good developer, you want to make sure your game code is simple, but flexible and maintainable, so you can easily tweak behavior without having to make costly changes to your code or track down difficult bugs.
After all, being able to tweak behavior easily and effectively is a harbinger of fun gameplay.

An obvious strategy to go about this is to model each of the 'things' in your game as classes, and instantiate those classes at runtime.
However, when you start coding this up, you notice you're running into some code duplication problems.
A lot of things in your game share similar behavior, but rarely are two different objects fundamentally the same.
_Most_ game props in the scene are rigid bodies which support collision detection, but only some can move, and only some can take damage or be destroyed, and only some are both.
Other props might fall into one of these categories, but may also trigger special game logic under certain conditions.

So you end up addressing this complexity with degenerate strategies like creating 'uber-objects,' which change behavior drastically based on which flags are set, or you set up long, complex inherientance chains to share your behavior optimally, or you end up just duplicating logic all over the place.
All of these strategies end up hurting your ability to experiment in the long run, as even small changes are liable to create far-reaching bugs.

All this and we haven't even started talking about objects interacting!

The entity-component pattern is an attempt to address some of the shortcomings of this strategy without losing the object-oriented structure for our game code.
The main idea is to factor all game state and behavior into small, self-contained units called 'components.'
Then, each object (a.k.a. "entity") in our game is just a generic container for one or more component instances.

This is to address two major shortcomings from the strategy above:

* Components are a straightforward way to implement a behavior once and share it across multiple semantically-different types of game entities.
When designed well, components make changes and bugfixes less likely to cause undesired side effects elsewhere.

* Object interaction is greatly simplified: when components handle interaction between entities, they don't need to understand 'what' each entity is semantically.
As long as two objects have a 'rigid body physics' component, for example, then they can both detect and respond to collisions in a physically realistic way.

Of course, like anything else, entity-component systems come with tradeoffs.
A well-designed entity-component system needs to strike a balance for component size.
If components are made too specific, each entity won't need to have many components, but the components will be hard to share between entities.
On the flip side, if components are overbroad, they're easy to share, but each object may need to have lots of interacting components.
The need to strike this balance can make development more of a headache, especially in the beginning.
That said, for the types of problems game developers need to solve, the entity-component pattern is often an apt solution.

Anyways, that's the gist.
If you're interested, there's much more in-depth discussion just a few searches away online.
So without further ado, let's return to developing games in Unity.

## In a nutshell

Unity provides an editor and a game engine, which are tightly integrated with each other.
You use the editor to build an initial game state (a Scene), and then compile one or more Scenes into a game.
In essence, the editor builds your game by first compiling your code, and then packaging the binaries with your game art/assets and Unity's own game engine binaries.
When you run the resulting game, the program starts Unity's game engine, which loads the Scene and runs the game's main loop.

As previously mentioned, Unity implements an entity-component system.
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
