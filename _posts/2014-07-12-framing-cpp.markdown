---
layout: post
title: Framing C++ with 3 Core Ideas
author: Dave
draft: true
---

C++ is an enormous language, with dozens upon dozens of language features.
Beginning can seem daunting at times, but luckily C++ is old and ubiquitous,
which means there's tons of information online to help you get started.

In that case, why write another guide?
This one aims to be a little different from most C++ intros.
We won't spend much time on language features or syntax - you can find all that
stuff easily and better-documented elsewhere
[(1)](http://en.wikipedia.org/wiki/C%2B%2B)
[(2)](http://www.cplusplus.com)
.
Instead this guide will cover the Big Ideas behind C++.
The aim is that, once you finish this guide, you should be able to read about
specifics and understand why each feature works the way it does.

To get the most out of this guide, you'll need some background in a more modern
object-oriented language with C-style syntax.
Examples include Java, C# and Go.
No prior knowledge of C is required, nor will we cover C at all in this guide.
We also recommend doing a cursory tutorial to get a feel for the basic syntax.
Finally, consider taking each section one at a time, with some time in between
to mull things over and wait for them to click.

## 1: Typing with Bare-Metal Memory

## 2: Resource Lifetimes

## 3: Compilation Model

C++ supports a different compilation model than you may be familiar with.
Namely, C++ is design to work with single-pass compilers,
whereas modern languages tend to rely on multiple-pass compilation.
To understand what this means, we'll have to take a tiny peek at compiler 
internals.

In a compiled programming language, the compiler's job boils down to:

1. Finding type definitions
2. Compiling each definition to executable machine code
3. Hooking definitions together to form a complete program

Let's compare a multi-pass Java compilation to a single-pass C++ compilation.
Consider this Java source file:

```java
public class Counter {

    private int field;

    public Counter() {
        field = 0;
    }

    public int getNext() {
        System.out.println("Somebody called getNext()");

        incrementField();
        return field;
    }

    private int incrementField() {
        field += 1;
    }
}
```

Here's how Java handles this class:

1. First, it scans the source file for type definitions.
   It finds definitions for symbols like `Counter`, `Counter.field`,
   and `Counter.getNext()`, to name a few.
   It saves basic type information about these symbols for later.

2. Then the compiler revisits the source file and compiles each definition.
   Whenever a definition references another type, Java uses the basic type
   information to generate a stub referencing that type.

3. Finally, once each definition has been fully compiled,
   Java combines the compiled definitions together into a single program.

This model is called _multi-pass_, because Java needs to scan each source file
multiple times to compile it.
Now let's take a look at the equivalent C++ class:

```cpp
class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};

Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    incrementField();
    return field;
}

Counter::incrementField() 
{
    field += 1;
}
```

C++ is designed to be compatible with single-pass compilation: a compiler
should only need to walk through a C++ file once to completely compile it.
Given a source file, the compiler parses it from top to bottom and compiles it.
Whenever it runs into a definition, it compiles the definition on the spot.
Then, at the end of compilation, it combines all those compiled definitions
into a single program.

This poses a problem, though: if the definition references another type, the
compiler must already know about the type being referenced. 
Otherwise the compiler wouldn't be able to validate that this type actually
exists later on in the file, and isn't just a user typo.
To solve this problem, C++ makes extensive use of a technique called _forward
declarations._

Unlike a definition, a forward declaration doesn't give the compiler enough
information to compile the type being declared.
Instead, the declaration just tells the complier that you intend to fully
define that symbol later, and also gives the compiler basic information about
what that symbol is (a class, a variable, a method, etc).

In the example above, we gave the compiler two forward declarations: inside
the `class Counter` scope, we gave the compiler forward declarations for 
`getNext()` and `incrementField()`.
Later in the file, we actually defined those methods by writing a method body.
The compiler processed the file from top to bottom, and since we declared these
two methods at the top, the complier knew about each method before it reached 
either method's definition near the bottom.
This allowed us to reference `incrementField()` inside of `getNext()`,
even though `incrementField()` is defined later in the file.

Here's the same example file as above, except we put the method definitions
before their declarations.
This file won't compile, since the compiler won't know about 
`incrementField()`, or even `class Counter`, by the time it gets to the 
definition for `getNext()`:

```cpp
Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}

Counter::Counter()
{
    field = 0;
}

class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};
```

One last note about forward declarations: the compiler allows you to declare a
symbol as many times as you want, provided each declaration is the same.
However, you can only define a symbol once.

So far we've only been talking about compiling a single file,
but only the smallest of programs consist of just one source file.
The natural next step is to wonder: how would a C++ compiler compile multiple
files into a single program?
C++ programs are compiled in two steps:

* First, the compiler compiles each source file individually.
  For each source file, it produces a corresponding _object file_ containing
  compiled machine code for each definition contained in that source file.
  References to external types are stubbed out in these object files.

* Second, the linker links the object files into a single _binary file_,
  which is typically either a library or an executable program.
  The linker basically joins all the compiled definitions,
  and then fills in stubs left by the compiler.

Herein lies another problem: each time the compiler processes a source file, it
starts from scratch and scans the source file from top to bottom.
No state is shared between different invocations of the compiler.
But source files often reference each other: it's not uncommon for one source
file to define a method, and for another source file to use it.

A simple, but hard-to-maintain way to get around this is to manually declare 
objects and methods you want to use at the top of every C++ source file.
But this is cumbersome and hard to maintain, especially once you start trying
to use classes in the standard library.
The C++ solution to this is the _header file_.

The idea behind header files is to move declarations (but _not_ definitions!)
to a single shared file.
Every .cpp source file that needs those declarations can import the .h header
file using the `#include` directive.
For example, we can rewrite our example above as follows:

**counter.h:**

```cpp
class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};
```

**counter.cpp:**

```cpp
#include "counter.h" 

Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}
```

Again, notice how `counter.h` only has declarations, and `counter.cpp` only has
definitions.
Now we can write a `main()` method that uses the counter class:

**main.cpp**

```cpp
#include "counter.h"

int main(int argc, const char *argv[])
{
    Counter myCounter;

    for (int i = 0; i < 10; ++i) {
        int num = myCounter.getNext();
        bool debug = (i == myCounter.getNext());
            // debug should always be true
    }

    return 0;
}
```

One important thing of note: the compiler processes `#include` directive using
a component called the _preprocessor_, and the preprocessor is pretty dumb.
Here's how include directives are processed:

* First, the compiler makes a temporary copy of the source file in memory.
  It invokes the preprocessor on this in-memory source file
  (note this never gets written back to the original file on-disk).

* The preprocessor walks through the file from top to bottom.
  Whenever it finds an `#include` directive, it loads the file to include.
  If there is no such file, compilation is aborted.

* Next, the preprocessor deletes the source file line containing the `#include`
  directive

* Finally, the preprocessor pastes the full text of the included file into the
  .cpp source file at the line of the #include directive

* Once the preprocessor exists, the compiler walks through the preprocessed
  file to compile the program.

That has a few important ramifications.
One is that, even though the `counter.cpp` above compiles correctly,
this one won't:

```cpp
Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}

#include "counter.h" 
```

Why doesn't this work?
Because the full preprocessed text of this file is as follows.
We've already discussed why the compiler will choke on this:

```cpp
Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}

class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};
```

To recap, here's the core idea from this section:

> _C++ requires every symbol you reference to be declared or defined prior to
> the time you use it. 
> Publish declarations in header files, so that any definition in a source file
> can reference the symbols it needs._

Now it's time to try putting everything together.
We're going to compile the counting example above into a working program.
Here's our high-level strategy:

* We will invoke the compiler twice: once on each source file.
  The output of each invocation will the source file's corresponding object
  file.

* Then we will invoke the linker on the object file.
  The output will be a working program!

Let's walk through the compilation process.

1. We (or our IDE) invokes the compiler on `counter.cpp`.
   The compiler loads this file into memory:

```cpp
#include "counter.h" 

Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}
```

2. The compiler preprocesses the in-memory file.
   Upon finding `#include "counter.h"`, the preprocessor loads it in.
   After preprocessing, the in-memory file looks like this:

```cpp
class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};

Counter::Counter()
{
    field = 0;
}

Counter::getNext() 
{
    std::cout << "Somebody called getNext()\n";
    incrementField();
    return field;
}

Counter::incrementField()
{
    field += 1;
}
```

3. The compiler walks through the file.

    1. At the top, it finds a definition for `class Counter`.
    2. Walking through that definition, it finds forward declarations for the
       constructor, `incrementField()`, and `getNext()`.
    3. As it continues to walk the file, it finds definitions for those
       methods.
       It compiles each method as it encounters the definition.
       The resulting compiled definition contains a stub wherever the
       code referenced another definition.
    4. The compiler saves all the compiled definitions in an object file for 
       counter.cpp (usually `counter.o` on UNIXes, `counter.obj` on Windows).

4. We repeat step (2.) on `main.cpp`.
   The preprocessed result looks like this:

```cpp
class Counter
{
private:
    int field;
    int incrementField();

public:
    Counter();
    int getNext();
};

int main(int argc, const char *argv[])
{
    Counter myCounter;

    for (int i = 0; i < 10; ++i) {
        int num = myCounter.getNext();
        bool debug = (i == myCounter.getNext());
            // debug should always be true
    }

    return 0;
}
```

5. We repeat step (5.) on `main.cpp`.
   The resulting object file is either `main.o` or `main.obj`,
   depending on your platform.

6. We invoke the linker on `counter.o[bj]` and `main.o[bj]`.

    1. The linker concatenates each of the compiled definitions
       into a single program binary.
    2. The linker scans through the compiled definitions.
       Whever it finds a stub for referencing another definition,
       it fixes the stub to actually call into the referenced code.

7. The linker writes the final binary to disk.
   Now we can run it!

## Recap

Once again, here are the three core concepts this guide boils down to:

> 1. _TODO_
>
> 2. _TODO_
>
> 3. _C++ requires every symbol you reference to be declared or defined prior to
> the time you use it. 
> Publish declarations in header files, so that any definition in a source file
> can reference the symbols it needs._

Having grokked the concepts above, we hope you now have enough of a working
framework to fit in all the small details about writing C++ code.
You're not done with your journey, but you're off to a good start.
Best of luck!

_Find this guide helpful? harmful? Did you find a bug?
Please give any feedback by opening an issue on 
[this blog's git repo](https://github.com/davekilian/davekilian.github.com).
Or feel free to fix it yourself and open a pull request :)_

