---
layout: post
title: Fixing Function Pointers with decltype
author: Dave
---

This post is about a particularly tricky bug I recently fixed.
The issue was an application crash which appeared to violate C's syntax rules.
In the end, it turned out to be an issue with function pointer declarations in our app.
In this post we'll discuss the problem, what caused it, and how to use the C++11 `decltype` keyword to prevent this bug in your own codebase.

## The Problem

The original issue was found in a long-running batch operation implemented in C++.
The operation was crashing with an invalid pointer read near the very end.
Typically, when you see an invalid pointer read, these are good places to start looking:

* Was a pointer used before it was initialized?
* Was a pointer used after it was freed?
* Was the pointer freed twice?

We got the crash under a debugger, and it turned out to be none of the above.
What we saw instead was much more strange.

Let's say this was the crashing function:

~~~cpp
 1 bool main()
 2 {
 3     mystruct *data = new mystruct;
 4     if (!data) return false;
 5 
 6     process(data);
 7     
 8     if (data->myvalue > 0) {
 9         // some additional processing
10     }
11 
12     delete data;
13     return true;
14 }
~~~

In the debugger, we were seeing the app crash at line 8, while dereferencing `data->myvalue`.
It turns out `data` was `nullptr` at that point.

What's particularly strange about this invalid pointer read is we know `data` was allocated successfully; otherwise, we would have returned from `main()` on line 4.
We could verify the pointer was valid up until line 6 when we called `process()`, but stepping through the program, we could see the value changing as soon as `process()` returned.
Then we would crash the first time we read the null pointer (line 8).

It looked like `process()` was nulling the `data` pointer, but that shouldn't have been possible: C functions pass parameters by value, not by reference.
Yet we could clearly see the value change when `process()` returns.

How can that be?

## Stack Balancing

After some other angles didn't pan out, we ended up dropping the debugger into assembly mode to trace the buggy pointer.
We saw the pointer get initialized in register `esi`.
During the call to `process()` from `main()`, we saw the compiler correctly generate `push esi` and `pop esi` instructions to store the value on the stack.
Everything was looking fine so far.

Then we noticed something fishy: the stack pointer value was different for the `push esi` and `pop esi` instructions, meaning we were restoring _the wrong value_ into `esi`. 
That's why `data` magically changed into a null pointer when `process()` returned.

Since the stack pointers weren't lining up, we could conclude that someone had pushed more bytes to the stack than they had popped.
This class of issue is called an _unbalanced stack_.
That term was a new one for me; I suppose this type of problem isn't very common nowadays :-)

So came the question: how did the stack get unbalanced?

## Calling Conventions

To answer that question, we need to review the assembly-level convention for passing function parameters in C.
Specifically, when you call a function in C:

* The **caller** pushes all arguments to the stack
* The **callee** pops all arguments from the stack

Even though the caller pushes and the callee pops, as long as the number of arguments is the same on both ends, the stack ends up balanced.
Your C compiler uses these rules in most cases, with only a few exceptions:

* When you use varargs, the caller has to clean up, because only the caller knows how many arguments it pushed to begin with.
* You can explicitly tell the compiler to use other calling convetions, and it will generally comply.

## Gone Awry

Once we had figured out we were dealing with an unbalanced stack, we spent several a lot more time searching for the problem.
Eventually, we found the problematic function call nested a half dozen function calls below `process()`.

The function we were calling had been loaded using Windows's `LoadLibrary` and `GetProcAddress` APIs, which allow you to manually load a DLL and extract a bare pointer to a specific function.
The code looked something like this:

~~~cpp
typedef void (*MyFunction)(int arg1, int arg2);

// ...

HMODULE module = LoadLibrary("thedll.dll");
if (module) {
    MyFunction *theFunc = reinterpret_cast<MyFunction*>(
            GetProcAddress(module, "MyFunction")
        );

    if (theFunc != nullptr) {
        theFunc(1, 2);
    }
}

~~~

Someone had recently changed the function being called, and someone else had handled a merge conflict from that change.
The merge conflict was done incorrectly, causing the `typedef` to fall out of sync with the real implementation.
Compare the `typedef` above with the definition below:

~~~cpp
void MyFunction(int arg)
{
    // ...
}
~~~

Remember how we said in the standard C calling convention, the caller pushes the arguments and the callee pops them?
In this case, the caller and the callee had different opinions about the number of arguments to manage:

* The caller thought the function took two arguments, and pushed 8 bytes
* The callee thought it only took one argument, and popped 4 bytes
* In the end, the stack pointer was 4 bytes from balanced

Unfortunately for us, the program had just happened to work for a while after we had unbalanced the stack.
The program returned all the way to `main()` where we eventually failed, nowhere near the root cause.

## Avoiding the Problem

This is not the kind of bug you want to find in a production system, or under any kind of time pressure.
We figured out the stack was unbalanced pretty quickly, but from there we didn't have a good way to figure out who had unbalanced the stack: we had to manually guess and test until we found the culprit.

Luckily, you can avoid these problems by doing a little engineering up front.

The trick is to get the compiler to catch a mismatch between the typedef and the function definition at compile time.
This didn't happen in the example above because the compiler never saw both the typedef and the definition at the same time.
However, we can link the two together using a shared header and the `decltype` keyword, like so:

* Declare the function in a common header shared by both the implementing DLL and the module calling `LoadLibrary`.
* In the callee module, `#include` the header and implement the function
* In the caller module, `#include` the header and declare the function pointer using `decltype`.

For example, we would first define **MyFunction.h**:

~~~cpp
void MyFunction(int arg1, int arg2);
~~~

Then we could implement it in **MyFunction.cpp**:

~~~cpp
#include "MyFunction.h"

void MyFunction(int arg1, int arg2)
{
    // ...
}
~~~

Finally, we could load that API in **ExternalCaller.cpp** like this:

~~~cpp
#include "MyFunction.h"

typedef decltype(&MyFunction) MyFunctionPtr;

bool main()
{
    HMODULE module = LoadLibrary("thedll.dll");
    if (module) {
        auto func = reinterpret_cast<MyFunctionPtr>(
                GetProcAddress(module, "MyFunction")
            );

        if (func != nullptr) {
            func(1, 2);
            return true;
        }
    }

    return false;
}
~~~

If you haven't seen it before, `decltype` is a new C++ keyword defined in C++11.
It is a language construct which evaluates to the type of the expression in parentheses.
In this case, `&MyFunction` evaluates to the function pointer signature for `MyFunction`, which means:

~~~cpp
typedef decltype(&MyFunction) MyFunctionPtr;
~~~

evaluates to just what we'd expect:

~~~cpp
typedef void (*MyFunctionPtr)(int arg1, int arg2);
~~~

This way we've linked the caller's typedef with the callee's implementation: both must match the declaration in the shared header.
If we decide to change the implementation, all the necessary changes become compiler breaks, which are much easier to find.

Knowing this trick, hopefully you'll never need to debug an issue like this yourself :-)
