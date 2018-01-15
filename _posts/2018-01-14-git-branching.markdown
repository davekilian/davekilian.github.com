---
layout: post
title: Effective Git
author: Dave
draft: true
---

TODO

This was originally a Git uber-post but I decided to break it out.
Nix all of the git internals stuff, except for what we will need, link to the book for the rest.

Then cover topic branches:

* What will go wrong if you do work in master and hit a merge conflict
* Use topic branches instead
* Keep topic branches up to date by routinely merging in the latest master branch
* Use `git pull --rebase origin master` to keep all your commits on the tip of your topic branch
* Use interactive rebase to clean up history and/or drop commits as needed
* Keep a copy of your topic branch remotely, force-push liberally
* Merge your topic branch into master as a fast-forward merge

Then cover feature branches:

* If possible, try to avoid making a feature branch, work in master with the feature "off" instead
* Seriously, these are way overused, you really really don't need one of these
* What will go wrong if you share topic branches with someone else
* Use a feature branch as you would master: create topic branches and merge them in as fast-forward merges

Then cover release branches:

* For stabilizing a release (bugfixes only) while development continues (bugfixes and other work)
* Default mode: fix in master, cherry-pick to release branch(es)
* If master diverged too far, spin two separate fixes by hand
* What will go wrong if you try to fix in the release branch first -- frankenstein bugs

---

Git's flexibility is both its greatest asset and its greatest downfall.
By avoiding pushing you to use it in any particular way, Git makes it easy for you to use it however you wish -- even if that means letting you do things that mess up the repo or generate needless busywork.

Consider this post a style guide for using Git effectively.

## Know Git Data Structures

Git is somewhat unique among version control systems in that it centers around an on-disk data structure which can be manipulated by a sort of command-line "API," rather than providing commands that try to fit a particular workflow.
More than anything else, understanding this data structure is key to using Git effectively and avoiding trouble.

In short, any version control system needs to be able to diff past versions of files and reconstruct past snapshots of files.
As such, there are two high-level approaches to designing a version control system:

1. Store diffs on disk, reconstruct snapshots from diffs on-the-fly
2. Store snapshots on disk, diff snapshots on-the-fly

Git is firmly in the second camp.
Its internal database stores snapshots of files (with deduplication and compression to save space), and whenever a diff is requested, one is generated on-the-fly from corresponding snapshots.

Git's fundamental unit of history is called a *commit.*
A commit is a complete snapshot of all the files in you repository.
So if you had an empty repo, added 5 files, committed, changed one of those files, and committed again, you'd end up with two commits, each containing snapshots of all 5 files.

Commits are chained together to compose a source history.
Each commit points to the parent commit(s) it came from.
Most commits have just one parent: the commit you were working off when you ran `git commit` to create the newer commit.
In some cases, a commit might have more than one parent (e.g. if you combined two commits as part of running `git merge`).
Either way, if you have a commit, you can walk the history of your files backwards in time by recursively following each commit's parent pointer(s).

In this way, the commits of a Git repo form a directed graph of snapshots.
Each node in this graph is a commit (which, in turn, is just a snapshot of all your files); each edge in the graph points from a commit to the commit(s) that directly preceded it.
Starting at any commit in the graph, you can walk the history back in time by following the edges, all the way back to the initial commit.

A Git branch is a pointer to a commit.
Git doesn't need to store any additional information in a branch, because to follow the history of a branch back in time, you can start at the commit the branch points to and walk the commit's parent chain from there.

Git tracks which branch you're currently working on in a global variable called `HEAD`.
That is, `HEAD` points to the current branch, which in turn points to the most recent commit on that branch.
Somewhat confusingly, `HEAD` can mean any of these, so for this guide we'll use some specific terminology to disambiguate:

1. **The `HEAD` variable**: `HEAD` itself
2. **The `HEAD` branch**: whichever branch the `HEAD` variable points to
3. **The `HEAD` commit**: whichever commit the `HEAD` branch points to

Each Git repository has a complete copy of your files, which you can directly edit, compile, debug, and so on as needed.
Collectively, these files form the repository's *working tree.*
For convenience, Git also provides a *staging area* as an intermediate between the working tree and the commit database.
The staging area exists for convenience, and helps you get your changes in order before committing them to Git's internal database.

There's more to how Git stores data on disk, but the above is really all you need to know to work effectively with Git.
If you're interested in a more complete discussion, or in delving deeper into the on-disk format, try reading [Pro Git, Chapter 10](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).

## Know How Commands Map to Data Structures

If you understand how Git stores source history internally, it's straightforward to figure out which commands you'll need most often, and what they do internally.

The working tree, staging area, and `HEAD` commit all start out in sync with one another.

To edit the working tree, you just edit files on disk directly.

To revert changes to the working tree, bringing it back in line with the `HEAD` commit, you can run `git checkout <path>`.
This copies the file (or folder, recursively) out of the `HEAD ` commit.

To propagate changes from the working tree to staging area, thereby staging the changes to be committed, you run `git add <path>`.
If you need to undo this, `git reset <path>` copies that path from the `HEAD` commit to the staging area, thereby reverting your changes.
Running `git reset` does not affect your working tree.

Once you're happy with the changes in the staging area, you run `git commit` to commit them.
This does several things in lockstep:

1. Creates a new commit whose content is the data in the staging area, and whose parent commit is whatever `HEAD` points to
2. Adds that new commit to the database
3. Updates the `HEAD` branch to point ot this new commit (making it the new `HEAD` commit)

To create a new branch, you run `git branch <name>`, which creates a new branch with the given name pointing to the `HEAD` commit.
To delete a branch, you add the `-D` flag to the same commit (`git branch -D <name>`).

To switch branches, you use `git checkout <branch>` (notice we've overloaded `git checkout`).
Checking out a branch brings everything back into sync:

* The `HEAD` variable is modified to point to the given branch
* The staging area is reset to match the new `HEAD` commit
* The working area is reset to match the new `HEAD` commit

---

TODO

* You need a way to change what a branch points to -- `git reset --hard`
* You need a way to compare stuff -- that's all git diff
* You need a way to walk history bakc in time -- that's git log

## Know How to Share Data Between Repos

TODO

* 

## Hacking History

* 

## Branching

## Fixing Things

### Immutability

### Resetting

### Rebasing

### Cherry-Pick

### The Reflog

### The Big Hammer

