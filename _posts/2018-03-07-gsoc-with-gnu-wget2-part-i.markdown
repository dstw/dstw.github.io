---
layout: post
title: "GSoC with GNU Wget2 - Part I"
date: 2018-03-07 03:21:00 +0700
comments: true
redirect_from: /blog/2018/03/06/gsoc-with-gnu-wget2-part-i/
categories: open-source programming collaboration gsoc
---

This series of writing summing up of what I did in Google Summer of Code (GSoC)
2017\. GSoC 2017 be held in three period of time. In this part I will tell you
about the first period.

### Community Approaching

After looking for organizations to work on, I decide to choose GNU Wget2 (part
of GNU Project). Why I choose this organization? Some reason I can say:

* The codebase is small (but not the scope, for me), it gives me advantage
  that it is easy to deploy on my local machine.
* It use C as its primary programming language. At this time, I would like to
  learn about C (Linux was influenced me) on real world application. I was
  starting to learn C but just on the theory. So, working on Wget2 could help me
  to learn C with practice.
* The GNU Project reputation is well recognized in Open Source community so
  working on their project will give me excellent opportunity to learn.

I immediately take a look on their repository. I try to clone their repository
and build on my machine. GNU Wget2 use mailing list as their primary
communication channel, so I sent greeting to them about what project I
interesting at.  
The contributor of the project is so friendly. They give me some clues about
what should I do. I should do some _microproject_ work to show effort from me to
understand their workflow and also their codebase. That means, I have to send a
patch (there was Github pull request at this time) to the project.  I choose to
work on adding new unit test to function `wget_robots_parse()` [0].  Just like
most of open source contribution, it took some process of revision of patch so
it can be merged to upstream. I respect the reviewing process, it help me learn
a lot. Also from now I learn a lot to use _GDB (GNU Debugger)_ to debug
application.  
At first I choose to use subject "Design and Implementation of a Framework for
Plugins". But I rethink again because what I do in micro project is about
Testing. Finally I change my subject so I take "Design and Implementation of
Test Suite Using Libmicrohttpd" as my project. I send my proposal to project's
mailing list [1]. After some reviewing and revision process, I officially sent
my proposal to GSoC homepage and my proposal was accepted.  
The project aim to use Libmicrohttpd as test suite for Wget2. I planned to
complete this by change on function `wget_test_start_server()` also
`wget_test_stop_server()` from src/libtest.c of Wget2. With this
approach, I don't need to change existing test suite which call the internal
server code through functions mentioned above.  
I've count there are 36 test file which use
`wget_test_start_server()`. I must ensure all the test passed.  And
for installation prerequisite, I must ensure that Libmicrohttpd are included
when building Wget2 binary. Then I need to modify configure.ac. I will give
proper warning about this requirement. There is a section in README.md where I
must explain to user to provide Libmicrohttpd to make all test running
correctly.  With Libmicrohttpd I can add new test using feature that not yet
implemented in old server code, but ready on Libmicrohttpd, such as HTTP
authentication and concurrent request checking.

### Community Bonding Period

During this period, I spend my time to learn codebase about Wget2 and
Libmicrohttpd. I was busy to learning them, that I forget to stay active in
community. To prevent mentors give me minus points, I try to work on an issue. I
try an easy one, adding enhancement for capabilities of file extended
attributes. In short words, Wget2 could save target file's extended attributes
if filesystem support it. It is like the feature on Wget 1.x. To solve this
issue, I have to dive a little bit on Wget2 codebase. That's give me a view
about how Wget2 works.

### 1st Week

There comes the first week of my official coding period. I was work on some
basic tasks for this projects, such as modify the build tools. I have modifed
configure.ac to include Libmicrohttpd into Wget2. I just include the package,
and not adding, modifying or removing anything else yet.  
With Libmicrohttpd becomes mandatory package to install before building Wget2
binary, there must be proper warning about this requirement, otherwise the
building process will fail. I have add oneliner information into README.md.
Actually, I was misunderstood about this. Christian Grothoff then give me
suggestion to correct it. Dependency to Libmicrohttpd should not become
mandatory. The solution it to make conditionally-compile and run the tests only
if Libmicrohttpd is present while build Wget2. Libmicrohttpd does the same for
Libcurl. Making the dependency optional also avoids the obvious possibility of
circular dependencies if we ever were to add Libwget2-based tests to
Libmicrohttpd.  
Tim Ruhsen also reminds that I should to skip the appropriate tests if
Libmicrohttpd is not installed/ available. I should definitely avoid to use
Wget2 legacy server code in Libwget and focus on client functionality instead.
Because after we successfully integrating Libmicrohttpd, we should remove the
legacy server code. I finally understood what they mean and will fix this the
next week.  
On the CI/CD section, I have ensured that all make check passed on several
testing machine including: Debian/GCC, Fedora/Clang, MingW64 and OSX.
Fortunately, most of those OS have provided Libmicrohttpd package on their
respective repository, except MingW64 (the representation of Windows build).
Especially for MingW64 build, because I haven't found the correct package for
Libmicrohttpd, I include Libmicrohttpd by download the source and compile
manually. What I've done is add the following lines on .gitlab-ci.yml:

