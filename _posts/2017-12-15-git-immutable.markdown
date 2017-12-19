---
layout: post
title: Getting Out of Trouble with Git's Immutability
author: Dave
draft: true
---

For how popular a tool Git is, we sure don't seem to understand it very well!

Beginners routinely have trouble getting Git to work smoothly, and even more advanced users stumble from time to time.
Most developers who have used Git extensively for any reasonable period of time have had the experience of trying to do something straightforward, only to find the repo in such a weird state that it almost seemed simplest to just delete the whole thing and start over.

Git is a powerful and flexible version control system, but its Achilles' heel is the unintuitive way it works.
Both of these stem from the fact that Git cleverly (perhaps over-cleverly) manages to solve most version-control problems with a small set of data structures and some low-level tools for editing those data structures on disk.
That lets you use Git in ways its designers never foresaw, but it also means you have to think how Git works, instead of Git working how you think.

To its credit, Git is designed to help you reliably get out of trouble.
Git's on-disk data structures are immutable, so it's always possible to reset back from a bad state to a good one with a handful of commands.
This is no solace to beginners, however, because using this to go back to a known-good state requires intermediate-level knowledge of how Git works internally to begin with.

In this post we'll talk about how Git structures data on disk, and how to use that structure to revert unintended changes and get back to a known-good state.
Once you understand this, you'll never need to blow away your repo again!

## Git in a Nutshell

As previously mentioned, Git works in an unintuitive way.

Programmers generally think of their source control system as a database of "diffs."
That is, you generally think of your source control as a linear history of changes, each of which added or removed some lines of code on top of what was previously checked in.
What makes Git odd is that it doesn't work like this; instead, each commit in Git is actually a snapshot of your entire source tree!

To make that point a little clearer, imagine the act of cloning the repository.
As part of that, we'll need to copy the latest state of the source code to your local disk, so you can edit it.

With an "intuitive" source control manager, the algorithm for doing so would be something like this:

> *Start with an empty directory.
> Then, walk the changes in the history from oldest to newest, each time applying that change to the directory.
> The end result is the current state of the repo.*

With Git, the algorith is instead much simpler:

> *Look up the latest commit, and then copy the contents of the commit to the local directory.*

The Git algorithm works because it stores snapshots of your files, not diffs.

None of this is to say Git doesn't use diffs at all; when you're working with Git day to day, you generate and view diffs all the time.
The important point to keep in mind is that, whenever Git gives you a diff between something and something else, that diff was generated on-the-fly.
For example, if you diff two commits, Git just looks at the contents of each commit (since each commit is a snapshot of your entire working tree) and compares the contents to generate a diff.

To combine commits into a history, each commit has a pointer to its 'parent' commit.
By convention, the parent commit should be the commit on top of which this commit was authored; thus, if I diff a commit vs its parent commit, the result should be the changes introduced by that commit.

## Basic Operations

## The Big Hammer

