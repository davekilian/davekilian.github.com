---
layout: post
title: Trouble with Best Practices
author: Dave
---

Learning and implementing best practices is an essential step to getting to know a new programming language/framework/etc.
In general, best practices get you better results faster, and with less hassle; however, they're no no substitute for thinking before you code :-)

To apply a best practice, it's important to remember what makes that practice best, and decide whether that really applies to the situation at hand.
Otherwise, thoughtlessly following so-called "common sense" rules can lead to trouble.

In this post we'll take a look at one of the more universal best practices in programming: naming literal values with constants.
We'll see how even the lowly constant can cause more harm than good.

## Constant Obfuscation

Constants hide a literal value behind a variable name.
According to best practices, there are two good reasons to use a constant instead of the literal by itself:

1. To assign a more descriptive name to a value that isn't clear enough on its own.
2. To link all uses of a value to a single code location, making the value easier to tweak later.

Consider the following example:

~~~cpp
// (A)
int size = 1048576 * getFileSize();

// (B)
int size = BYTES_PER_MEGABYTE * getFileSize();
~~~

Clearly example (B) above is easier to read.
In example (B), we can tell that getFileSize returns a size in megabytes, and that we intend to convert that to a byte count.
In example (A), it isn't immediately clear where the number 1048576 came from.

One common mistake with this practice is to over-generalize, by saying _every_ value should be tucked behind a descriptively-named constant.
This isn't a great idea!
Using constants where they're not needed obfuscates your codebase.
And, in fairly common cases, abusing constants can also read to real code bugs.

Let's illustrate this an example.
Say we want to write a parser which can read XML documents like this:

~~~xml
<Books>
    <Book Title="A Game of Thrones" Author="George R. R. Martin" />
    <Book Title="American Gods" Author="Neil Gaiman" />
    <!-- ... and so on -->
</Books>
~~~

We can write a simple parser like this:

~~~cpp
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
~~~

Not a bad first draft; but this code has some bare literals mixed in with the logic.
Why don't we improve this code by hoisting those out as nicely-named constants?

~~~cpp
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
~~~

We can even go all-out and make a 'constants' file which is global to the entire program:

**constants.h**

~~~cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define TITLE_ATTRIBUTE_NAME "Title"
#define AUTHOR_ATTRIBUTE_NAME "Author"
~~~

**parser.cpp**

~~~cpp
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
~~~

Before we added constants, you could look at a copy of books.xml and parser.cpp, and be sure parser.cpp was implemented correctly.
You could even reverse-engineer the structure of books.xml just by reading parser.cpp.
But now with constants, you'd have to jump back and forth between parser.cpp and constants.h to fully get what's happening.

Using constants in this way doesn't play to either of the strengths of constants:

1. The strings were already descriptive; aliasing them with a constant name didn't tell us anything we didn't already know.

2. Each string literal was only referenced once to begin with.
   Changing node names is an equally time-consuming task whether or not we employed constants.

In the last example, where we created a separate code file for constants, there's a more insidious problem:
because the constants are declared globally to the entire program, there's a hazard someone might later pick up those constants for an unrelated scenario.
Once unrelated scenarios are linked by the use of global constants, changes in either scenario can unintentionally break the other.

For example, say you wrote your book XML parser like in the final example above.
Then later, a design requirement comes in for a movie XML parser.
The person who writes the movie parser reuses `TITLE_ATTRIBUTE_NAME`, because movies also have titles.
Now both the book parser and the movie parser depend on `TITLE_ATTRIBUTE_NAME` having the value `"Title"`.

Later on, a design change comes in for the book parser, to rename the `Title` attribute to `Name`.
The developer making this change might decide change the value of `TITLE_ATTRIBUTE_NAME` to `"Name"`.
Unfortunately, the title attribute for books and movies have been coupled together by the constant, so this inadvertently breaks the movie parser.

Maybe the developer falls into this trap, maybe they don't.
But either way, they still have to decouple the two scenarios somehow.
With the current design, the changes look awkward.
They'll probably either look something like this:

~~~cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define BOOK_TITLE_ATTRIBUTE_NAME "Name"
#define MOVIE_TITLE_ATTRIBUTE_NAME "Title"
#define AUTHOR_ATTRIBUTE_NAME "Author"
~~~

or this:

~~~cpp
#define BOOKS_NODE_NAME "Books"
#define BOOK_NODE_NAME  "Book"
#define BOOK_XPATH "/" BOOKS_NODE_NAME "/" BOOK_NODE_NAME
#define TITLE_ATTRIBUTE_NAME "Title"
#define NAME_ATTRIBUTE_NAME "Name"
#define AUTHOR_ATTRIBUTE_NAME "Author"
~~~

But we could have avoided all this trouble if the constants were never shared or made global in the first place.
Really the book XML and movie XML parsers are separate, and any constants related to parsing should also be maintained separately.

> ### Aside: Really going overboard with constants
>
> Once, in all my code-reading adventures, I stumbled upon a tool for generating constants.
> A developer could run this tool on their codebase to convert literals into automatically-generated constants.
>
> For example, if your code had a string like this:
>
>     "http://www.example.com"
>
> The tool would replace the literal with a constant like this:
>
>     __http_colon_fwdslash_fwdslash_www_period_example_period_com__
>
> This is super neat, because it violates both of our reasons for using constants:
>
> 1. **Use constants to assign a more descriptive name to an otherwise bizarre-looking value.**
>    The automatically-generated name is exactly as descriptive as the value itself, because the name is the value itself!<br><br>
>    
> 2. **Use constants to link all uses of a value to a single code location, to be tweaked later.**
>    Since the name of the constant is a derivation of its value, if you change the value, you'd also need to rename the constant and fix all references.
>    So you haven't saved yourself the trouble of sweeping the whole codebase for references.
>
> Unfortunately, by the time I found this, the author of this tool was several years gone, and had left no way of being contacted.
> This tool was clearly a lot of work, and I would have loved to understand their thinking behind writing it.

So now that we know there are pitfalls to using constants, how can we avoid them?
Here's my advice:

1. Before you hoist a value to a constant, ask yourself: does this value need a better name?
   Or am I going to reference it in multiple places?
   If the answer is no for both, consider skipping the constant and just using the value directly.

2. When you do decide to use constants in your code, scope them as tightly as possible.
   The more globally you define the constant, the greater risk you run of leaking the constant into unrelated scenarios.
   An unnecessary constant only reduces readability, but an overscoped constant can lead to new bugs.

## Takeaway

There's an old saying that goes, "every problem in software design can be solved with another layer of indirection."
Like constants, many best practices add some form of indirection to your codebase.
But adding indirection to your code doesn't come for free:

* **Indirected code is harder to read.**
  The reader has to jump around between multiple locations to get the full picture.
  Jumping around breaks the reader's concentration, making it harder to see how everything interacts.

* **Indirected code is harder to refactor.**
  Indirection adds an internal code contract between two or more parts of the implementation.
  When the time comes to refactor, reworking an internal contract is harder than simply adding one.

If applying indirection provides benefits which offset these costs, then it's the right thing to do.
In fact, if it's beneficial enough, we'd even call it elegant.
But you have to consider costs and benefits on a case-by-case basis, "best practice" or not.
Otherwise you can accidentally make your codebase messier than it needs to be.