``` shell
before_script:
- dnf -y install wget
- wget http://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.55.tar.gz
- tar zxf libmicrohttpd-0.9.55.tar.gz && cd libmicrohttpd-0.9.55/
- mingw64-configure --prefix=/usr/x86_64-w64-mingw32/sys-root/mingw/
- mingw64-make -j$(nproc)
- mingw64-make -j$(nproc) LOG_COMPILER=wine install
- cd -
```

I asking if my step was right. Tim give me advice on that MingW64 issue, that I
should provide a script (e.g. in contrib/ folder) that will
downloads/builds/installs Libmicrohttpd. That script then added to the CI runner
YAML file(s) so we can test on MinGW64 as well. This will keep us to stay run as
many tests as possible. He suggest that I should move my MinGW64 build script to
contrib in the end of GSOC as a cleanup.  
On the other hand, Evgeny Grin tell me there is a package for Libmicrohttpd on
MinGW64, it provided on Msys2 project and has HTTPS support. Another solution,
is to use W32 binary build provided on GNU mirrors, but it has not HTTPS
support. I keep this information for later decision. For now, I should focus on
the test code first.

### 2nd Week

Things get more interesting. Because in this period, I will starting to work on
Wget2 main testing function. I started on `wget_test_start_server()`.
I must change that function in order to call Libmicrohttpd as service for
`wget_test()`. Workflow to resolve this:

* Disable initial process for HTTP server socket.
* Disable `_http_server_thread`, instead call new function which call
  Libmicrohttpd.
* Create `_http_server()` function, wrapper for Libmicrohttpd. There
  is also function `ahc_eco()` which use to create proper HTTP
  response.

Some issues I found in this period:

* Decide what the best threading model for Libmicrohttpd. I currently using
  `MHD_USE_INTERNALLY_POLLING_THREAD` which use external select. I still check
  the comparison with legacy code that use Wget2 API `wget_thread_start`.
  Darshit Shah give me a clue, that I should choose any mechanism that uses
  `select()`. Then we can change the threading model at a later stage if it
  turns out to be a bottleneck. `epoll` is Linux-only and even `poll` isn't
  always available, so as long as I choose a `select` based implementation, it
  should be fine. Evgeny said that I can use `MHD_is_feature_supported()` with
  `MHD_FEATURE_POLL` and `MHD_FEATURE_EPOLL` to check for supported polling
  functions.  Alternatively, with latest versions of Libmicrohttpd I can use
  `MHD_USE_AUTO` and Libmicrohttpd will choose `select()`, `poll()` or
  epoll-based polling automatically.
* I make the `http_server_port` still hardcoded. Darshit tell me that this
  becomes important note. The port should be a random number. Usually, passing 0
  in the port number makes the kernel choose an open one for it. Having a
  randomised port is important to ensure that multiple runs don't step on each
  other.  
  I think still don't know how to accomplish this. Maybe, it just my
  understanding that when I pass 0 in MHD port number, the result is still 0.
  Another approach, when I look into the old code, it generate port number by
  calling `wget_tcp_get_local_port()`. But, I need to call `wget_tcp_init()` and
  `wget_tcp_listen()` respectively in order to get proper result.  Conclusion,
  do I need to use existing `wget_tcp_get_local_port()` to get the port, or
  maybe there is a function in Libmicrohttpd to do that.  
  Tim added that all I need is the socket descriptor. How to call
  `getsockname()` + `getnameinfo()` to retrieve the port number I see in
  `libwget/net.c`/ `wget_tcp_get_local_port()`. If Libmicrohttpd doesn't have
  such a function, either try to get the socket descriptor or extend
  Libmicrohttpd with a small function (similar code as in
  `wget_tcp_get_local_port`).  
  Evgeny give more detailed explanations. There are several ways to resolve it:

  1. Initialise socket externally and start `listen()` on it. I can use any bind
     and port detection function. Pass socket `FD` to `MHD_start_daemon()` by
     `MHD_OPTION_LISTEN_SOCKET`.
  2. Use `MHD_start_daemon()` with "0" as port number then use
     `MHD_get_daemon_info()` with `MHD_DAEMON_INFO_LISTEN_FD` to get listen
     socket FD. Use any port detection.  
  3. Use `MHD_start_daemon()` with "0" as port number then use
     `MHD_get_daemon_info()` with `MHD_DAEMON_INFO_BIND_PORT` to get port
     number.  Works with `MHD_VERSION >= 0x00095501` and on platforms that
     support `getsockname()`.  
     I can combine second and third methods.

