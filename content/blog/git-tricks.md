---
title: "Things I wish I learned earlier about git"
description: "This is a collection of little workflow secrets that will help you maintain a cleaner git commit history and write better PRs."
summary: |
  Most programmers already know the good old "git flow" with a handful of easy commands: branch, commit, merge. 
  However, these are usually not enough to guarantee you produce clean commit histories and good and easy 
  to review pull requests. They're also not enough to recover your work when you inevitably screw up.
  This blog post will address all these things by explaining some techniques I wish I had learned earlier.
lastmod: 2024-10-13
layout: single
---

# Preface

First and foremost: this post assumes you already have some familiarity with git. It supposes you're already reasonably comfortable with:
- Basic terminology such as "commit", "branch", "merge", "merge conflict", "pull request" etc
- The basic "git flow" workflow:
  - Creating a new branch with `git branch`, `git switch` and `git checkout`
  - Manipulating your staging area with `git add` and `git reset`
  - Creating commits with `git commit`
  - Syncing your changes with the remote server remote using `git push`, `git pull` and `git fetch`
  - Merging your changes back to the upstream branch with `git merge`
  - Inspecting your commit history with `git log`
- Asking for reviews, reviewing pull requests and working in a team with other developers

If you're not yet familiar with these concepts, that's no shame! Go learn git basics and come back to this once you're ready :)


With that out of the way, in this blog post we'll go over the following topics:
- why you should care about having a clean commit history
- why and how to use `git rebase` instead of `git merge` for syncing `main` onto your branch
- using `git rebase --interactive` and `git commit --fixup` to fix past mistakes and address pull request comments
- using `git reflog` to make sure you'll _never ever_ lose your work by accident
- breaking large changesets into stacked pull requests
- how to combine it all as one coherent workflow

Feel free to skip to the sections that interest you most!

# Why you should care about having a clean commit history

Before getting onto the meat and potatoes of the techniques this blog post covers, I want to motivate it by explaining why I truly believe you should aim for writing clean commit histories. Most of the techniques I'll cover revolve around the basic assumption that you care about that. If you don't really care that much, I'll try to change your mind with the following list of reasons to start caring.

- **An organized history helps you consult changes from the past.** The commit history serves as a sort of documentation about how the project evolved and why certain changes were made. If you're ever questioning why something is the way it is, you can always `git blame` and read the commit message where that confusing change was introduced. If the commit history is good, that commit message will contain which other alternatives were considered but discarded, why the confusing alternative was chosen and why its pros outweighed its cons.
- **A messy history makes it hard to revert commits if something goes wrong in production.** You mixed "add feature A" and "refactor B" in the same change set, even though they're unrelated. Unfortunately, "refactor B" accidentally introduces "bug C", and now there is no way to revert this without also removing "feature A". You're stuck having to either rush a forward-fix to bug C (which could be risky) or completely reverting feature A (which is far from ideal). If your history were clean and you properly separated these, you could just revert "refactor B" and reintroduce it again once you figured out why the bug happened and how to fix it, on your own pace, without rushing.
- **An organized history makes it easier for peers to reason about your changes.** You asked for review of a PR which changed 20 files and a few thousand lines all over the codebase, but your commit history is messy. Your collaborators are confused by your changes, and either cluelessly reply with `LGTM!`, giving you no feedback at all, or shower you with a myriad of basic questions. Depending on how bad the situation is, they might even want to join a call with you so you can explain to them what your commits didn't. Instead, you could have avoided this by making small, easily digestible changes that slowly ramp up their knowledge about your changes, and you could just ask them to review your changes commit by commit.

I hope you're convinced by now :smile:

With that out of the way, let's dive in!


# Using `git rebase` to sync the `main` branch with your working branch

You've been working on your branch for a while, and a collaborator just added new changes to `main` which you'd like to bring into your working branch. In that case, most people would do

```sh
git fetch origin/main
git switch my-working-branch
git merge main
```

While it does the job and is not technically incorrect, this approach has one major drawback: you just introduced a "merge commit" into your branch, which makes your commit history a bit uglier. Your commit history now looks like

```text
commit a: Refactor `MyClass` to improve performance
commit b: Add `method_a` to `MyClass`
commit c: Call `MyClass.method_a` from `MyOtherClass`
commit d: Merge branch 'main' onto my-working-branch    <-- this kinda sucks :/
```

