---
layout: post
title: 3 Core Ideas Behind C++
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
Before starting this guide, we recommend cursory completion of a basic C++
tutorial, to get a feel for the basic syntax.
Finally, consider taking each section one at a time, with some time in between
to mull things over and wait for them to click.

## Table of Contents

1. [Bare-Metal Memory and Data Types](#part1)
2. [Resource Lifetimes](#part2)
3. [Compilation Model](#part3)

## <a name="part1"></a> Idea 1: Bare-Metal Memory and Data Types

One of the commonly cited reasons people use C++ is that it provides 'low-level
memory access.'
To understand what that means, let's reflect for a moment on programming type
systems.

You've probably heard before that everything stored by a computer, whether in
RAM, on a hard disk, in a CPU register, or barreling down an Ethernet cable, is
just bytes.
One of the chief jobs of a programming language is to insulate you from this.
To you, `int`s are just mathematical integers, on which you can perform
arithmetic.
`string`s are just bits of text that can be sliced, diced and spliced.
But every time you manipulate a variable, you're really just changing a number
that's stored as binary in some cell in RAM.
Different 'types' are just different ways of interpreting these numbers.

Like many programming languages, C++ provides an abstraction over bytes as
primitive types, and objects composed of primitive types.
Unlike many programming languages, C++ provides a very thin abstraction, and
makes it easy to drop into the actual byte representation.

For starters, C++ makes some guarantees about the byte representation for its
primitive types.
It provides a range of numeric types, which differ only by the number of bytes
it takes to represent them:

* `char` is always 1 byte (for this reason, programmers often use `char`s to
  manipulate byte buffers)
* `short` is larger than `char`, but smaller than a standard `int` (typically
  16 bits)
* `int` is typically 32 bits
* `long` is typically 64 bits
* and so on.

For user defined types (`class`es and `struct`s), C++ simply concatenates the
byte representation for each field in the type.
So the following struct:

```cpp
struct MyData
{
    int SomeData;
    int MoreData;
};
```

gets represented in memory somewhat like this:

```
|-0-|-1-|-2-|-3-|-4-|-5-|-6-|-7-| byte offsets
|-- SomeData ---|-- MoreData ---| field names
```

If you had an instance of `MyData` and set its `MoreData` field to some value
(say, 7), the compiler would generate machine code that says:

* Start at the byte address of the `MyData` instance
* Skip forward 4 bytes
* Set the next 4 bytes to `0x00 0x00 0x00` and `0x07`, respectively.

One consequence of this model is that type information is known only at compile
time:
the compiler generates machine code that, at runtime, uses a fixed offset and
byte count to correctly assign a value to a variable.
Since type information is no longer available at runtime, C++ has rather
primitive (if any) reflection capabilities.
Another consequence is looking up a field is dirt-cheap: contrast this
fixed-offset technique with languages that look up properties in a hash table
keyed by property name!

Note that this representation works recursively in the case of nested 
user-defined types.
So if we nested struct as follows:

```cpp
struct Complex
{
    int Before;
    MyData Data;
    int After;
};
```

the byte representation would be similarly nested:

```
|-0-|-1-|-2-|-3-|-4-|-5-|-6-|-7-|-8-|-9-|-A-|-B-|-C-|-D-|-E-|-F-|
|--- Before ----| Data.SomeData | Data.MoreData |---- After ----|
```

Now if you set `Complex.Data.MoreData = 7`, the compiler would start at the
address of the `Complex` instance, jump 8 bytes forward, and set that 32-bit
integer to 7.

You can test everything we said above using C++'s built-in `sizeof` operator,
which takes in a type and evaluates to the number of bytes needed to represent
that type:

```cpp
unsigned int numBytes = sizeof(char); // always 1
numBytes = sizeof(int); // usually 4, rarely 2 or 8
numBytes = sizeof(MyData); // 8, possibly with extra padding space
```

Since every type in C++ has a well-defined byte structure, and even a
programmer-friendly byte size, C++ treats both primitive and user-defined data
types as value types.
This means, when you assign one instance of a certain type to another, the
compiler simply does a byte-by-byte copy from the source instance to the
destination instance.
Similarly, each time you call a function, C++ copies in each argument by value,
using this byte-by-byte copy technique.
One side effect of this: a function can modify the argument it receives from
the caller, without affecting the caller's instance.

```cpp
void fiddleWithData(MyData data)
{
    data.SomeData = 1234;
    data.MoreData = 5678;
}

// ...

MyData data;
data.SomeData = 4321;
data.MoreData = 8765;

fiddleWidthData(data);

assert(data.SomeData == 4321);
assert(data.MoreData == 8765);
```

Contrast this with newer languages, where primitives are often passed by value,
but objects are passed by reference.

Like function arguments, function return values are passed back to the caller
by value, using a byte copy.
This means that modifying the return value of a function doesn't modify the
original value that was returned:

```cpp
class Container
{
private:
    MyData data;

public:
    Container() { data.SomeData = 1234; data.MoreData = 5678; }
    MyData getData() { return data; }
};

// ...

Container container;
MyData data = container.getData();
data.SomeData = 4321;
data.MoreData = 8765;

assert(container.getData().SomeData == 1234);
assert(container.getData().MoreData == 5678);
```

Of course, all this copying isn't free.
It's not too difficult to create large C++ objects, hundreds of bytes long.
This is especially true if one or more fields is an array.
Passing around huge objects can be a source of _uniform slowness_, a
performance problem where your program has no specific bottleneck to optimize,
because as a whole it wastes time doing basic things like passing arguments to
function.
The fix for excessive copying is the ability to pass instances around by
reference.
To do that, we'll need to start using _pointers_.

Among other things, pointers a way to reference objects in C++.
At the byte level, a pointer is just an integer.
In fact, you can do a lot of `int`-like things to a pointer, like assigning it
an arbitrary numeric value and doing arithmetic on it.
Unlike `int`s, though, pointers can be _dereferenced_.

When you dereference a pointer, the compiler treats the pointer's value as a
byte address in memory.
So if you picture all of RAM as a giant array of bytes (regardless of how your
program is using these bytes), a pointer is just an index into this byte array.
C++ provides the pointer-dereferencing operator `*` to allow you to read or
write the value in RAM at the byte address stored in the pointer:

```cpp
void pointerTest(int *numPtr, MyData *dataPtr)
{
    int num = *numPtr;
    num += 1;
    *numPtr = num;

    MyData data = *dataPtr;
    data.SomeData = 4321;
    *dataPtr = data;
}
```

C++ also provides field deferencing operator `->`

```cpp
void pointerTest2(MyData *data)
{
    (*data).SomeData = 4321; // This syntax is equivalent
    data->SomeData = 4321; // to this syntax
}
```

Let's reconsider the example near the beginning of this section, where we
walked through how the compiler would set `data.MoreData = 7`.
Let's do the same thing with a pointer to a `MyData` instance:
what does the compiler do with `data->MoreData = 7`?
The answer, is basically the same, except for the first step:

* Start at the byte address stored in the `data` pointer
* Skip forward 4 bytes
* Set the next 4 bytes to `0x00 0x00 0x00` and `0x07`, respectively.

What's important to realize is that _we never actually checked that what we're
doing makes sense_.
When you say `data->MoreData = 7`, the compiler generates machine code that
does some basic arithmetic and then copies some byte values.
It does not, and in fact cannot, verify that the address you're writing to is
really an instance of `MyData`, or that the memory address is valid at all!
In other words, the snippet `data->MoreData = 7` does exactly the same thing
as the following snippet:

```cpp
char *bytes = (char*)data;
int *moreData = (int*)(bytes + 4);
*moreData = 7;
```

These semantics for referencing memory and dealing with types are what makes
C++ so 'close to the metal'.
The entire type system is basically syntactic sugar on top of byte offsets that
get hardcoded into machine code by the compiler.
Although this is about as fast as you can get, it also comes with the caveat
that you can't be sure the pointer contains a meaningful memory address.
If you're lucky, a buggy pointer will contain an invalid address, and attempts
to dereference it will crash the program.
If you're unlucky, a buggy pointer will actually point back into data your
program is using, returning bogus data when read from and corrupting memory 
when written to!

To help prevent this kind of bug, C++ has reference types.
Reference types are just pointers in disguise.
They also come with additional semantics: unlike pointers, references must
always be initialized to the address of an existing object.
You also cannot do arithmetic on references as you can with pointers.
This makes reference types safer than pointers for referencing object
instances, but less powerful than pointers overall.

Back to our original discussion about passing function arguments by value, we
can now use pointer types to pass a `MyData` instance by reference.
Note that, internally, what actually happens is the compiler copies in the
pointer itself by value; but since both copies of the pointer contain the same
memory address, the same object gets modified when either copy of the pointer
is dereferenced:

```cpp
void acceptPointer(MyData *data)
{
    // ...
}
```

We can accomplish the same thing using reference types instead of pointers:

```cpp
void acceptReference(MyData &data)
{
    // ...
}
```

For performance reasons, it's often advantageous to pass in a reference to an
object to a function rather than the object itself by value, to reduce the
overhead needed to copy in the object.
In fact, this is true pretty much any time `sizeof(Object) > sizeof(Object*)`.
However, passing the object by reference comes with the downside that it's now
possible for the function to have a side effect of modifying the object you
passed in.
Thus, a common idiom C++ is to accept a `const` reference to an object as the
input to a function:

```cpp
void acceptConstRef(const MyData &data)
{
    // ...
}
```

With this idiom, we can reap the benefits of passing an object by reference,
while also guaranteeing that calling the function will not modify the original
object (if any code inside `acceptConstRef()` attempted to modify `data`
itself, the compiler would throw an error and stop compiling).

Note that a function that takes a const reference can still make internal
by-value copies of an object and modify the internal copies:

```cpp
void makesInternalCopy(const MyData &data)
{
    MyData dataCopy = data;
}
```

This is possible because, in order to copy `data` to `dataCopy`, the compiler
only needs to read from `data` in order to copy its bytes into `dataCopy`.
Of course, in doing so, the function pays the full performance penalty of
copying the object byte-by-byte, which is onerous if the object is large.

That wraps up our discussion of types, pointers and memory addressing!
The key takeaways are as follows:

> _Types are an interpretation of byte offsets relative to a starting byte.
> Pointers are integers which store a starting byte.
> Variables are always initialized using a byte copy,
> but you can pass a pointer by value to share a reference._

## <a name="part2"></a> Idea 2: Resource Lifetime

You may have heard before that C++ has no automatic memory management.
There is no garbage collector to clean up memory after you're done with it:
whenever you use `new` to allocate an object, you must later use `delete` on
that object to release its memory, once you're done with it.
That's not to say that C++ doesn't help you manage your memory at all!
But to understand how C++ helps you manage memory, first we'll need to talk
about the types of memory that need managing.
There are two types of memory available to you when writing in C++:
stack memory and heap memory.

## <a name="part3"></a> Idea 3: Compilation Model

C++ supports a different compilation model than you may be familiar with.
Namely, C++ is designed to work with single-pass compilers,
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

> 1. _Types are an interpretation of byte offsets relative to a starting byte.
>    Pointers are integers which store a starting byte.
>    Variables are always initialized using a byte copy,
>    but you can pass a pointer by value to share a reference._
>
> 2. _You control when memory is allocated and freed.
>    Use language features to tie memory lifetime to execution scope
>    for maximum convenience.
>    Explicitly define in your program logic who owns which object(s)._
>
> 3. _C++ requires every symbol you reference to be declared or defined prior
>    to the time you use it.
>    Publish declarations in header files, so that any definition in a source
>    file can reference the symbols it needs._

Having grokked the concepts above, we hope you now have enough of a working
framework to fit in all the small details about writing C++ code.
You're not done with your journey, but you're off to a good start.
Best of luck!

_Find this guide helpful? harmful? Did you find a bug?
Please give any feedback by opening an issue on 
[this blog's git repo](https://github.com/davekilian/davekilian.github.com).
Or feel free to fix it yourself and open a pull request :)_

