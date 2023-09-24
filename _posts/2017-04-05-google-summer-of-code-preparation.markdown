---
layout: post
title: "Google Summer of Code Preparation"
date: 2017-04-05 12:23:38 +0700
comments: true
categories: open-source collaboration programming gsoc
---

> GSoC (Google Summer of Code), a program where university students spend their
> 3 month summer break coding on an open source project.

That time finally comes. After almost one year I have been lurking, observing,
it's time to decide whether I will participate or not. It's not easy
to find organization who fit with who I am. But anyway, this year, I will try.  
Despite administrative prerequisite, such as proof of enrollment, there are some
technical stuff involving coding activities we need to do, which is fun.
This is a list of what I've done so far in order to be considered as good
candidate:

### Pick a project with programming language that I passionate about

There is a word "code" in Google Summer of Code, meaning programming will be my main
activity in this program. So, I must find which project that has programming
language, that I familiar with. This is to avoid, I will stuck about the tasks
that will assigned to me as a program applicant.  
Actually, it can be done even long ago before the program being announced. I can
look at the GSoC archive. But, sometimes term will change as the organization
announced. Some organization will participate again, while others not. So, for
me it better to prepare after the program announced officially.  
From there, I can see which project fit with my programming language, one that I
love. So, after some effort of filtering and analyzing, then I choose an
appropriate project, I will not find a big technical barrier to pass this program.

### Pick a project that fit with my knowledge

This means, I must look at the project task. With given time frame, which about
just 3 month, does it sound realistic that I will finish my task? That it's, be
realistic is the key. Measure the difficulty of the task. When I face something
very unfamiliar and need more time to learn, I prefer to skip this. Because I
don't want to spend my time in things that I less understand.  
For example, this year, I choose project from GNU Wget that use C as their
programming language base, which currently I am familiar with. And for the task,
I pick idea of "Improvement of test suite using Libmicrohttpd".

### Get enganged with community

After I found the project that I want to apply, I must introduce myself to the
community. This is important step I must do. Because without it, mentor cannot
help me when I face difficulty.  
Find the place where they collaborate. There are some choice like mailing list,
slack, gitter, irc, etc. GNU Wget use mailing list as they medium to
communicate between developer. I send welcome message there, and replied kindly
by other members.
Follow their rules. Basically, there are some rules to be obeyed:
- Use plain English. GSoC is global program and use English as it language for
  communication. So, I need to use English in my conversation.
- Not to send message directly to the mentor or other person in the community.
  This is considered to be rude. Instead, I will use mailing list if I need to
  send a message.
- Avoid top post when I reply an email.

### Learn their environment, workflow and code base

After I "Say Hi!" to the organization, now it's time to dive into their code.
Find the repository. In Wget, their main repositories are hosted in Savannah,
Github and Gitlab. So, I clone the repo and then build the software from source.  
I have successfully built Wget from source. Then I try to trace what the program
do using gdb, to take a look what is going on inside its process.  
I familiarize my self with git, because Wget use git as its version control
system. Also the project uses Github to consolidate patches, open issue, write
wiki, etc. I try to hanging around on there. It fancy design keep me comfort
with environment and workflow of the project.

### Submit a patch, event with a trivial one

This patching section required quite technical skill. For newcomer like me, I
feel a bit confuse about what patch I can submit. Then, with guide from my
mentor, I was given an idea what simple task can be submitted. I pick an easy
one. A testing unit. Because, he mention, that Wget unit testing is still far
from perfect. I will take it for my chance. I try to add a function to reach
more coverage in the project unit test.  
In order to merged into upstream, I need to make pull request on that project.
To be honest, it is my first time pull request in a real open source project.
Of course my patch will not merged automatically to the upstream. Mentor and the
community members will review first before it can be accepted. And after several
step, edit code, re-push commit with guide from my mentor and other contributors
that help me, finally my patch get merged on the upstream. It's a nice feeling
for a rookie developer can have.

### Write a good proposal

After all, to be accepted in the GSoC program itself, I need to upload a
proposal. The proposal must be good enough to be considered accepted as GSoC
candidate. I pick the idea from list project that mentioned in project website.
I create a draft from this and share to other members. I receive a lot of
feedback. Thanks for who was concerned, so I can make improvements to my
proposal. After I feel it ready, I upload my final proposal to GSoC official.
And because the time between the date of proposal submit and the announcement of
accepted applicant is a month long, I will use it to get more familiar with
project code base.

That was my short list for what I do in order to be accepted to GSoC 2017. I
hope this article will help provide an overview of what preparations need to be
done so you can participate in GSoC. Good luck!