To fix this problem, we're gonna use a new command: `git rebase`. In summary, this command changes the "base" of a particular commit or branch to point to another commit (hence, re-base). In this same example, we could use `git rebase main` instead of `git merge main` to turn this

```text
x --- y --- z         (main)
       \
        a --- b --- c (my-working-branch)
```

into this

```text
x --- y --- z                  (main)
             \
              a' --- b' --- c' (my-working-branch)
```

Now, your branch has the latest changes from `main` (commit `z`), but your commit history is clean:

```text
commit a': Refactor `MyClass` to improve performance
commit b': Add `method_a` to `MyClass`
commit c': Call `MyClass.method_a` from `MyOtherClass`
```

If you've been looking closely, you can see that the commit SHAs have changed: commit `a` was replaced by `a'`, `b` was replaced by `b'` and `c` was replaced by `c'`. This is totally fine, and it's expected since git commits are immutable by definition. 

Under the hood `git rebase`:
1. checks out your new base `main`, a.k.a commit `z`
2. applies the changeset of `a` onto `z`, resulting in `a'`
3. applies the changeset of `b` onto `a'`, resulting in `b'`
4. applies the changeset of `c` onto `b'`, resulting in `c'`
5. changes `my-working-branch` to point to `c'` instead of `c`

Logically, your branch remains the same, but now you've got the changes from `z` without a merge commit to mess your beautiful history. Hooray! :smile:


## How do I sync back to my remote server?

As we've just seen, even though your changes _look_ the same, they're actually represented by different commits after the rebase. As such, if you run `git push origin my-working-branch`, your remote git provider will refuse your changes. In this case, it's OK to `git push --force`.

