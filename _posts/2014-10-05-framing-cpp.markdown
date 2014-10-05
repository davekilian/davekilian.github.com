---
layout: post
title: 3 Big Ideas Behind C++
author: Dave
---

Learning C++ as a beginner can seem daunting given the enormity of the
language.
However, as C++ is both old and ubiquitous, there's already tons of material
online to help you get started.
So then why write another guide?

This one aims to be a little different from most C++ intros.
We won't spend much time on syntax - you can find all that better-documented
elsewhere
[(1)](http://en.wikipedia.org/wiki/C%2B%2B)
[(2)](http://www.cplusplus.com)
.
Instead, this guide is a fast-paced lecture-style treatment of the 'Big Ideas'
behind C++.
These ideas are the fundamental reasons C++ is still widely used today, despite
all the progress that has been made in the field of programming languages.
The aim of this guide is to ground you so that, afterward, you should be able
read up on nitty-gritty details and work them into the bigger picture.

The only requirements are some background in a more modern object-oriented
language with C-style syntax (Java, C# and Go all qualify), and completion of
a basic syntax-oriented tutorial for C++. 
You don't need to know C, nor will we cover C.

Without further ado, here are the big ideas:

1. [Close-to-the-Metal Type System](#part1)
2. [Resource Lifetimes Tied to Execution Flow](#part2)
3. [Single-Pass Compilation Model](#part3)

## <a name="part1"></a> 1: Close-to-the-Metal Type System

One of the commonly cited reasons people use C++ is that it provides 'low-level
memory access.'
To understand what that means, let's reflect for a moment on programming type
systems.

You've probably heard before that everything stored by a computer, whether in
RAM, on a hard disk, in a CPU register, or barreling down an Ethernet cable, is
just binary.
One of the chief jobs of a programming language is to insulate you from this.
To you, `int`s are just mathematical integers, on which you can perform
arithmetic.
`string`s are just bits of text that can be sliced, diced and spliced.
But every time you manipulate a variable, you're really just changing a number
that's stored as binary in some cell in RAM.
Different 'types' are just different ways of interpreting these numbers.

Like most programming languages, C++ provides an abstraction over bytes as
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
* `long` is typically 32-64 bits, depending on your processor architecture
* `long long` is typically 64 bits
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

ends up stored something like this in memory:

<table style="width:100%">
    <tr>
        <td style="border: 1px solid #777; text-align: center">0</td>
        <td style="border: 1px solid #777; text-align: center">1</td>
        <td style="border: 1px solid #777; text-align: center">2</td>
        <td style="border: 1px solid #777; text-align: center">3</td>
        <td style="border: 1px solid #777; text-align: center">4</td>
        <td style="border: 1px solid #777; text-align: center">5</td>
        <td style="border: 1px solid #777; text-align: center">6</td>
        <td style="border: 1px solid #777; text-align: center">7</td>
    </tr>
    <tr>
        <td style="border: 1px solid #777; text-align: center" colspan="4">SomeData</td>
        <td style="border: 1px solid #777; text-align: center" colspan="4">MoreData</td>
    </tr>
</table>

If you had an instance of `MyData` and set its `MoreData` field to some value
(say, 7), the compiler would generate machine code that says:

* Start at the byte address of the `MyData` instance
* Skip forward 4 bytes
* Set the next 4 bytes to `0x00 0x00 0x00` and `0x07`, respectively.

One result of this strategy is that type information is only known at compile
time:
the compiler generates machine code that, at runtime, used a fixed offset and
byte count to assign a value to some variable.
Since the type information is gone at runtime, C++ has very primitive
reflection abilities.
Another result of this strategy is accessing some field of an object is dirt
cheap, especially compared to those modern languages which must look up every
field in a hash table!

C++'s object representation works recursively for nested user-defined types.
So if we nested the struct above in another one as follows:

```cpp
struct Complex
{
    int Before;
    MyData Data;
    int After;
};
```

the byte representation would be similarly nested:

<table style="width:100%">
    <tr>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">0</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">1</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">2</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">3</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">4</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">5</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">6</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">7</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">8</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">9</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">A</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">B</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">C</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">D</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">E</td>
        <td style="border: 1px solid #777; text-align: center; width: 6.25%">F</td>
    </tr>
    <tr>
        <td style="border: 1px solid #777; text-align: center" colspan="4">Before</td>
        <td style="border: 1px solid #777; text-align: center" colspan="4">Data.SomeData</td>
        <td style="border: 1px solid #777; text-align: center" colspan="4">Data.MoreData</td>
        <td style="border: 1px solid #777; text-align: center" colspan="4">After</td>
    </tr>
</table>

Now if you set `Complex.Data.MoreData = 7`, the compiler would start at the
address of the `Complex` instance, jump 8 bytes forward, and set that 32-bit
integer to 7.

Since every C++ object has a well-defined byte structure, C++ treats both
primitives and user-defined data types as _value types_; that is, when you
assign one instance of a certain object to another instance, the compiler
simply does a byte-by-byte copy from one instance to the other.
Subsequently modifying either of those objects does not affect the other.

```cpp
MyData a;
MyData b;

a.SomeData = 1234;
a.MoreData = 5678;

assert(a.SomeData == 1234);
assert(a.MoreData == 5678);

b = a;

assert(b.SomeData == 1234);
assert(b.MoreData == 5678);

a.SomeData = 4321;
a.SomeData = 8765;

assert(a.SomeData == 4321);
assert(a.MoreData == 8765);

assert(b.SomeData == 1234); // Important: b has not changed,
assert(b.MoreData == 5678); // even though a has been changed.
```

Similarly, each time you call a function, C++ copies in each argument by value,
using the same byte-by-byte copy technique.
As a result of this, a function which modifies one of its arguments does not
affect the value that the caller passed in:

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

And, like arguments passed into the function, return values are passed out of
the function using a byte copy as well.
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
One common criticism of C++ is the difficulty of diagnosing performance
problems caused by excessive copying.
It's not too unusual for objects to be dozens of bytes long.
A program that is constantly passing hundreds of bytes into every function call
is liable to be _uniformly slow_: there is no obvious bottleneck, because CPU
time is being wasted every time a function is called.
The fix for excessive copying is to pass objects by reference instead.
To do that, we'll need to start using _pointers_.

Like all data types, a pointer is just some binary with a special meaning.
C++ pointers work a lot like integers: you can assign them arbitrary numeric
values and do arithmetic on them.
Unlike `int`s though, pointers can be _dereferenced_.
When you dereference a pointer, the compiler treats the pointer's value as a
byte address in memory.
If you picture all of RAM as a giant array of bytes (regardless of how your
program is using these bytes), a pointer is just an index into this byte array.

C++ provides the pointer-dereferencing operator `*` to allow you to dereference
a pointer, in order to read or write the value at the RAM address stored in
that pointer:

```cpp
void pointerTest(int *intPtr, MyData *dataPtr)
{
    int num = *intPtr;
    num += 1;
    *intPtr = num;

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
The answer is basically the same; only the first step is different:

* Start at the byte address stored in the `data` pointer
* Skip forward 4 bytes
* Set the next 4 bytes to `0x00 0x00 0x00` and `0x07`, respectively.

What's important to realize is that _we never actually checked that what we're
doing makes sense_ :-).
When you say `data->MoreData = 7`, the compiler generates machine code that
does some pointer arithmetic and then copies some bytes.
It does not (and, in fact, cannot) verify that the address you're writing to is
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
get hardcoded by the compiler.
Although this is about as fast as you can get, you also can't be sure that the
pointer contains a meaningful memory address.
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

Back to our original discussion about passing function arguments, we can now
use pointer types to pass a `MyData` instance by reference.
Note that, internally, what actually happens is the compiler copies in the
pointer itself by value; but since both copies of the pointer contain the same
memory address, the same object gets modified when either copy of the pointer
is dereferenced:

```cpp
void acceptPointer(MyData *data)
{
    data->SomeData = 1234;
    data->MoreData = 5678;
}
```

We can accomplish the same thing using reference types instead of pointers:

```cpp
void acceptReference(MyData &data)
{
    data.SomeData = 1234;
    data.MoreData = 5678;
}
```

When calling functions, it's often better performance-wise to pass in a pointer
or reference to an object rather than copying the whole object by value. 
In fact, this is true pretty much any time `sizeof(Object) > sizeof(Object*)`.
Copying a reference rather than the full object reduces the overhead induced in
calling the function.
Unfortunately, though, passing by reference opens the door to functions
modifying the caller's passed-in objects as a side effect.
To work around this, a common idiom in C++ is to accept a `const` reference to
an object as the input to a function:

```cpp
void acceptConstRef(const MyData &data)
{
    // data.SomeData = 1234; <-- causes a compilation error

    MyData localCopy = data;
    localCopy.SomeData = 4321;
}
```

With this idiom, we can reap the performance benefits of passing an object by 
reference, while also guaranteeing that calling the function will not modify 
the original object.

Note that a function that takes a const reference to an object can still make a
local copy of the object (see `localCopy` in the example above).
This is allowed because all that's needed to initialize `localCopy` is to read
`data` byte by byte - no modification occurs.
Even though creating a local copy of an object negates any performance benefit
we gained by passing in the original argument by reference, it's rarely
necessary to make local copies, as functions tend to only read most of their
input arguments.

That wraps up our discussion of types, pointers and memory addressing!
The key takeaways are as follows:

> _Types are an interpretation of byte offsets relative to a starting byte.
> Pointers are integers which store a starting byte.
> Variables are always initialized using a byte copy,
> but you can pass a pointer by value to share a reference._

## <a name="part2"></a> 2: Resource Lifetimes Tied to Execution Flow

One common description of C++ is that it has no automatic memory management.
Whenever you use `new` to allocate an object, you must later call `delete` to
free the memory when you're done with it.
If you forget to delete this memory, it stays marked as allocated, but your
program stops using it (in other words, the memory gets _leaked_).
If you leak enough memory, eventually your program will run out.
And once it fully runs out, it basically has one option: crash.

Of course, all but the smallest programs require frequent memory allocation.
So why don't C++ programmers all go crazy trying to track their `new`s and
`delete`s?
The answer is simple: C++ doesn't require you to manage _all_ of your memory.
In fact, you can have C++ handle almost all the memory you use;
there are only a few cases where you need to manage memory yourself.

If you've been following this guide so far, you've already made plenty of
automatically managed allocations.
To have the compiler manage a variable's memory, just declare it directly:

```cpp
void stackAllocation()
{
    int value;   // Allocates sizeof(int) bytes on the stack
    MyClass obj; // Allocates sizeof(MyClass) bytes
}
```

These allocations are local to the _scope_ in which they appear.
In C++, a scope is defined by the `{ }` characters: `{` denotes the beginning
of a scope, and `}` marks the end of the matching scope:

```cpp
void aFunction()
{
    // function-level scope

    while (true) {
        // sub-scope
    }
}
```

This type of allocation is known as _stack allocation_, because the compiler
allocates memory for variables on the _call stack_.

The call stack is a data structure implemented in your computer's hardware.
It's called the _call_ stack because it keeps track of function calls, allowing
a callee function to return the context of the caller.
It's called the call _stack_ because the last bytes allocated are the first
bytes freed.
This is in line with function calling, where the last function called must be
the first function to return.

In C++, usage of the call stack roughly corresponds to program scope.
Memory for a variable within a `{ }` pair is allocated allocated when control
flow reaches the beginning of the scope (`{`), and is deallocated when control
reaches the end of the scope (`}`).

It might seem strange to mix a function's local variables with the bookkeeping
information needed to return from a function, but doing so has some nice
benefits.
For example, when a variable is allocated on the stack, the compiler knows
exactly when the variable is and is not accessible, and can thus allocate and
deallocate memory for that variable by itself:

```cpp
void someFunction()
{
    // (A)

    if (true) { // (B)
        int num = 1;
    } // (C)

    // (D)
}
```

At point (A) in the example above, the variable `num` is not yet accessible, so
no memory needs to be allocated for it yet.
At point (B), we enter the scope in which `num` is declared.
At this point, the compiler will allocate `sizeof(num)` bytes on the call
stack to hold the value for `num`.
THen, at point (C), we exit the scope that contains `num`.
At this point, the compiler deallocates the memory for `num`.
At point (D), `num` is no longer accessible, and there is no longer memory
allocated for `num`.

There's a second benefit to stack allocation as well: on modern CPU
architectures, the compiler can allocate room on the stack with just one
instructions.
This is about as fast as memory allocation can get :-)

Unfortunately, there are some drawbacks to stack allocation.
The compiler _always_ deallocates memory once its parent scope exits.
If you need to return a value to a caller outside that scope, you have to incur
the cost a full byte-by-byte copy.
Also, the call stack generally has a fixed size (usually a handful of
megabytes).
If you try to stack-allocate more memory than the stack has room for, you end
up _overflowing_ the stack, which crashes the program.

_Heap allocation_ addresses the shortcomings of stack allocation.
You can allocate on the heap using `new` and `delete`, as previously mentioned:

```cpp
void heapAllocation()
{
    MyClass *obj = new MyClass;
    delete obj;
}
```

The primary difference between the stack and the heap is that memory allocated
on the heap is not freed until your program uses the `delete` operator on a
`new`-allocated pointer.
This has more or less the inverse pros and cons as stack allocation:

* Unlike the stack, the heap makes it easy to return memory from a local scope
  to the parent scope.
  In stack allocation, this was impossible without a byte-by-byte copy.

* The size of the heap is more or less unlimited, constrained only by the
  amount of physical storage your machine has.
  In stack allocation, we were limited to a handful of megabytes.

* The compiler does not know when you're done with heap memory; you have to
  tell it explicitly using the `delete` oeprator.
  In stack allocation, we let the compiler clean up automatically.

* Allocating on the heap is relatively slow, since the heap allocator must run
  a complex algorithm to find a free spot.
  In stack allocation, we could allocate with a single instruction.

The key takeaway is that allocating on the stack is both fast and hard to mess
up.
You want to use the stack whenever possible.
You only want to allocate on the heap if the object must survive after its
original scope ends, and/or if the object is very large.

Let's take another look at the heap allocation snippet we showed earlier:

```cpp
void heapAllocation()
{
    MyClass *obj = new MyClass;
    delete obj;
}
```

There are two allocations in the function above.
Can you spot them both?
The answer is in the next paragraph.

The first allocation (chronologically speaking) is the trickier of the two to
spot.
The compiler first stack-allocates a `MyClass *` (a pointer to a `MyClass`
object).
Second, the `new` operator allocates a `MyClass` instance on the heap, and
returns a `MyClass *` that points to that instance.
The stack-allocated `MyClass *` is initialized to a reference to the
heap-allocated `MyClass`.

When the `delete` operator is used in the example above, the `MyClass` instance
on the heap gets deallocated.
It's important to note that the `MyClass *` on the stack is still allocated, 
and now points to an invalid memory address!
The `MyClass *` gets automatically cleaned up once `heapAllocation()` returns,
ending the scope in which `obj` is defined.

The following rewrite of the example above makes it a little clearer what's
going on:

```cpp
void heapAllocation()
{
    MyClass *obj; // Stack-allocate MyClass *
    obj = new MyClass; // Heap-allocate MyClass
    delete obj; // Heap-deallocate MyClass
} // Stack-deallocate MyClass *
```

So far we've discussed 'use-after-free' bugs with heap-allocation, a class of
bug where a program continues to dereference a pointer after the value the
pointer contained has already been freed.
There is another similar type of bug in heap allocation called a _double-free_. 
C++ requires that any pointer returned by `new` must be `delete`d exactly once.
A double-free occurs when the program tries to `delete` a pointer whose value
has already been `deleted`.

C++ programmers deal with the risk of double-frees by employing the concept of
'ownership'.
C++ itself does not have a notion of ownership; instead, ownership rules are
usually communicated through function-level documentation comments.
The function or object which 'owns' a pointer is responsible for freeing it.
This can be done either by directly calling `delete`, or by transferring
ownership to another function or object, which will then take on the onus of
deleting it later.

A common ownership pattern is for an object to own a pointer.
The pointer is a member of the object, and gets deleted by the object in its
destructor.
That way, whenever the object gets deleted, the data it owns also gets deleted.
Refer to the following example:

```cpp
class Owner
{
private:
    MyData *m_data;

public:
    Owner() 
        : m_data(NULL)
    { }

    ~Owner()
    {
        if (m_data != NULL) {
            delete m_data;
            m_data = NULL;
        }
    }

    // Returns the MyData object tracked by this Owner.
    // The caller does not own the MyData * returned and must
    // not release its memory.
    MyData *getData()
    {
        return m_data;
    }

    // Sets the MyData object tracked by this Owner.
    // This object will take ownership of the given MyData instance.
    void setData(MyData *data)
    {
        if (m_data != NULL) {
            delete m_data;
        }

        m_data = data;
    }
};
```

In the example above, the `Owner` object 'owns' the `m_data` pointer by
convention.
The `getData()` method is documented as not transferring ownership to the
caller, meaning the caller must not `delete` the return value.
The `setData()` method is documented as transferring ownership of the pointer
to the `Owner` object, meaning the `Owner` is taking on the onus of
calling `delete` in the future, and that, again, the caller must not `delete`
the value being passed in.

The `Owner` object above lets us do something interesting: even though `Owner`
owns a heap-allocated pointer to a `MyData` object, `Owner` itself can be stack
allocated:

```cpp
void someFunction()
{
    Owner own;
    own.setData(new MyData);
}
```

When control flow reaches the `}` at the end of `someFunction()` above, `own`
(which is stack allocated) goes out of scope.
This causes the destructor `own.~Owner()` to get called, which in turn
heap-deallocates `own`'s `m_data` instance _before_ the `own` itself is
stack-deallocated.
Thus, even though the `MyData` instance in the example above is heap-allocated,
we're still able to tie its lifetime to control flow, just like stack-allocated
variables!

The `Owner` object above exhibits the basis for a type of C++ object called a
_smart pointer_.
Smart pointers are typically stack-allocated, but track a single heap-allocated
'inner' pointer.
When the outer object goes out of scope, C++ calls the smart pointer's
destructor, which in turn heap-deallocates the inner pointer.
This is the pattern exhibited by the `Owner` object above.
Unlike `Owner`, real smart pointer objects typically use advanced features like
templates and operator overloading so that the smart pointer object behaves as
much as possible like the inner pointer type.

In summary,

> _You control when memory is allocated and freed.
> Use destructors to tie memory lifetime to execution scope for maximum 
> convenience.
> Explicitly define in your program logic who owns which object(s)._

## <a name="part3"></a> 3: Single-Pass Compilation Model

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
How can we compile multiple files into a single program?
In C++, programs are compiled in two steps:

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
The fully preprocessed text of this file is as follows, and we've already
discussed why this doesn't compile:

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

In summary, here are the three core concepts this guide boils down to:

> 1. _Types are an interpretation of byte offsets relative to a starting byte.
>    Pointers are integers which store a starting byte.
>    Variables are always initialized using a byte copy,
>    but you can pass a pointer by value to share a reference._
>
> 2. _You control when memory is allocated and freed.
>    Use destructors to tie memory lifetime to execution scope for maximum 
>    convenience.
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

