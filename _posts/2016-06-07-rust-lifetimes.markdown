---
layout: post
title: Rust Lifetimes
author: Dave
draft: true
---

Big long rambly multi-email brain dump

## 1

Based on learnings from reading this: http://stackoverflow.com/questions/31609137/why-are-explicit-lifetimes-needed-in-rust/31609892#31609892

The typical treatment of lifetimes is 'bottom up': explaining the rules and trying to work toward a motivation, and often not quite getting there :-). Maybe it'd be a bit easier to understand if we started with motivation and worked down to the specific rules.

In short, explicitly named lifetimes allow you to tell (the compiler | other people reading your code) about the relative lifetimes of different references in your (struct | function). By doing so,

- You tell the compiler that you intend certain lifetimes to be linked, allowing the compiler to warn you with a compilation error if your code does that meet that intent

- You tell people reading your code how it's safe to use references with your (function | struct)

In other words, lifetimes aren't programming constructs like a for loop; rather, they're restrictions you're placing on your own code as part of an API contract with the caller, like strong typing.

So asking why compilers can't infer explicit lifetimes is like asking why compilers can't infer types: they can, but that can mean unintended consequences for you when you change the code ;-)

In light of that, now we can look at fjh's example. By using explicit lifetimes, we restrict function foo such that the return value is guaranteed to be valid as long as parameter x is valid. The caller can rely on this assumption when they call foo, and for illustration fjh's main() explicitly takes a dependency on this behavior. 

From there we can try and derive the lifetime elision rules

## 2

Maybe there's a little more nuance here.

One point I touched on but want to rephrase: an initial misconception is that explicit lifetimes changes the way the program is compiled to behave. It doesn't; rather, explicit lifetimes express restrictions on the program that you are responsible for upholding, the compiler will check. If you write code that's valid in terms of normal scope-based borrowing, but violates a rule spelled out by explicit lifetimes, then the compiler will fail to compile the program and it'll be up to you to fix the problem.

Now, rust wants to be sure, at compile time, that references never outlive the owner binding. Lifetimes provide breadcrumbs linking handoffs of a reference, such that the compiler can walk the chain of lifetime rules all the way back to the original owner, to determine of the owner scope is >= the reference scope.

In other words, the point of defining explicit lifetimes is to allow you to link references together (e.g. a reference received as function input to a reference returned as function output). The rust compiler can follow these links at compile time back to the owner and check for references that outlive the owner, and fail. But when it finds a failure like this, it's up to you to fix it!

The complier has the whole program available to it during compilation, so it's totally possible for the compiler to infer all of this for your automatically without you using lifetimes. But this opens you up to the possibility of unintended consequences, where a program may be 'right for the wrong reasons.' An innocuous-seeming change can have far-reaching effects on the behavior of totally unrelated functions for reasons you didn't foresee.

Plus, by defining the lifetime rules on function boundaries, humans trying to read the code and understand the lifetimes don't have to fully read function definitions to understand what the relative lifetimes are. You annotated those into the function signature yourself when you defined the function.

And now we can descend into the rules for lifetimes ...

- Lifetimes appear any time references are moved off the stack (including in function inputs/outputs and in struct parameters). 

- The complier will infer lifetimes in a set of select cases that are super common (lifetime elision)

So anyways, super rambly but those are the thoughts

## 3

Further thoughts

We can break down and motivate by using the example of a C++ function

const string &foo(const Obj &a, const Obj &b)

Pretending for a moment that there are no globals/static variables in C++, we can be absolutely sure that the return value comes from one of the input parameters, but which one? You can write two examples, one where a goes out of scope before the return value, and one where b goes out of scope before the return value. Is either example valid? If so which? It depends on where that return value can come from. Enter lifetimes.

Now yeah, the compiler has access to the full function definition and can infer the answer from that. But I need to know that too to write code, and I am not motivated enough to trace through libraries and try to figure out how these relationships work.

And, on top of that, the lifetime of the return value is part of the functions API contract. If it changes, calling code can break in unpredictable ways at runtime. No quiero!

Rusts solution is a system of explicit annotations which trace references back to owners they [possibly] came from. In essence, lifetime annotations provide links that the compiler can trace back to stack allocations, so it can check for use-after-free problems. This system works recursively (or by induction if you're mathematically inclined), so that the compiler can consider each function in isolation, and rely on the fact that each function being called must conform to the lifetime rules annotated into its signature (else it would not have been compiled in the first place!)

These annotations are added to type signature so they become part of the API, and are enforced by the compiler to help prevent accidental breaking changes to existing public APIs. If you make a breaking change to return value lifetime rules, you'll certainly know you did, and so will people who take up your changes upstream 

The lifetime annotation rules themselves fall out of the fact that rust always needs this information, but it's tedious for the programmer to write. Thus the rules are 'always annotate, except in these very common cases.' These lifetime elision rules basically work by making extremely weak assumptions about your lifetimes (eg the ref returned from a function could be from any of its inputs). If you want to make more nuanced assertions, you can do so by explicitly providing lifetimes to the function signature.

## 4

Even further thoughts

We start out with examples that only refer to primitive types. Then we break out structs. Structs can hold references to things they don't own, which complicates our examples: if you copy that reference and return it, the reference may be allowed to outlive all the functions input parameters. 

So, we add lifetime annotations to the type signatures of structs as well. Like with functions, once you add an annotation to the struct's type signature, the type is fundamentally different (you don't have a foo anymore, you have a 'foo referring to one or more variables in lifetime 'a). Once that type is introduced, functions can return the reference out of that struct if the return value lifetime is the lifetime that foo refers to.

Huzzah, we can still follow references up the call stack.

Now, what's nice about this annotation system is that both you and the compiler can consider any function totally in isolation while setting up lifetimes.

- you know your function's signature, including lifetimes
- you know the signatures, including lifetimes, of each function you call
- now, parameters you accept as inputs, and the things they refer to, are guaranteed not to go out of scope at any point during your function
- so that means you only have to worry about the lifetimes over your local variables and the function return value

This is manageable for a human and efficient for a compiler. And this is the point where we can show by recursion or induction, depending on your preferred parlance, that this works for the whole program.