Quick note: if you've already asked for review on your changes, avoid force pushes as they can make it confusing for reviewers who are in the middle of a review. Also, from my experience, GitHub can sometimes get confused by force pushes and forget about PR comments, which makes force pushes not ideal in this case. If that's the case, prefer [fixup commits](#correcting-and-improving-past-commits) and only rebase at the last stage of review, right before merging.


## What if there are conflicts?

In that case, git will pause the rebase and display a warning that tells you there was a conflict somewhere. Just go and fix your conflicts like you'd normally do during a `git merge` but, instead of `git add .` and `git commit`, now you'll run `git add .` and `git rebase --continue`. 

Differently from "merge" conflicts which are grouped in one big conflict set (during a merge you're trying to bring all new changes from `main` onto a single "merge commit" at the head of your branch), "rebase" conflicts might happen in stages. This is because `git rebase` applies one patch on top of each other, one at a time, so there could be conflicts during any of these applications. Don't let this scare you, though! Just solve them one by one, doing `git add .` and `git rebase --continue` until the whole rebase is complete.

If you've messed up the rebase, you can always `git rebase --abort` to roll back to the state before you started the process!


## When should you merge instead of rebase?

Do you want to seamlessly bring in new changes without making noise in the history? Use `git rebase`. Do you want to let others know of this sync between branches by creating a merge commit? Use `git merge`.

I try to follow this general rule of thumb:
- Use `git rebase` to sync downstream. For example: 
  - to sync `main` onto your working branch 
  - to sync a parent working branch onto a child working branch (we'll see why you'd want to do this later in the [stacking pull requests](#stacking-pull-requests) section)
- Use `git merge` to sync upstream. For example:
  - to sync a working branch back onto `main` when you're done working on that feature


# Using `git rebase -i` to modify past commits

You're working on your branch, and you've already added a few commits `a`, `b` and `c`. Then, something happens and you realize there's a problem with one of your older commits:
- you forgot to run your code formatter, or
- you introduced a typo, or
- you forgot to run tests and they're all breaking due to a stupid oversight, or
- you forgot to add an if statement to an important piece of logic, or
- ...

If the problem is in your latest commit (`c`), you can just fix it with `git commit --amend --no-edit` and call it a day. However, if your typo was introduced before that (let's say in commit `a`), a `git commit --amend` won't cut it.
  
A naive (and very time-consuming) solution to this problem is to `git reset HEAD~2`, re-add all your files and re-commit everything again. Not ideal.

You get lazy, create a `Fix typo` (`Fix tests`, `Run formatter`, ...) commit and move on. Your history now looks like this:

```text
commit a: Refactor `MyClass` to improve performance
commit b: Add `method_a` to `MyClass`
commit c: Call `MyClass.method_a` from `MyOtherClass`
commit d: Fix typo                                      <-- do I need to tell you this kinda sucks?
```

There is a better way, though: **interactive rebases**. If run `git rebase --interactive aaaaaa~1`, where `aaaaaa` is the SHA of commit `a` (you can get it from `git log`), you'll be prompted with a _rebase file_ that's opened in your editor. It will should look something like

```text
pick aaaaaa Refactor `MyClass` to improve performance
pick bbbbbb Add `method_a` to `MyClass`
pick cccccc Call `MyClass.method_a` from `MyOtherClass`
pick dddddd Fix typo
```

This rebase file is a list of instructions that git will run, top to bottom, when performing your rebase. The file starts at commit `a` and ends at commit `d` because you specified you want to run your rebase from `a` all the way to `HEAD`.

If you just save the file as-is, essentially nothing will change, and git will `pick` all the commits to apply one after another, one by one, similarly to what we did when rebasing `main` onto our branch.

The real power lies in editing the rebase file, though:

```text
pick aaaaaa Refactor `MyClass` to improve performance
fixup dddddd Fix typo
pick bbbbbb Add `method_a` to `MyClass`
pick cccccc Call `MyClass.method_a` from `MyOtherClass`
```

We moved the `Fix typo` commit to the line after commit `a`, which introduced the typo. We also changed its instruction from `pick` to `fixup`. Now, when we save the file and quit the editor,, git will:
1. Apply commit `a`'s changes (`pick`)
2. Patch commit `a` with commit `d`'s changes, keeping only `a`'s message (`fixup`)
3. Apply commit `b`'s changes (`pick`)
3. Apply commit `c`'s changes (`pick`)

Now, our commit history looks much nicer!
```text
commit a': Refactor `MyClass` to improve performance     <-- the typo is fixed here
commit b': Add `method_a` to `MyClass`
commit c': Call `MyClass.method_a` from `MyOtherClass`
```

It's worth noting that this will also rewrite your commits (like all other rebases!), so you'll end up with `a'`, `b'`, and `c'`.


## Speeding it up with `git commit --fixup`

We've had to type a lot to make all this happen: we had to create the `Fix typo` commit, then do `git rebase --interactive`, then move lines around and write things to the rebase file.

You can make this quicker by commiting directly to a fixup commit, using `git commit --fixup <parent-of-the-broken-commit>`. This will generate a commit that starts with `fixup!`. Then, you can run `git rebase -i --autosquash parent-of-the-broken-commit>~1`. This will generate a rebase file with all the `fixup` commits in the correct places. Just save that file and you're all set!

To avoid having to use `--autosquash` everytime, you can configure git to use it by default with `git config rebase.autoSquash true`.


## Using fixup commits to address pull request comments

`fixup!` commits are very convenient for asking for reviews. If one of your reviewers is not happy with a change in commit `b`, you can incorporate their feedback, create a `fixup!` commit on `b` and `git push`. Then, you reply to their comment, tell them that your new `fixup!` commit fixes the issue, and paste a link to the commit. Now, they have a reference to a commit that specifically addresses that one comment, and they can easily verify it by following the link.

At the end of the review, when you've addressed all comments and your reviewers are OK with your pull request, you run a single `git rebase -i --autosquash` to put all your `fixup!` commits into where they belong. After all of this, you've got a clean history which has incorporated all your review suggestions!


## Other kinds of changes to old commits

So far, we've only explored `pick` and `fixup`. However, rebase offers a lot of different options to change old commits:
- `reword`: rewrite the message of an old commit
- `drop`: just drop an old commit and pretend it never happened :face_with_peeking_eye:
- `break`: stop there and interrupt the rebase. This allows you to make changes before git applies the next patch. You can continue by running `git rebase --continue`
- `squash`: similar to fixup but merges commit messages
- and more!

Check out the coments on your rebase file or git's official documentation for more rebase actions.


## Using alternative editors for the rebase file

By default, git will edit rebase files with your default `EDITOR` (set via an environment variable). I personally use neovim, but other people prefer other editors, such as VSCode. To make git use VSCode as its rebase file editor, run the following command

```sh
git config --global core.editor "code --wait"
```

Note that this will also make git use VSCode for editing commit messages, as they are also just regular files (try to `git commit` and then open `.git/COMMIT_MSG` to see for yourself).


# Using `git reflog` to make sure you'll _never ever_ lose your work

We've now seen how use `git rebase` to rewrite our past commits to change the base or our branch or address mistakes. What if, for some reason, we mess up a rebase and drop an important commit, or accidentally delete something we shouldn't have deleted?

Well, even though `git rebase` changes your branch to point to the new rewritten commits, it will _never_ delete old commits unless you explicitly tell it to. In other words, even if you rewrite `a` as `a'` and `b` as `b'`, the original `a` and `b` won't get discarded. It can be hard to find them, though, since after the rebase there won't be any branches pointing to them. This is when you can use `git reflog`.
 
`git reflog` (reference log) is exactly what it says it is: a log of references. If you need to reference an old commit before a rebase, run `git reflog`, find its SHA, and `git checkout <sha>`. Now you're checked out at your old commit and you can recover your work!

You can even configure how long git will wait before expiring very old reflog entries by setting `gc.reflogExpire` using `git config` (it defaults to 90 days). You can also prune very old reflog entries manually by running `git reflog expire` or `git reflog delete`.


# Stacking pull requests

So far, we've talked about splitting your changes into good commits that are easy to review and neatly document the history of the project. We've talked about using pull requests to ask for reviews from your peers, but we haven't discussed a key characteristic of pull requests that can be problematic if we don't think carefully about it: pull requests are "all or nothing", i.e they can only be merged all at once or not merged at all.

This isn't really a problem when we're implementing relatively simple changes that don't really need to be split into multiple smaller merges. However, imagine you're working on a big feature that possibly changes hundreds of files and contains a lot of different moving parts. It's probably a bad and risky idea to merge all of it at once. It's also not good etiquette to ask your reviewers to give one yes or no approval to one huge chunk of changes like this: they can't possibly reason about hundreds of files all at once, even if they do it commit by commit.

That's when **stacked pull requests** come in. The idea is simple: you open a series of pull requests which branch off of each other, and work simultaneously on all of them, making sure to introduce each change on the pull requests where it belongs. 

For example, let's say you're working on a frontend project feature which involves three major steps: refactoring some code, adding a new component, and using them to create a new page. In this case, one way to split your changes would be to create three major PRs that get reviewed and merged separately:

```text
pull request 1: refactor components X, Y and Z
  commit a1: refactor base component
  commit b1: refactor component X
  commit c1: refactor component Y
  commit d1: refactor component Z
  commit e1: add new tests to refactored components 

pull request 2: introduce new component A
  commit a2: create component A based on components X, Y and Z
  commit b2: add tests for component A

pull request 3: introduce new page
  commit a3: create skeleton of page using component X
  commit b3: complete page implementation
```

In terms of branch structure, your PRs should look like a "stack of branches"
```text
x --- y --- z                                             (main)
       \
        a1 --- b1 --- c1 --- d1 --- e1                    (new-page/1-refactor)
                                     \
                                      a2 --- b2           (new-page/2-component-a)
                                              \ 
                                               a3 --- b3  (new-page/3-create-page)
```


## Manually managing a stack of pull requests with `git rebase --onto`

Now that you know what are PR stacks and when to use them, the next sections will dive into how you'd manually manage a stack. As you'll see, this process can be complicated and tedious, and [there are tools to make it much easier](#using-external-tools-to-manage-stacked-pull-requests), which I'd recommend you use in real life. Nonetheless, I think it's valuable to go over how this works in "pure git" so you can understand what these tools do under the hood.

The first thing when dealing with a PR stack is creating the stack itself. This is as simple as `git branch`ing from parent to child. So, in our example, the stack would be created by doing:
```text
git switch main

git branch new-page/1-refactor
git switch new-page/1-refactor
# do some work and add commits

git branch new-page/2-component-a
git switch new-page/2-component-a
# do some work and add commits

git branch new-page/3-create-page
git switch new-page/3-create-page
```

Then, once you're done, you can open the pull requests one by one in your Git host interface. Make sure to set the correct base branches, so in this case:
- The PR for `new-page/1-refactor` requests to merge into `main`
- The PR for `new-page/2-component-a` requests to merge into `new-page/1-refactor`
- The PR for `new-page/3-create-page` requests to merge into `new-page/2-component-a`

Now comes the fun part: one of your collaborators requested a change in the PR for `new-page/1-refactor`. You `git rebase` that branch, but you notice that `new-page/2-component-a` now contains the commits that were on the old `new-page/1-refactor` before the rebase. This is expected behavior, since `git rebase` creates a new set of commits based on the same changesets, so `new-page/1-refactor` is now a technically a different branch, which is not what `new-page/2-component-a` is based on. In other words, your branches now look like this:
```text
x --- y --- z                                             (main)
      |\
      | a1' -- b1' -- c1' -- d1' -- e1'                   (new-page/1-refactor)
       \
        a1 --- b1 --- c1 --- d1 --- e1
                                     \
                                      a2 --- b2           (new-page/2-component-a)
                                              \ 
                                               a3 --- b3  (new-page/3-create-page)
```

To solve this, we need to tell git that `a2`'s parent is actually `e1'` instead of `e1`. We need to run `git rebase --onto e1' e1` while we're on `new-page/2-component-a`. This essentially means:

> "change the parent of `a2` from `e1` to `e1'`"

Using more precise language, it means:

> "find the child of commit `e1` that's reachable from the current `HEAD` and make its parent be commit `e1'`" 

I know, this command looks a bit confusing, and it does take some practice to get used to it. A good way to memorize it is to think of it as `git rebase --onto <new-parent> <old-parent>`.

For brevity, I won't go into more details of `git rebase --onto` usage since that warrants its own blog post. However, if you want to really understand what's going on (which I highly suggest you do), check out this excellent explanation from [Stack Overflow](https://stackoverflow.com/a/29916361) or refer directly to the [Git Rebase docs](https://git-scm.com/docs/git-rebase).

Great! We brought the changes from our first branch onto the second one. However, our third branch is still unsynced, and our tree looks like this:

```text
x --- y --- z                                             (main)
       \
        a1' -- b1' -- c1' -- d1' -- e1'                   (new-page/1-refactor)
                                    |\
                                    | a2' -- b2'          (new-page/2-component-a)
                                     \
                                      a2 --- b2
                                              \ 
                                               a3 --- b3  (new-page/3-create-page)
```

We can do the same thing and run `git rebase --onto b2' b2` while checked out to `new-page/3-create-page` and that will bring the changes over to the last branch and we'll be fully synced. This process of syncing all branches in a stack is sometimes referred to as a "cascading rebase".


## Merging PR stacks: top-down and bottom-up

You're done working on your pull request stack, and now you want to merge it back into `main`. There are two approaches to this:
- top-down: merge the base of the stack onto `main`, then its child onto `main`, then the child's child onto `main` and so on until everything is merged. Using our example:
  - merge `new-page/1-refactor` onto `main`
  - merge `new-page/2-component-a` onto `main`
  - merge `new-page/3-create-page` onto `main`
- bottom-up: merge the last PR (the "leaf") onto its parent, then that into its parent and so on until you've collapsed all PRs onto the base of the stack. Then, you merge the base PR onto `main` as one big chunk containing all the children. In our example:
  - merge `new-page/3-create-page` onto `new-page/2-component-a`
  - merge `new-page/2-component-a` onto `new-page/1-refator`
  - merge `new-page/1-refactor` onto `main`

The vast majority of times, I find myself merging **top-down**, for two reasons. 

The first reason is that (95% of the time) the whole purpose of PR stacks is to split a big piece of work into multiple mergeable units. Since that is usually the primary goal, it doesn't really make sense to merge bottom-up, as that combines all pull requests into the base PR, and I'd end up merging one big chunk of code into `main` nonetheless.

The second reason I usually merge top down is that the parent PRs are usually more "mature" than its children. This is only natural, since, by definition, the parent has to have existed for longer than is child, and so it has probably had more rounds of review and feedback incorporated into it. It's not uncommon to merge a parent PR but wait before merging its children if they're not yet ready to go into `main`.

All that said, I will sometimes opt to merge bottom-up if the reason for creating a stack was solely facilitating/isolating reviews, but it still makes sense to merge it all as one single unit of work. If that's the case, I'll usually wait until all children are _fully_ reviewed and approved to collapse the stack back into the root PR.


## Using `git rebase --onto` when merging top-down

Let's say you chose to merge the PR stack top-down, and you're done merging the base PR (`new-page/1-refactor`). You navigate to the second PR, but you notice now there are a ton of merge conflicts. Let's take a look at what our branches look like after the merge


```text
x --- y --- z ------------------------ m                    (main)
       \                              /
        a1 --- b1 --- c1 --- d1 --- e1
                                      \
                                       a2 --- b2            (new-page/2-component-a)
                                                \ 
                                                 a3 --- b3  (new-page/3-create-page)
```

As expected, `git merge` replays all the commits from `new-page/1-refactor` onto the head of `main` (commit `z`), and records the changes onto a new "merge commit", named `m`. This explains why there are merge conflicts: both `m` and `[a1, b1, c1, d1, e1]` contain the same exact same changes, and if you tried to merge your branch onto `main`, you'd be reapplying the same patches that are already contained by `m`, plus `a2` and `b2`.

You might be guessing where we're heading towards: we need to rebase `new-page/2-component-a` so that it sees `m` as its new base instead of `e1`. You've probably guessed that we'll run `git rebase --onto m e1`.

After our rebase, the branch structure looks like this:

```text
x --- y --- z ------------------------ m                      (main)
       \                              / \
        a1 --- b1 --- c1 --- d1 --- e1   a2' -- b2'           (new-page/2-component-a)
                                     \
                                      -- a2 --- b2
                                                  \ 
                                                   a3 --- b3  (new-page/3-create-page)
```

We've successfully removed the duplicated patches, and we won't get any conflicts when we merge `new-page/2-component-a` onto `main` since we're now only trying to merge `a2` and `b2`. We're ready to merge the second PR! Note that in this case we won't bother syncing `new-page/3-create-page` since, after merging its parent onto main, we'd have to resync all over again. 

After merging the second PR, our tree looks like this
```text
x --- y --- z ------------------------ m ---------- m2         (main)
       \                              / \          /
        a1 --- b1 --- c1 --- d1 --- e1   a2' --- b2'           (new-page/2-component-a)
                                     \
                                      -- a2 --- b2
                                                  \ 
                                                   a3 --- b3   (new-page/3-create-page)
```

Now, let's sync the third PR by first checking out to it, then running `git rebase --onto m2 b2`. Our tree is now
```text
x --- y --- z ------------------------ m ---------- m2             (main)
       \                              / \          /  \
        a1 --- b1 --- c1 --- d1 --- e1   a2' --- b2'   a3' --- b3' (new-page/3-create-page)
```

We can finally merge the last PR. After that, the tree is
```text
x --- y --- z ------------------------ m ---------- m2 ---------- m3             (main)
       \                              / \          /  \          /
        a1 --- b1 --- c1 --- d1 --- e1   a2' --- b2'   a3' --- b3'
```

Hooray! :confetti_ball:


## Using external tools to manage stacked pull requests

I just taught you how to manually manage a PR stack. As you've probably noticed, it's not the easiest thing in the world. You have to keep all the branches in sync, be careful to merge and rebase in the correct order, make sure you track of all the reviews and incorporate feedback all at the right place. It's a lot of different commands to run, and all of this can be a lot on the mind. It's error-prone and time-consuming.

Luckily, very smart people have also faced this problem and created out-of-the-box solutions for managing PR stacks automatically. This blog post is not a tutorial on any of those tools, so the following is a brief summary of each of them. You can check them out by yourself and choose which one fits your workflow best. Personally, I like to use Git Town for its simplicity, but I know excellent engineers which use Graphite, others who prefer to go the manual `git rebase` way, and others who created their own bespoke collection of bash and python scripts to manage stacks. There's no right or wrong as long as you choose what works for you and your team.

- [Git Town](https://www.git-town.com/): A high level Git CLI which adds a few extra commands such as `git town sync` for cascading rebases across a stack, `git town propose` to automatically open a PR for the current branch and `git town ship` to merge a PR and rebase its children.
- [Graphite](https://graphite.dev/): It started out as tool which only managed PR stacks, but over time it's evolved extra bells and whistles such as its own review interface, insights page, merge queue and VSCode extension. It requires mandatory signup and larger companies are required to pay for it if they want to use it.
- [spr](https://ejoffe.github.io/spr/) and [ghstack](https://github.com/ezyang/ghstack): these tools are _very opinionated_ as they completely abstract away the concept of "branches" and force you to commit directly to `main`. Then, they open one pull request per commit and that's your stack. Personally, I think this approach is a little too limiting because in real life you're often going to have to implement larger things that can't be easily separated into one commit at a time. Some people seem to like this, so I'm leaving these here so it's up to you to judge for yourself!


## Should you stack another PR or adding more commits to the existing PR?

Getting the right balance between adding commits versus opening a new PR is very reliant on team/project dynamics and taste. There is no clear line between what's right and what's wrong.

In the example I chose, I divided the feature into 3 steps. You could argue this could be divided even further, since the first PR groups refactoring of 3 different components and that could be split into 3 separate PRs. You wouldn't be wrong! However, as with many other things, in the real world you should decide at which point to stop. Stacking too many PRs can also create its own set of problems which decrease your productivity:
- The more PRs you open, the more things you need to keep track of, and the more reviews you'll need to ask for
- You potentially stagger the discussion across multiple different pages
- If you're _too_ granular, you're just introducing extra overhead and thrash without gaining much out of it. Remember, the problem we were trying to solve in the first place is allowing separate merges/reviews

When considering whether to split your current changes into another pull request, first ask yourself the following questions:
- Can this new piece of work be merged separately? Does it make sense to do so? If the answer to any of these is no, don't stack another PR.
- Is there any advantage to merging this new chunk of work separately?
  - Maybe merging it separately is less risky
  - Maybe you need to/want to roll out your changes slowly
  - Maybe your work is blocking someone else's work, and so you need to merge it as soon as possible
  - Maybe your changeset getting too large and you want to avoid merge conflicts
  - Maybe you want to logically isolate the review and discussion of both parts of your work to keep each review more focused
  - Or maybe there's no advantage to it and you're just adding extra complexity for no reason. If that's the case, don't stack another PR.

Personally, I try to follow these very general guidelines:
- If my PR is blocking for a collaborator's work, I'll ask for review and merge it ASAP, and I continue the rest of the work in a separate stacked PR.
- If my PR is getting longer than 6-8 commits and there's still significant work to be done, I'll open a stacked PR
- If the rest of the work is very different and would distract from the overall "theme" of the PR, I'll open a stacked PR
- If I have to do some kind of bulk work that will change a ton of files all in the same way (renaming, formatting, adding a new parameter to all callsites etc), I'll open a stacked PR just for that. This avoids polluting the changeset of other PRs in the stack and keeps reviews cleaner.


## A common pitfall of stacked pull requests

As the name suggests, a stack of pull requests is a stack (duh): you can't access the bottom element without first removing all the elements which are on top of it. As such, you won't be able to merge a PR that is down the stack without merging its parent first. In other words, there is a strict merge order: first you merge the parent, then its child, then the child's child and so on.

Due to this, avoid stacking PRs when it's not necessary. If you're making two changes that are completely independent of each other (i.e they can be merged in any order), you'll be better off by branching both of them off of `main` to keep this independence. Otherwise, you'll be forced to merge your PR stack in order, which could be blocking and unnecessarily delay merges in case the child is ready before the parent.


# Combining it all

After learning all of this, your new, enhanced git workflow should look like

- Check out to a new `my-feature/1-chunk-of-work` working branch
- Make changes, commit, repeat
- Sync with `main` by running `git rebase main`
- Fix your past mistakes and edit past commits with `git rebase -i` and `git commit --fixup`
- Use `git reflog` in case you screw up a rebase or need to inspect dropped commits
- Once (and if) you start working on a new chunk of work that can be merged separately, create a child branch `my-feature/2-my-other-chunk-of-work`
- Sync all your working branches manually with `git rebase --onto`, or with tools like Git Town or Graphite
- Once it's approved, rebased and ready to be merged, merge the first PR into `main`, and rebase the second PR onto the updated main by using `git rebase --onto` (or Git Town/Graphite)
- Merge the second PR after the base PR has been merged.

That's all! I hope this helps you write better commit histories and pull requests as much as it helped me! :heart:

\- Lucas


# Acknowledgements

I'd like to thank Tim Connor, the great Senior Staff software engineer that taught me all of this during a [dbt Labs](https://www.getdbt.com/) company off-site. This knowledge has saved me countless hours of head scratching and greatly improved the way I use git and submit PRs for review.
