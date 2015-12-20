---
layout: post
title: decltype Your Function Pointers
author: Dave
---

Herein is a description of a particularly tricky bug I recently fixed.
The issue was a crash which appeared to violate C's function calling / pass-by-value semantics.
We'll discuss the problem, what caused it, and how to use C++11's `decltype` keyword to avoid causing this problem in your own codebase.

## The Problem

The original issue was found in a long-running batch operation implemented in C++.
The operation was crashing with an invalid pointer read near the very end.
Typically, when you see an invalid pointer read, these are good places to start looking:

* Was a pointer used before it was initialized?
* Was a pointer used after it was freed?
* Was the pointer freed twice?

We got the crash under a debugger, and it turned out to be none of the above.
What we saw instead was much more strange.
Say this was the crashing function:

```cpp
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
```

In the debugger, we saw a crash at line 8 due to dereferencing `data`, which was a null pointer.
That's particularly weird, though, because `data` was allocated successfully and was perfectly valid on line 6 when we called `process()`.
Stepping through the program, we could see `data`'s value changed to NULL as soon as `process()` returned.

For background, in C functions pass all parameters by value.
As such, `process()` should not have access to `data` in `main`'s stack frame; yet here it was, clearly modifying it.
How can that be?

## Stack Balancing

After some other angles didn't pan out, we eventually dropped the debugger into assembly mode to trace the buggy pointer.
That's when we saw something fishy: 

1. While `main()` was executing, `data` was stored in register `esi`
2. When we called `process()`, the compiler generated a `push esi` to save the value to the stack
3. When `process()` returned, the compiler generated a `pop esi` to restore the `esi` value for `main()`.
4. But the stack pointers were different between the `push` and `pop` instructions

Since the stack pointer was wrong for the `push` and `pop` instructions, the wrong value was restored for `esi` (a.k.a. `main!data`).
Indeed, by manually dumping the data on the stack, we could see the correct value of `esi` just four bytes away, right where we initially left it.

Someone had pushed more bytes to the stack then they had popped.
This class of issue is known as an _unbalanced stack_.
The term may be new to you, as this problem is not very common nowadays :-)

So came the question: how did the stack get unbalanced?

## Calling Conventions

To answer that question, we need to review the assembly-level convention for passing function parameters in C.
Specifically, by default when you call a function in C:

* The **caller** pushes all arguments to the stack
* The **callee** pops all arguments from the stack

Even though the caller pushes and the callee pops, the number of arguments is the same on both ends, so the stack ends up balanced (usually).
Your C compiler uses these rules in most cases, with only a few exceptions:

* When you use `varargs`, the caller cleans up, because only the caller knows how many arguments it pushed to begin with
* You can explicitly tell the compiler to use other calling convetions, and it will generally comply.

## Gone Awry

After hours of searching, we eventually found the function call which was messing up the stack.
The binary in question was using Windows's `LoadLibrary` and `GetProcAddress` APIs to manually load a DLL and load a function by pointer.
The code looked something like this:

```cpp
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

```

Due to an incorrect merge while integrating source control branches, the `typedef` had fallen out of sync with the real implementation, which looked like this:

```cpp
void MyFunction(int arg)
{
    // ...
}
```

Remember how we said in the standard C calling convention, the caller pushes the arguments and the callee pops them?
In this case, the caller and the callee had different opinions about the number of arguments to manage:

* The caller thought the function took two arguments, and pushed them both (a total of 8 bytes)
* The callee thought the function took only one arg, and popped just one (a total of 4 bytes)
* After that function returned, the stack pointer was off by one argument (4 bytes)

Unfortunately for us, the program had just happened to work for a while after we had unbalanced the stack, causing the eventual failure nowhere near the root cause.

## Avoiding the Problem

Even once we knew what the problem was, it took a lot of time and brute-force effort to find the root cause.
There's no silver bullet for tracking down the cause of an unbalanced stack as far as we know; we just walked through assembly and wathced for something nonsensical to happen.

Since tracking down these issues is hard, a better bet is to avoid creating this issue for yourself in the first place :-)

One elegant way to do this is with headers and `decltype`.
The issue here was that we had two definitions of `MyFunction`: the real implementation, and the `typedef` the caller was using.
The compiler couldn't help us catch this because the compiler never could see both at the same time.

Starting in C++11, we can link the definitions together, like so:

* Declare the function in a common header shared by both the implementing DLL and the module calling `LoadLibrary`.
* In the implementation, `#include` the header and define the function normally
* In the caller module, `#include` the header and `typedef` the function using `decltype`

For example, we would first define **MyFunction.h**:

```cpp
void MyFunction(int arg1, int arg2);
```

Then we could implement it in **MyFunction.cpp**:

```cpp
#include "MyFunction.h"

void MyFunction(int arg1, int arg2)
{
    // ...
}
```

Finally, we could load that API in **ExternalCaller.cpp** like this:

```cpp
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
```

If you haven't seen it before, `decltype` is a new C++ keyword defined in C++11.
It is a language construct which evaluates to the type of the expression in parentheses.
In this case, `&MyFunction` evaluates to the function pointer signature for `MyFunction`, which means:

```cpp
typedef decltype(&MyFunction) MyFunctionPtr;
```

evaluates to just what we'd expect:

```cpp
typedef void (*MyFunctionPtr)(int arg1, int arg2);
```

Now everyone is linked together:

* If you modify either .cpp without modifying the .h, the compiler will complain when compiling that .cpp
* If you modify the .h and forget to modify one of the .cpp files, the compiler will complain when compiling that .cpp

And know that, hopefully you'll never need to debug this issue yourself :-)