``` c
  int port_num;

  if(0) {}
#if MHD_VERSION >= 0x00095501
  else if (MHD_NO !=
     MHD_is_feature_supported (MHD_FEATURE_AUTODETECT_BIND_PORT))
  {
    const union MHD_DaemonInfo *dinfo;
    dinfo = MHD_get_daemon_info (d, MHD_DAEMON_INFO_BIND_PORT);
    if (NULL == dinfo || 0 == dinfo->port)
    {
      /* Insert code to handle error. */
      return -1;
    }
    port_num = (int)dinfo->port;
  }
#endif /* MHD_VERSION >= 0x00095501 */
  else
  {
    const union MHD_DaemonInfo *dinfo;
    MHD_socket sock_fd;
    dinfo = MHD_get_daemon_info (d, MHD_OPTION_LISTEN_FD);
    if (NULL == dinfo)
    {
      /* Insert code to handle error. */
      return -1;
    }
    sock_fd = dinfo->listen_fd;
    /* Insert code to use port detection on provided socket. */
  }

```
He add note that socket type used by MHD functions is `MHD_socket`.
`MHD_socket` is `int` on POSIX platforms (Linux, \*BSD, Unix, Darwin) and
`SOCKET` on Windows platform (excluding Cygwin, where `MHD_socket` is `int`).  

* In `ahc_eco()` of Libmicrohttpd, urls data still using static checking for
  matching with requested urls. In other word, it's hardcoded. It needs to be
  changed to dynamic method to accomodate variadic data.
* I still not touched HTTPS yet.
* I ask about what to do with FTP and FTPS functions. Since Libmicrohttpd just
  provide service for HTTP. I was asking, do we need keep the function for
  FTP{s}, or removing it. Darshit told me that we keep the FTP code intact for
  now. Then we should look into different libraries that provide a FTP server in
  C and try to integrate that into your test suite as well. But for this period,
  it is out of scope.
* The test was failed when I try to resolve URL with question mark. For example:
  `/subdir1/subpage1.html?query&param`, when I debug, it return just
  `/subdir1/subpage1.html` so the result is 404 not found. I also check using
  logging example source code provided in Libmicrohttpd tutorial. When I access
  using http client such as Wget2 and Firefox, the result is still the same. The
  URL result omit the query part. I confirm to Libmicrohttpd side about
  this, whether it is intended behaviour or not. Christian said it that's
  intended, for URL parameters/arguments I need to use
  `MHD_get_connection_values()` with `kind=MHD_GET_ARGUMENT_KIND` to inspect
  them.

### 3rd Week

Based on feedback from mentors on the previous week, I made some fixes.

* Remove initial process for HTTP server socket.
* Create `_http_server_start()` function, wrapper for Libmicrohttpd. There is
  also function `answer_to_connection()` which use to create proper HTTP
  response.
* Use select method `(MHD_USE_SELECT_INTERNALLY)` for threading model in
  Libmicrohttpd to get better compatibility.
* `http_server_port` seized automatically using Libmicrohttpd function by
  passing `MHD_DAEMON_INFO_BIND_PORT` or `MHD_DAEMON_INFO_LISTEN_FD` parameter
  to `MHD_get_daemon_info()`.
* Using iteration to parse urls data in `answer_to_connection()`. This
  guarantee we can pass any variadic data to Libmicrohttpd and prevent
  segmentation fault.
* Fix `answer_to_connection()` function to create proper HTTP response (to deal
  with parameters and arguments on url, to add proper HTTP headers).

