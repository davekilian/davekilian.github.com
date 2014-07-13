---
layout: post
title: The Web is a Series of Hacks
author: Dave
draft: true
---

## Brain Dump

Let's take a look back at where we came from and where we are

* HTML was TeX with hyperlinks. Do a compare/contrast table. Point out HTML's key differentiators
* CSS was the answer for people who cared about data presentation, even though HTML (like TeX) was designed to fundamentally disregard data presentation
* JavaScript was the answer for people who wanted to manipulate the document structure dynamically, without page loads.
* When did HTTP add verbs like PUT? I wonder if even that was a concession to people who wanted to misuse HTML
* One good reason JavaScript wasn't type-safe or object-oriented: it was supposed to be a toy scripting language that kind of looked like Java
* What makes JavaScript even more bizarre is it's the first concession for people who want to turn the browser into an interpreter, rather than a tex compiler
* HTML+CSS+JS are slowly becoming more application-friendly, but they're fundamentally what they always were
* Server-side languages are -- hang on, there are no server-side languages
* Server-side libraries are built on one fundamental principle: string concatenation. Wanna generate some HTML? Insert some CSS? Create some JavaScript literals? Query a database? 
* Most of the security problems in webapps stem from all this concatenation -- it puts injection vulnerabilities _everywhere_. String validation and escaping is monkey-patching. Why don't we separate data from logic? 
* The other security problems stem from the fact that the HTML stack was never meant to be a secure system that can run government systems and world banks -- it was a document library with convenient inter-document references!
