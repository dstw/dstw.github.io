---
layout: post
title: "Workflow on Contributing to Open Source Project"
date: 2017-04-08 13:43:38 +0700
comments: true
categories: open-source collaboration programming
---

In the open source world, different project comes with different workflow, using
different medium. For instance, Linux Kernel development, use mailing list to
gather patches for many developer around the world. The maintainer pick
patches from developer with careful supervision. Then, they maintain release of
the stable version, while Linus Torvalds himself maintain the mainline stream.  
Other project, Git -- a revision control system. It use same model as Linux
Kernel, patches and conversations go to mailing list. With additional part, for
the Continuous Integration system, it is recommended to use Github and Travis
CI. Although they will reject all pull request from there.  
Another example, Wget2 used Github mainly to perform collaboration (Because GNU
not allowed their project to use Github, their move it to Gitlab [0]). Issues are
discussed here. Pull request also getting merged here. Although, it also have
mailing list to discuss issues and problems.  
Here I share my experience how I contribute to open source, so my code could be
merged in the project upstream. For an example, I will write several steps of
my way doing Wget2 project for my GSoC 2017 application. Below are some points I
follow:

### Obtain Source Code

Startup procedures:

Fork project so you can modify yourself via Gitlab interface.

``` shell
git clone https://gitlab.com/dstw/wget2
git remote add upstream https://dstw@gitlab.com/gnuwget/wget2.git
```

Create an own branch for each 'task' you are working on and make your commits
within it. When done, sync with upstream, rebase/merge and create your patches
via format-patch.

Do not make changes to your branch 'master' - this should always reflect the
original repo. This makes it easy for you to update your 'master' branch with
changes from the 'original/upstream' master branch on Savannah.

Sync your Gitlab repo with upstream:

``` shell
git checkout master
git fetch upstream
git merge upstream/master
```

Sync your 'new-feature' branch before generating patches:

``` shell
git checkout new-feature
git rebase master
git push -f  # pushing updated tree to your Gitlab new-feature branch)
```

Assume 'new-feature' is a private branch where you can do all the dirty things
that you shouldn't on public/shared branches. As soon as your patch has been
accepted, remove the branch 'new-feature' locally and from Gitlab.

### Build the Executables

As prerequisite, install all mandatory software as listed in README.md.

### Find the Problems

I usually look at Wget2 Gitlab issues pages to find issues and problems which
appropriate to newcomer like me.

### Make Changes to the Code

In this part, my knowledge in coding skill are truly challenged. Wget2 use C as
its programming language. In my case, I just add some test based on requirements
and of course with help from other contributors.

### Create Pull/Merge Request

After I feel my code ready to merge, I push my commit to upstream and make a
pull request from Gitlab interface.

### Wait for Review

After create a merge request, I wait for feedback, criticism, suggestions, etc.
from other developer.  
Respond to it! and follow up with improved versions of your change. Even for a
trivial patch you shouldn't be surprised if it takes two or more iterations
before your patch is accepted. This is the best part of participating in the Git
community; it is your chance to get personalized instruction from very
experienced peers!

### Propose New Change

After getting a review, do some improvement. After all, make a new commit. Here,
there is option to overwrite my existing commits or create new merge commits to
keep history.

``` shell
git push origin new-feature
```

If there is a conflict, Git will ask me to resolve the problem this. Then, try
to push and merge commits. This won’t remove any existing commits. And if I
don’t want to keep any commits history, I usually force to push:

``` shell
git push -f origin new-feature
```

~~This is considered a little bit harmful. Before I do this, I will make sure
that I know what I do. Also, forcing commits is should only do if it believed to
final commits or at least near final, I should not do any more change to it.~~ 
Just make sure that I don't force push to the **master** branch of the project,
otherwise, is safe.

Finally, after some iteration, if project maintainer agree with my changes, then
they will merge it into their repository. If not, I should be take a look on my
proposed changes more carefully, what could be wrong and what should be fixed.
And the best of it all, I could learn a lot from the process.

References:  
[0] (http://lists.gnu.org/archive/html/bug-wget/2017-04/msg00052.html)
