---
layout: post
title: A Layman's Guide to JavaScript Objects
author: Dave
draft: true
---

JavaScript may not seem like an object-oriented programming language at first
glance. 
Although the language does provide object-oriented features, they may not be
straightforward to learn if you're used to a more classical object-oriented
language like C++ or Java.
This guide aims to be a clear and succinct introduction to objects in
JavaScript.

## Everything in JavaScript is an Object

That is, any value you can assign a variable to must be an object.
JavaScript has a very simple built-in type system, the root of which is a type
unsurprisingly named `Object`.

`Object`s are dictionaries that map `String` keys to arbitrary `Object` 
values.
You can specify `Object` literals using the notation below:

```js
    function getValue(num) { ... }
    var obj = { 'first': getValue(1), 'second': getValue(2) };
```

The variable `obj` above holds a reference to an object.
The object it references has two properties: 'first' and 'second'.
We can retrieve them two different ways:

```js
    // The convenient way:
    var first = obj.first;

    // The long way (useful when the property name contains spaces or
    // punctuation):
    first = obj['first'];
```

A quick note about semantics: the variable `obj` holds a reference to an
_instance_ of `Object`, not a _subclass_.
We'll talk about inheritance and subclassing later in this guide.

JavaScript provides a few built-in subclasses of `Object`, each of which
corresponds to a certain type of literal in the language:

* `Boolean` is a subtype of `Object` that is used by the `true` and `false`
  literals, and can be used by logical operators.

* `Number` is a subtype of `Object` that is used by numerical (i.e.
  floating-point) literals, and can be used by arithmetic operators.

* `String` is a subtype of `Object` that is used by textual literals.

Another built-in subtype of `Object` is `Function`, which is used for function
definition literals:

```js
    var func = function(a, b) { return a + b; }
```

(Everything to the right-hand side of the `=` sign is a function literal)

---

### TODO fix this up to flow correctly

* `Function` can accept an argument list and function body as a value,
  and can be called.

The last item above is key: JavaScript functions are `Object`s, just like any
other data value.
This makes it easy to assign functions to variables, which is one of the things
that make callbacks so ubiquitous in the JavaScript language.

### TODO since functions are variables we can add methods

### TODO we can use `this` inside methods to access the outer function

### TODO example defining an object

## Object Instantiation

### TODO call new on an example ctor function

### TODO what new does

### TODO aside: the meaning of `this`

## Prototypes

### TODO motivate problem: what about class methods?

### TODO introduce the prototype chain

### TODO introduce the 

## Inheritance

### TODO copying prototypes

### TODO calling the parent constructor using `Function.call()`

