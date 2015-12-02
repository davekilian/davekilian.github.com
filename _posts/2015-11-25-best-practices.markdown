---
layout: post
title: Best Practices Gone Wrong
author: Dave
draft: true
---

"Best practices" are rules to help design better software and avoid common pitfalls.
When you apply a best practice to your designs, it's important to always keep in mind _why_ something is considered a best practice.
Many common-sense rules, when applied to the wrong problems, actually degrade the quality of your codebase.

There's an old saying that goes, "every problem in software design can be solved with another layer of indirection."
Many best practices add some form of indirection to your codebase.
But adding indirection to your code doesn't come for free:

* **Indirected code is harder to read.**
  The reader has to jump around between multiple locations to get the full picture.
  Jumping around breaks the reader's concentration, making it easier to miss subtle interactions.

* **Indirected code is harder to refactor.**
  Indirection adds an internal code contract between two or more parts of the implementation.
  Maintaining or reworking an internal contract is harder than simply adding one.

If applying indirection provides benefits to offset these costs, then it's the right thing to do.
However, you have to consider benefits on a case-by-case basis.
Applying "best practices" to the wrong situations or over-broadly gets you no benefit from indirection.
If you're not benefitting from indirection, then the only effect is these costs hurting you.

In this post we'll take a look at a couple common anti-patterns which crop up from applying common best practices to the wrong problems.

### Constant Obfuscation

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

> ### Aside: Really going overboard with constants
>
> Once, in all my code-reading adventures, I stumbled upon a tool for generating constants.
> With this tool, a programmer could write code using inlined literal values.
> Then the programmer could run this tool on their source code.
> The tool would scan the code for bare literals, replacing each with an automatically-generated constant.
>
> For example, if your code had a string like this:
>
>     "http://www.example.com"
>
> The tool would replace the literal with a constant like this:
>
>     __http_colon_fwdslash_fwdslash_www_period_example_period_com__
>
> This is super neat because it violates both of our reasons for using constants:
>
> 1. **Use constants to assign a more descriptive name to an otherwise bizarre-looking value.**
>    The automatically generated name is clearly harder to read than the plain value!
>
> 2. **Use constants to link all uses of a value to a single code location, to be tweaked later.**
>    Since the name of the constant is a derivation of its value, if you change the value, you'd also need to rename the constant and fix all references.
>    So you haven't saved yourself the trouble of sweeping the whole codebase for references.
>
> Really awesome stuff.
> Unfortunately, by the time I found this, the author of this tool was several years gone, and had left no way of being contacted.
> This tool was clearly a lot of work, and I would have loved to understand what their logic was for building it.

In summary, here are two bits of advice for using constants in your future code:

1. Before you hoist a value to a constant, ask yourself: does this value need a better name?
   Or am I going to reference it in multiple places?
   If the answer is no for both, consider skipping the constant and just using the value directly.

2. When you do decide to use constants in your code, scope them as tightly as possible.
   The more globally you define the constant, the greater risk you run of leaking the constant into unrelated scenarios.
   An necessary constant only reduces readability, but an overscoped constant can lead to real bugs.

## Never Repeating Anything

"[Don't repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)" is an often-quoted tenet of software engineering.
The idea is to design software so fixing a bug or changing requirements changes few locations in the code (ideally one).
The way to achieve this is usually some form of indirection.

Consider this example.
We're trying to parse some flags from command line arguments, using the Windows convention (i.e. we accept both `-flag` and `/flag` for each).
A simple, clean first design might look something like this:

```cpp
output.foo = false;
output.bar = false;
output.baz = false;

for (auto it = args.begin(); it != args.end(); ++it) {
    auto arg = *it;

    if (arg == "-foo" || arg == "/foo") {
        output.foo = true;
    }
    if (arg == "-bar" || arg == "/bar") {
        output.bar = true;
    }
    if (arg == "-baz" || arg == "/baz") {
        output.baz = true;
    }
}
```

Later a requirement comes in to also support "Unix-style" flags (`--flag`).
This is going to require cascading changes: every `if` is going to also need this flag.
Now that we're sensing a pattern of changing requirements, we might choose to add indirection, in accordance with 'not repeating ourselves:'

```cpp
bool isFlag(const str &arg, const str &flag) {
    return arg == "-" + flag || arg == "/" + flag;
}

...

output.foo = false;
output.bar = false;
output.baz = false;

for (auto it = args.begin(); it != args.end(); ++it) {
    if (isFlag(*it, "foo")) {
        output.foo = true;
    }
    if (isFlag(*it, "bar")) {
        output.bar = true;
    }
    if (isFlag(*it, "baz")) {
        output.baz = true;
    }
}
```

After this refactor, adding Unix-style flags just means adding `|| arg == "--" + flag` to `isFlag()` to be done with it.
We added a level of indirection (the `isFlag()` helper) to remove duplicated code (the `if` clauses) which wasn't truly necessary.
Assuming there are future changes in this area, deduplicating benefits us by making fixes simpler and less error-prone.
Not repeating ourselves was a good idea!

It's possible to take DRY too far: sometimes code *should* be duplicated!
Removing duplication in related code is usually a win, but removing duplication in unrelated code just links two unrelated scenarios.
Today the code might be identical for both, but tomorrow you might get a bug or a requirement change which needs a fix to one scenario and not the other.
Because the code was shared, the fix becomes tricky.

Consider this method:

```cpp
void doWork(bool isFoo, bool isBar) { ... }
```

The `isFoo` and `isBar` flags are a code smell.
They indicate that when we created `doWork`, we were trying to link unrelated things together.
Although `Foo` and `Bar` may have originally been identical, over time we realized each needs specific logic that the other doesn't.
The flags are a hack to make this work quickly.

With a little more work, it's usually possible to refactor something like this:

```cpp
// @param isFoo: true for Foo scenario, false for Bar scenario
void doWork(bool isFoo)
{
    // (A)

    if (isFoo) {
        // (B)
    }
    else {
        // (C)
    }

    // (D)

    if (isFoo) {
        // (E)
    }
}
```

to look more like this:

```cpp
void doA() { /* (A) */ }
void doB() { /* (B) */ }
void doC() { /* (C) */ }
void doD() { /* (D) */ }
void doE() { /* (E) */ }

void doFoo() {
    doA();
    doB();
    doD();
    doE();
}

void doBar() {
    doA();
    doC();
    doD();
}
```

This nets you a few benefits:

1. **Changes to `doFoo` and `doBar` are isolated.**
   If you change `doFoo` and only `doFoo`, there's no way that change can possibly impact the `doBar` code path.
   In the first example, and change to `doWork` could possibly impact either scenario, intended or no.

2. **Fewer conditions to work with.**
   Conditional code is harder to think about than unconditional code, especially if there are lots of conditions, or the conditions are complicated.
   Unconditional code is also easier to cover with tests.

3. **Scales with your feature set.**
   If you already have scenarios for `Foo` and `Bar`, it might not be a stretch to think the requirement for `Baz` is just around the corner
   The latter strategy makes it easy to add `Baz`: just make a `doBaz()` and pick the pieces which make sense for `Baz`.

This example might look a little silly, but this can be tempting in practice, especially when sections `(A)` and `(D)` are long, and the others are short.

