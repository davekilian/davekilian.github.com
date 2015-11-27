---
layout: post
title: Design Hurtles for Intermediate Developers
author: Dave
draft: true
---

Mistakes are a healthy part of the learning process.
Everyone makes them.
Learning from your mistakes and the mistakes of others is the fastest way to become good at something.
I've seen lots of materials online about the mistakes beginning programmers make, and how to avoid them.
I've seen less about what kind of mistakes developers start to make once they move past the beginner level.

In recent years I've been running into a lot of this kind of code, so in this post I'd like to share some common patterns I've seen, why I think they're harmful, and how I think you could avoid or improve on each.

## Too Much Indirection

(also known as: [You aren't gonna need it](http://martinfowler.com/bliki/Yagni.html))

Of all the patterns in this article, this one is the most pervasive.
It crops up at every level of design.

Here's the problem: adding indirection to some code always makes it a little bit harder to read or refactor that code.
If you get a tangible benefit from indirection, then this penalty may be worthwhile.
Surprisingly often, people take this penalty without an easy to state tangible gain.

I think most programmers who make this class of mistake know about this idea and are wary of it.
In many instances, I see this mistake appear when people blindly follow 'best practice' rules without considering the underlying reasoning.

Let's look at some examples of best practice rules which can actually be harmful if misapplied.

### Constants everywhere

There are two good reasons to hide literals behind well-named constants:

1. Constants let you assign a more descriptive name to an otherwise-inscrutable 'magic' value
2. Multiple functions can use the same constant, allowing you to change the constant once and reflect the change everywhere

Consider the following example:

```cpp
int func1(int value) { return 1048576 * value; }

int func2(int value) { return BYTES_PER_MEGABYTE * value; }
```

Which of the following functions is easier to read?
The obvious answer is `func2`.
Only `func2` communicates to the reader that the input value is in bytes, and the output value is converted to megabytes.

One common mistake is to over-apply this principle to mean _every_ literal should be hidden by a constant.
Adding a constant in between the code and the literal doesn't always add anything beneficial.
But hiding a literal behind a constant does add indirection, which does make code harder to read.

Consider this example.
Say we want to write a parser which can read XML documents like this:

```xml
<Books>
    <Book Title="A Game of Thrones" Author="George R.R. Martin" />
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

Have you ever seen something like this instead?

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

Or even this?

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

In the original example, you could look at a copy of books.xml and parser.cpp, and be sure parser.cpp was implemented correctly.
You could even reverse-engineer the structure of books.xml just by reading parser.cpp.
In the last example, you can't -- you'd have to jump back and forth between parser.cpp and constants.h.

Using constants in this way doesn't play to either of the strengths of constants:

1. The string literals themseleves were already perfectly descriptive; adding a constant name didn't tell us anything we didn't already know
2. Each string literal was only referenced once; if you want to change the name of a node, it costs the same thing in either example.

There's another, more insidious flaw in the last example.
Because the constants are declared globally to the entire program, use of the constantmay accidentally leak into an unrelated context.

Say you write the book XML parser like this.
Then later, a design requirement comes in for a movie XML parser.
The person who writes the movie parser reuses `TITLE_ATTRIBUTE_NAME`, because movies also have titles.
Now both the book parser and the movie parser depend on `TITLE_ATTRIBUTE_NAME` having the value `"Title"`.

Later on, a design change comes in for the book parser, to rename the `Title` attribute to `Name`.
Unfortunately, the title attribute for books and movies have been coupled together by the constant.
In all likelihood, the programmer making the schema change will rename the constant, inadvertently breaking the movie parser.
When they find this bug, they'll have to do something awkward like this to decouple those constants:

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

Two takeaways for this in closing:

1. If you have the impulse to use a constant, ask yourself what tangible benefit you get from using a constant.
   If you can't think of one, don't use a constant.

2. If you decide you do need a constant, scope the constant down as far as possible.
   The higher you hoist a constant, the greater the risk you run for coupling unrelated code through a constant.

---

Each section should have a description of the mistake, an example, and a
discussion of how to fix the example

* The big one: speculative abstraction
    * Creating layers of indirections because you 'think' you might need them
    * You can't prove you need them, and you probably won't
    * aka: You Aren't Gonna Need It
    * A common reason for adding abstraction is to make software 'extensible'
    * But the easiest software to extend is plain code with no abstraction
    * Specific examples
        * Why should every literal be hidden in a constant?
          If a literal is used once, just let it be a literal
          If a literal is used all over, fix your code so it's only needed once
        * The 'everything is an interface' anti-pattern
        * Abusing depedency injection/inversion of control
* The minimum code spanning tree
    * Over-adhering to 'don't repeat yourself'
    * You have less code, but the code is extremely complex
    * The code is difficult to reason about
* Non-reusable abstraction
    * "This chunk of code looks complicated. Let's pull it out and make a
      method"
* Clever syntax
    * Most language syntax exists for a single killer app
    * Trying to use it for something else feels clever, but usually just harms
      readability in the end.

Takeaway: these all share an underlying trait: they're way too complex

* Software should be stable first, and readable second. All else is ancillary
* Simple software is readable
* Simple software is easy to reason about
    * Easier to catch bugs early
    * Easier to test
    * Easier to debug
* Simple software is easier to reuse
* Simlpe software is easier to refactor
* Simlpe software is easier to extend

Bonus: Take a popular language and pull out its big-name features. Then show
the killer app for which the feature is useful, and show one or more examples
of abuses.

