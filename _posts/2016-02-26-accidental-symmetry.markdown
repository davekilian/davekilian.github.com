---
layout: post
title: Beware of Accidental Symmetry
author: Dave
draft: true
---

Brain dump

Let's say you're writing a CLI to exercise a couple different entry points of a shared lib.
At first you only support one argument, so you implement it as:

~~~cpp
if (0 == strcmp("-foo", argv[1])) {
    myLib->foo();
}
~~~

Later you add another entry point:

~~~cpp
if (0 == strcmp("-foo", argv[1])) {
    myLib->foo();
}
else if (0 == strcmp("-bar", argv[1])) {
    myList->bar();
}
~~~

And later, two more:

~~~cpp
if (0 == strcmp("-foo", argv[1])) {
    myLib->foo();
}
else if (0 == strcmp("-bar", argv[1])) {
    myList->bar();
}
else if (0 == strcmp("-baz", argv[1])) {
    myList->baz();
}
else if (0 == strcmp("-quux", argv[1])) {
    myList->quux();
}
~~~

We're starting to see a pattern here -- every entry point is a void function that takes no arg.
So let's simplify this a bit for people adding new entry points:

~~~cpp
typedef void (*EntryPoint)();

static struct { const char *argname; EntryPoint func; }
EntryPoints[] =
{
    { "-foo",  myLib->foo },
    { "-bar",  myLib->bar },
    { "-baz",  myLib->baz },
    { "-quux", myLib->quux },
};

#define ArraySize(a) (sizeof(a) / sizeof((a)[0]))

for (int i = 0; i < ArraySize(EntryPoints); ++i) {
    if (0 == strcmp(argv[i], EntryPoints[i].argname)) {
        EntryPoints[i].func();
    }
}
~~~

Now if a user wants to add a new entry point, they just add a new item to the `EntryPoints` table.
The extra boilerplate from arg parsing is factored away.

So, is this a good refactor?

It depends.

Can you guarantee that every entry point will always conform to the signature `EntryPoint`?
What happens if some poor developer wants to add a command line entry point for a function which accepts a single user argument?
Before the refactor we did above, this was easy:

  1. Add an `else if` for the new function
  2. Check for `argv[2]`
  3. Call the function, passing `argv[2]`

But after our refactor, adding something which should seem simple is actually a big mess. 
The developer trying to add the entry point is basically screwed at this point.
What are they tod o?

* Extend the `EntryPoints` table to somehow represent functions which require an argument?
* Add a special case for the one specific function which requires an extra parameter (and pray this case never comes up again)?

Either way, what would have been a simple refactor is now going to be a major rework of the subsystem being touched.
These kinds of unexpected costs are what put software projects behind schedule!

The lesson learned: your software development education and experience will teach you to become a very good 'code compressor.'
Over time you'll get better and better at seeing when duplicated code could be replaced by another programming construct which reduces boilerplate.

Sometimes you'll see refactors that are possible based on the way things are today.
Remember when doing a refactor to think about invariants.
For example, this isn't a good reason to do the refactor above:

> For example, today all entry points in my CLI take zero arguments.<br>
> Therefore I can represent all of them using the same function pointer.<br>
> Therefore I can refactor the entrypoints into a table indexed by a for loop.

But this is:

> By nature of the library I'm writing, I can guarantee all entry points in my CLI take zero arguments.<br>
> Therefore I can represent all of them using the same function pointer.<br>
> Therefore I can refactor the entrypoints into a table indexed by a for loop.