### 4th Week

Finally it comes to the end of the first period. I report to mentors what I've
done in this week.

* I have finished modify configure.ac to include Libmicrohttpd into Wget2.
* I have ensured that all make check passed on several testing machine
  including: Debian/GCC, Fedora/Clang, MingW64 and OSX.
* Started working on `wget_test_start_server()`.
* I must ensure that all test suite running correctly. To give better
  visualization, I've created spreadsheet about summary of test file which use
  `wget_test_start_server()`. Currently, as far as I know, it reaches 87.5% (28
  of 32). I need to complete all test suite reach 100%.

After I send it to the mailing list, I have got many feedback from my mentors
for this report.

When Darshit trying to check my code, he has noticed that I was working only
on a single branch and have only one commit until the date. This much work is
not enough for the mid semester evaluation. He asks me to split my work into
smaller commits which are discrete units of work. I follow up this with breaking
down my previous one big commit into smaller ones so it could represent my work
in period of time.  
About the commit style, he gave me a clear clue. My commit messages are pretty
hard to read. Also, he asks to maintain the standard style of commit messages.
At Wget, the commit messages are converted into a ChangeLog, hence my message
should be written in that format.

```
git commit

Commit Message title (60 chars)
[Blank Line]
* file_changed(function_name): Description of change
* file_changed(function_name): Description of change

Any other details I want to write
```

The commit messages I have written are actually fine, but when it exceeds 60
chars in the first line, I should convert my message to the longer format he
just described.  
Another thing, he asks to not force push to my (_master_) branches. There are
other people that pull from these branches to check in on the code. If I force
push, it messes the repository up for all. Force pushing should only be reserved
as a last resort. However, in this case, when I rewrite the commit messages, I
should force push to my own current branch. That will allow me to maintain that
branch later.  
I realize that my commit messages are bad practice indeed. I check it through
`git log --pretty=oneline`. I have made some changes about that.
But, when I want to push my changes online, it rejected by git. That's why I
usually force push. I make sure I just make a changes to my own current branch,
not the master one.  
The CI system states failures. He asks whether this is an expected failures. I
said that it not intended to be failure. On my machine which use Ubuntu/GCC, I
can run `make check`, and some test could pass. Same like on Gitlab CI that use
Debian/GCC. Based on CI artifacts, most of them failed on `make check-valgrind`
that I miss to check before. I still find out how to deal with this memory leak
error. Actually, I still stuck in this. Then I ask, how to debug using Valgrind.
I read the error logs and it still it pointed me out of nowhere. He said that I
really should discuss these issues with them more. That is exactly why they are
here to mentor me. Valgrind's memcheck tool which is what is being used in the
`make check-valgrind` is a memory leak checker. He will look a little into it
once he can get the basic tests running on his machine. I asked to be focus on
that first. He said we will look into the memory leaks in a while. Most probably
the leaks happen because either I lost the pointer to some allocated memory or
are using the libmicrohttpd API wrong and forgot to `free()` some data structure
returned by it.  
I analyze how the Valgrind check was failed. On my system which using
Ubuntu/gcc, some tests are failed, while some not. Through this, I will analyze
based some available pattern.  Before I found that Valgrind check error, my work
with test suite stopped at how I must deal with timestamp/modified properties in
`test-wget-1`. Another test waiting to finish: `test-metalink` (looping
undefinitely), `test-i-https` (https implementation) and `test-auth-basic`(basic
authentication implementation).  
He also sees that ./configure always tries to link Libmicrohttpd. But Wget2
should not depend on that. It should try to add the linker flags *only* when
trying to compile the test suite. The Wget2 binary should not use that flag. I
realize that was there are flaws in my code. I will figure out after some work
against other test files.  
On his local system, not a single test passes. `make check -j$(nproc)` causes a
segmentation fault and a simple `make check` gets stuck on test-wget-1 and never
progresses ahead. He asks me to fix the branch and/or provide them with
information on how to make the tests work. I look at CI system with Fedora/Clang
and indeed the result is all segmentation faults. I still have no clue about
that. With this case, I cannot claim that I have almost 85% tests passing
already. From his perspective, it is still 0%.  
In fact, all the tests that my spread sheet marks are passes, end up causing a
segfault on his system. This seems to stem from the fact that the
`http_server_tid` is stored as "0" and I try to call `pthread_cancel` on that
value. I asks for more detail information about this. I have use some advice
from Evgeny Grin how to deal with dynamic port. Even when I use hard coded port,
the result is still the same. He explains that he don't think this is connected
to the port number at all. If it is, then my variable naming scheme requires a
serious change. This issue is happening because I have never stored the tid of
the server in the variable, but I still try to cancel that thread id. After I
take a look on my code again, I realize something wrong. Darshit was right. That
segmentation fault error happened because I forgot to remove
`wget_thread_cancel`(`http_server_tid`) which is used by the old code at server
termination time. After I remove that line, the CI runner of Fedora/Clang
passes some tests, not all test were resulted segmentation fault (14 of 32
pass, while 4 skipped). The other fail because likely Valgrind check error.  
Darshit asks me to make sure that my code works as I have mentioned at least one
day before the deadline for the evaluations. On my Week 3 report I planned to
get 100% of passing all of the test suite. But, until Week 4, my job still not
yet all done (even excluding other error generated outside `make check` on
Debian/GCC system). I realized I have missed my (personal) target. Hence, he
asks me where do I think I missed out which caused me to fall behind. In
hindsight, he asks what according to me could have been done better. So, I and
they should be look into this together to see where it can be improved. He asks
what topics were I most stuck on, and why did it take me so much time. Also, if
I stuck on any single point for more than a day, he asks to send they a message.
There is a very good chance that they might be able to help me out and fix it
much faster.  
Darshit sends a message to both Libmicrohttpd maintainers, Christian and Evgeny:
that the issues in Valgrind seem to come from within Libmicrohttpd. One of the
blocks reported by Valgrind is:

