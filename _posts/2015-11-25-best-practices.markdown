---
layout: post
title: Best Practices Gone Wrong
author: Dave
draft: true
---

When you write a program, you usually have lots of options, even if only a few are good.
Software 'best practices' are rules of thumb to help guide you to the good solutions, and avoid deceptive pitfalls.
These are invaluable both for examining your past designs in retrospect, and informing your new designs as time goes on.

But many well-accepted best practices come with caveats.
When faced with a choice between competing designs, it's easy to invoke a rule of thumb to pick a design, with no further consideration.
But invoking a best practice rule without considering the reasoning behind it can worsen your code.
Do this too many times, and your designs can degrade to the point of ummaintainability.

The main culprit is indirection.
Most software practices involve adding some type of indirection; as the old joke goes, "every problem in software design can be solved with another layer of indirection."
Of course, adding indirection isn't free:

* **Indirected code is harder to read.**
  The reader has to jump around between multiple locations to get the full picture.
  Jumping around breaks the reader's concentration, making it easier to miss subtle interactions.

* **Indirected code is harder to refactor.**
  Indirection adds an internal code contract between two or more parts of the implementation.
  Maintaining or reworking an internal contract is harder than just adding one.

Sometimes indirection is worthwhile despite these costs.
But that's not always the case, even when you apply a rule commonly accepted as a best practice!
In this post we'll go over a few common best practices, and note how they can backfire when applied to the wrong situation.

### Constants. Constants everywhere

Constants hide a literal value behind a variable name.
There are two good reasons to do this:

1. To assign a more descriptive name to an otherwise bizarre-looking value.
2. To link all uses of a value to a single code location, to be tweaked later.

Consider the following example:

```cpp
// (A)
int size = 1048576 * getFileSize();

// (B)
int size = BYTES_PER_MEGABYTE * getFileSize();
```

Clearly example (B) above is easier to read.
In example (B), we can tell that getFileSize returns a size in megabytes, and that we intend for size to be expressed in bytes.
In example (A), it isn't immediately clear where the number 1048576 came from.

So, best practice: use constants to make it easier to read your code.

One common mistake is to generalize to saying _every_ literal value should be tucked away behind a nicely-named constant.
This isn't true.
Using constants where they're not needed obfuscates your codebase.
In fairly common cases, abusing constants can also read to real code bugs.

Let's illustrate this an example.
Say we want to write a parser which can read XML documents like this:

```xml
<Books>
    <Book Title="A Game of Thrones" Author="George R. R. Martin" />
    <Book Title="American Gods" Author="Neil Gaiman" />
    <!-- ... and so on -->
</Books>
```

We can write a simple parser like this:

```cpp
struct book
{
    std::string title;
    std::string author;
};

std::vector<book> parse_doc(xml_doc *doc)
{
    std::vector<book> result;

    auto books = doc->select("/Books/Book");
    for (auto book = books.begin(); book != books.end(); ++book) {
        
        book item;
        item.title = (*book)->get_attribute("Title");
        item.author = (*book)->get_attribute("Author");

        result.push_back(item);
    }

    return result;
}
```

Ick -- this code has some bare literals mixed in with the logic!
Let's "improve" this code by hoisting those out as nicely-named constants:

```cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define TITLE_ATTRIBUTE_NAME "Title"
#define AUTHOR_ATTRIBUTE_NAME "Author"

std::vector<book> parse_doc(xml_doc *doc)
{
    std::vector<book> result;

    auto books = doc->select(BOOK_XPATH);
    for (auto book = books.begin(); book != books.end(); ++book) {
        
        book item;
        item.title = (*book)->get_attribute(TITLE_ATTRIBUTE_NAME);
        item.author = (*book)->get_attribute(AUTHOR_ATTRIBUTE_NAME);

        result.push_back(item);
    }

    return result;
}
```

We can even go all-out and make a 'constants' file which is global to the entire program:

**constants.h**

```cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define TITLE_ATTRIBUTE_NAME "Title"
#define AUTHOR_ATTRIBUTE_NAME "Author"
```

**parser.cpp**

```cpp
std::vector<book> parse_doc(xml_doc *doc)
{
    std::vector<book> result;

    auto books = doc->select(BOOK_XPATH);
    for (auto book = books.begin(); book != books.end(); ++book) {
        
        book item;
        item.title = (*book)->get_attribute(TITLE_ATTRIBUTE_NAME);
        item.author = (*book)->get_attribute(AUTHOR_ATTRIBUTE_NAME);

        result.push_back(item);
    }

    return result;
}
```

