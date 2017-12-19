---
layout: post
title: Getting Out of Trouble with Git's Immutability
author: Dave
draft: true
---

For such a popular tool, we sure don't seem to understand Git very well!

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

When you work with version control systems, you're constantly dealing with diffs:

* You diff your code versus the checked-in state to see what you've changed
* When you're done with a change, you check it in to make it permanent
* To understand the history of a file and how it was authored, you check the list of changes that have been made to that file over its lifetime

Given the above, it's natural to assume version control systems work by internally storing a database of diffs, which can be replayed to reconstruct the current state of any file in the repository on-the-fly.
Indeed, there are many version control systems that do work this way.

However, Git, in its typical fashion, does almost the exact opposite of that.
Instead of storing diffs that can be replayed to construct a snapshot, Git stores snapshots, which can be diffed on-the-fly as needed.

The basic unit of state in Git is called a commit.
A commit sounds similar in concept to a "change," "changelist" or "changeset" from a traditional version control system, but don't be fooled:
a Git commit is not a set of changes, but rather a complete snapshot of your entire repo at the time the commit was made.

To organize commits into a logical "history," each commit has a pointer to its "parent" commit.
A commit's parent commit stores a complete snapshot of the repo from before the new commit was created.
As such, if you have a commit, you can walk backwards in time by following each commit's parent pointer recursively.
Eventually you'd reach the repo's initial commit, which has no parent commit.

It's perfectly valid and pretty normal for more than one commit to point to the same parent commit.
It's also valid for a commit to have more than one parent commit!
The former typically happens when two changes are authored in parallel; the latter typically happens when those two commits are merged together.

Even though you can walk commits backward in time, you cannot walk commits forward in time.
That is, commits have pointers to their parents, but they do not have pointers to their children.
To view the repo's history, your only option is to start with the latest commit and walk backwards in time.

Git provides branches for tracking these latest commits with a well-known name.
Internally, each branch is nothing more than a pointer to a commit.
When you run `git log` for a branch, Git starts at the commit the branch points to and walks backward in time until it reaches the repo's initial commit.

If you grok all of the above, then congratulations!
You understand almost everything you need to know in order to work with Git effectively.
Without further ado, we'll move on the tools you'll find useful for getting out of trouble.

## Resetting

## Rebasing

## Cherry-Pick

## The Reflog

## The Big Hammer

