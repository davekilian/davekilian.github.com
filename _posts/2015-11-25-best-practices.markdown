---
layout: post
title: Commonly Abused "Best Practices"
author: Dave
draft: true
---

Over time, every programmer accrues a repertoire of best practices and pitfalls to avoid when designing software.
As you build yours, it's important to understand not only the rule itself, but also the reasoning behind it.
Without understanding the reasoning, it's easy to apply perfectly good rules of thumb to the wrong situations, with disasterous effects.

Commonly, the problem lies in indirection.
Most software best practices involve some form of indirection (as the old joke goes, "every problem in computer science can be solved with another layer of indirection.").
Sadly, indirection is not free.
It makes it harder to read code (since the code is spread across two or more locations instead of one).
It also makes it harder to refactor (since now there's another internal code contract to maintain).

Sometimes indirection is worthwhile despite this -- but not always!
In this post we'll go over a few common best practices, and note how they can backfire when applied to the wrong situation.

### Constants. Constants everywhere

There are two good reasons to hide literals behind well-named constants:

1. Constants let you assign a more descriptive name to a 'magic' value which is otherwise inscrutable.

2. Multiple functions can use the same constant, allowing you to change the constant once and reflect the change everywhere

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

One common mistake is to take this to mean _every_ literal should be hidden behind a constant.
Using constants where they're not needed obfuscates code, and over time can lead to real bugs.
Consider this example:

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

The implementation above is fairly clean, minimal, and easy to understand in a single sitting.
Have you ever ended up with something like this instead?

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
In the last example, you can't -- you'd have to jump back and forth between parser.cpp and constants.h to fully get what's happening.

Using constants in this way doesn't play to either of the strengths of constants:

1. The string literals themseleves were already descriptive enough; adding a constant name didn't tell us anything we didn't already know.

2. Each string literal was only referenced once; if you want to change the name of a node, the cost is the same whether or not you used a constant.

There's a more insidious flaw in the final example.
Because the constants are declared globally to the entire program, you hazard someone later picking up those constants for an unrelated scenario.
Changes to the constant then affects multiple scenarios, perhaps unintentionally.

To continue the example, say you wrote your book XML parser like in the final example above.
Then later, a design requirement comes in for a movie XML parser.
The person who writes the movie parser reuses `TITLE_ATTRIBUTE_NAME`, because movies also have titles.
Now both the book parser and the movie parser depend on `TITLE_ATTRIBUTE_NAME` having the value `"Title"`.

Later on, a design change comes in for the book parser, to rename the `Title` attribute to `Name`.
Unfortunately, the title attribute for books and movies have been coupled together by the constant.
The developer making this change will likely rename the title constnat, inadvertently breaking the unrelated movie parser.
But whether or not they get bitten by this, either way they'll still have to do something awkward to decouple those constants.
Something like this:

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

1. Before you add a constant to your code, ask yourself whether you're making the literal value more descriptive, or whether you're going to use the constants in multiple contexts which need to be kept in sync, or if you have other extenuating circumstances.
If it's none of the above, consider using the literal value directly instead.

2. If you do decide to use a constant, scope it as tightly as possible.
   The higher you hoist a constant, the greater risk you run of accidentally coupling unrelated code through the constant.

---

I rebranded this and flattended it a bit.
Now it's an examination of commonly abused best practices.

* Hiding every literal in a constant
* The 'everything is an interface' anti-pattern
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