In the first example, you could look at a copy of books.xml and parser.cpp, and be sure parser.cpp was implemented correctly.
You could even reverse-engineer the structure of books.xml just by reading parser.cpp.
In the last example, you can't -- you'd have to jump back and forth between parser.cpp and constants.h to fully get what's happening.

Using constants in this way doesn't play to either of the strengths of constants:

1. The strings were already descriptive; aliasing them with a constant name didn't tell us anything we didn't already know.

2. Each string literal was only referenced once to begin with.
   Changing node names is an equally time-consuming task whether or not we employed constants.

There's also a more insidious flaw lurking in the final example.
Because the constants are declared globally to the entire program, there's a hazard someone might later pick up those constants for an unrelated scenario.
Once unrelated scenarios are linked by the use of global constants, changes in either scenario can unintentionally break the other.

To continue the example, say you wrote your book XML parser like in the final example above.
Then later, a design requirement comes in for a movie XML parser.
The person who writes the movie parser reuses `TITLE_ATTRIBUTE_NAME`, because movies also have titles.
Now both the book parser and the movie parser depend on `TITLE_ATTRIBUTE_NAME` having the value `"Title"`.

Later on, a design change comes in for the book parser, to rename the `Title` attribute to `Name`.
Unfortunately, the title attribute for books and movies have been coupled together by the constant.
The developer making this change will likely rename the title constant, inadvertently breaking the unrelated movie parser.

Whether they already knew about the coupling or finds out about it due to a compile-time or runtime break, the developer making this change still has to decouple the two scenarios somehow.
With the current design, the changes look awkward.

They'll probably either look something like this:

```cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define BOOK_TITLE_ATTRIBUTE_NAME "Name"
#define MOVIE_TITLE_ATTRIBUTE_NAME "Title"
#define AUTHOR_ATTRIBUTE_NAME "Author"
```

or this:

```cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define TITLE_ATTRIBUTE_NAME "Title"
#define NAME_ATTRIBUTE_NAME "Name"
#define AUTHOR_ATTRIBUTE_NAME "Author"
```

But we could have avoided all this trouble if the constants were never shared or made global in the first place.
Really the book XML and movie XML parsers are separate, and should be maintained separately.

> **Aside: Really going overboard with constants**
>
> Once, in all my code-reading adventures, I stumbled upon a certain tool.
> With this tool, a programmer could write code using inlined literal values.
> Then the programmer could run this tool on their source code.
> The tool would find literal values, and replaced them with automatically-generated constants.
>
> For example, if your code had a string like this:
>
>     "http://www.example.com"
>
> The tool would replace the literal with a constant like this:
>
>     __http_colon_slash_slash_www_dot_example_dot_com__
>
> Sure the generated name is just a harder-to-read version of the original value.
> But what's truly awesome is, if the underlying value is ever changed, then the constant also needs to be changed, and all references need to be fixed!
>
> By the time I found this, the author of this tool was long gone, with no way of being contacted.
> I still haven't the foggiest what they thought they were getting out of this arrangement.

In summary, here are two bits of advice for using constants in your future code:

1. Before you hoist a value to a constant, ask yourself: does this value need a better name?
   Or am I going to reference it in multiple places?
   If the answer is no for both, consider skipping the constant and just using the value directly.

2. When you do decide to use constants in your code, scope them as tightly as possible.
   The more globally you define the constant, the greater risk you run of leaking the constant into unrelated scenarios.
   An necessary constant only reduces readability, but an overscoped constant can lead to real bugs.

## Interfaces. Interfaces everywhere

Linking two pieces of code together is called *coupling*.
Most developers know that 'coupling' is a naughty word.
However, when you write software, you expect all the code you write to be called at one point or another.
That means one way or another, each method has to be coupled to something else.

I'm going to assume you know what an interface is in object-oriented design, and skip the intro example :-).
Interfaces are a tool for making coupling explicit in your code.
One of their nicer features is to be able to write a contract once, and couple multiple things to it.
By writing the calling code against the interface, you can swap out the callee as desired.

But as before, indirection strikes again.

* **Interfaces make it hard to reason about specific flows**

---

More topics:

* Speculative extensibility
* Abusing dependency injection/inversion of control
* Hiding code instead of abstracting
* The minimum code spanning tree ^(TM)

Takeaway:

* Software should be stable first, and readable second. All else is ancillary
* Simple software is readable
* Simple software is easy to reason about
    * Easier to catch bugs early
    * Easier to test
    * Easier to debug
* Simple software is easier to reuse
* Simple software is easier to refactor
* Simple software is easier to extend