```
=26290== 72 bytes in 3 blocks are still reachable in loss record 6 of 6
==26290==    at 0x4C2BEEF: malloc (in
/usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==26290==    by 0x7E06608: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x7E077FB: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x7EBEFB1: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x7EBF02F: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x7E0655F: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x7E06764: ??? (in /usr/lib/libgcrypt.so.20.1.7)
==26290==    by 0x6634CF8: ??? (in /usr/lib/libmicrohttpd.so.12.43.0)
==26290==    by 0x400F339: call_init.part.0 (in /usr/lib/[1]ld-2.25.so)
==26290==    by 0x400F445: _dl_init (in /usr/lib/[2]ld-2.25.so)
==26290==    by 0x4000D39: ??? (in /usr/lib/[3]ld-2.25.so)
==26290==    by 0x9: ???
```

So, he guess is that this is some memory that is being allocated by
Libmicrohttpd.  The test code does indeed seem to call `MHD_stop_daemon()` which
should ideally ensure a clean exit. He asks, where did the implementation go
wrong.  
Christian said this is a bit hard to evaluate without the debug symbols. Most
likely it would be a false-positive from a globally allocated buffer of
Libgcrypt's initialization sequence. He asks to download and install debug
symbols for Libgcrypt and ideally Libmicrohttpd. It is unlikely to be
Libmicrohttpd fault, as except for responses there are no Libmicrohttpd buffers
my code would have to free.  
He also asks me to ensure that I have followed the Libmicrohttpd API and
documentation correctly. I have followed Libmicrohttpd API documentation and
tutorial for implementation carefully.  

### First Period Evaluations

After finish my 4th period report, and discuss with mentors about my
difficulties I found, I also must complete first evaluation report via GSoC web
app. For what I can just remember, it just a dashboard which asks my feedback
about the organization I working with. The real report is actually done via
email.  
The other day, GSoC announce the evaluations. The result, I passed the
evaluations, but with warning. Mentors give me an important messages about what
happened and what should I improve. The project is indeed behind schedule. The
mentors still give me a chance by passing me in the hopes that I and they will
be able to improve the communication and get the project back on track. One
major suggestion is that I increase the frequency of my project reports from
weekly to daily. Just a single line at the end of each day stating what I have
done and what are my plans for the next day.

### Conclusion

That was though days with my first time period of GSoC. There was many jobs left
to be finished in the next period. Mistakes happen, but I was learn a lot from
them. Thanks to my mentors who patiently guide me through this process.

Reference(s):  
[0] [https://github.com/rockdaboot/wget2/pull/155](https://github.com/rockdaboot/wget2/pull/155)  
[1] [https://lists.gnu.org/archive/html/bug-wget/2017-03/msg00156.html](https://lists.gnu.org/archive/html/bug-wget/2017-03/msg00156.html)
