---
layout: post
title: Introducing git-fastclone
---

Lightning fast, repeated git checkouts that are especially useful for CI. Get it [here](https://github.com/square/git-fastclone).

## Why fastclone?

When your application and organization are growing, module boundaries get redefined. This can take the form of splitting large components, or combining and rewriting several related smaller components. When pursuing these efforts, you may find yourself having many git submodules and a monolithic build. Ever-increasing build times can make you more frustrated and unproductive, and the cost of smaller tasks can be magnified if repeated on many machines.
Fastclone alleviates these problems by creating a reference repo for every repository and their submodules, all in a simple ruby gem that can be run by typing git fastclone. It updates mirrors from origin and clones them into the target directory, working quickly with a recursive and multithreaded approach.
The first clone will take some time, but Fastclone shines when there are repeated clones on build slaves or multiple checkouts of the same repo:

## Under the hood

In order to get to a clean checkout, we keep our .git directories around and do a quick check for incremental updates with the git server. After that, we perform a parallelized assembly of a proper working copy at the revision we asked for.

We checkout without a working tree using `git clone --mirror`. Since there is no working tree to update `git fetch --all` will always overwrite the local state with the remote state. These reference repositories have full history. Both git clone and git submodule init can be configured to use a reference repository over the remote by using `--reference`. By putting these all together, we get a recursive checkout with full history.

The rest of handling submodules is thread management and caching the contents of .gitmodules for each repository. Caching the submodule dependency tree allows us to spin all the update threads early in the clone operation. Since submodule links don’t change that often, prefetching submodules we’ll need from the git server gives a significant speedup over fetching lazily.

The submodule and git checkout caches are kept in `/var/tmp/git-fastclone`. This is configurable via an environment variable.
A note about submodules: getting a clean checkout for CI or development purposes is an absolute bear. You can git pull, but you don’t get submodule changes — and maybe some of your submodules have submodules. It gets more interesting when a teammate deletes a file in a submodule and `git submodule update --init --recursive` won’t succeed either. These sorts of edge cases become worse as you have more developers and more submodules. As your repository gets bigger, it also takes more time to clone. Many simple timing optimizations find themselves at odds with the edge cases and impact the overall reliability of your checkout.

## Internal use

Square uses git-fastclone as part of our iOS and hardware CI systems. Being able to quickly clone into an empty directory, saves us time and ensures we always know the starting state for our builds — no matter what has happened in previous builds. This in turn increases the reliability of the system overall and benefits our engineers.

## Maintenance

Fastclone uses git interfaces that are mostly stable. Future changes to git will most likely not impact fastclone.
We will be publishing improvements and bug fixes to git-fastclone as we write them. Pull requests for additional functionality are most welcome, and we’d love to hear if you find our little tool useful.

Get it [here](https://github.com/square/git-fastclone).

Acknowledgments to [Michael Tauraso](https://github.com/mtauraso) for the earliest versions of fastclone.
