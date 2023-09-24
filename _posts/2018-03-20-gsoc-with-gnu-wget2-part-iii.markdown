---
layout: post
title: "GSoC with GNU Wget2 - Part III"
date: 2018-03-20 05:21:00 +0700
comments: true
categories: open-source programming collaboration gsoc
---

This is the last part of my GSoC 2017 journal trilogy. In this period, my
patches of this project must be sent so they can be reviewed. In order to pass
this final evaluations, I must submitted my work and try to merge it to
upstream codebase. As my previous period, I face with some obstacles but
mentors help me a lot to get rid of it so the project goals was achieved.

### 9th Week

Some progress I made in this week:

* Fix connection handler using chunked transfer encoding
* Fix CI testing on Gitlab MingW64 and Travis OSX
* Add HTTP digest authentication test

I also started working on:

* Fix CI testing on Gitlab MingW64 and Travis OSX
  - MingW64: Need a script for complete setup Libmicrohttpd with HTTPS support
  - OSX: Need more verbose debug since it fails to run HTTPS server
* Add HTTP digest authentication test

### 10th Week

What I have done in this week:

* Fix CI testing on Travis OSX: I must install Libmicrohttpd from source to get
  support for HTTPS. Maybe the Libmicrohttpd package provided by Homebrew
  Package Manager not compiled with HTTPS support.

Tasks which still in progress:

* Fix CI testing on Gitlab MingW64
  - MingW64: Need a script for complete setup Libmicrohttpd
* Add HTTP digest authentication test

### 11th Week

I have some problem with Wget2 build with Libmicrohttpd on MingW64. I have add
some commands to build Libmicrohttpd from source, and it was successfully built
and installed. But when it used to build Wget2, I get this problem:

```
make[2]: Entering directory '/builds/gnuwget/wget2/tests'
  CC       libtest_la-libtest.lo
<command-line>:0:9: error: expected identifier or '(' before string constant
libtest.c: In function '_http_server_start':
libtest.c:595:4: warning: this 'if' clause does not guard... [-Wmisleading-indentation]
    if (getnameinfo(addr, addr_len, NULL, 0, s_port, sizeof(s_port), NI_NUMERICSERV) == 0)
    ^~
libtest.c:597:5: note: ...this statement, but the latter is misleadingly indented as if it is
+guarded by the 'if'
     if (SERVER_MODE == HTTP_MODE)
     ^~
Makefile:2512: recipe for target 'libtest_la-libtest.lo' failed
make[2]: *** [libtest_la-libtest.lo] Error 1
make[2]: Leaving directory '/builds/gnuwget/wget2/tests'
Makefile:1552: recipe for target 'all-recursive' failed
make[1]: *** [all-recursive] Error 1
make[1]: Leaving directory '/builds/gnuwget/wget2'
Makefile:1461: recipe for target 'all' failed
make: *** [all] Error 2
```

Based on my analysis, the error may caused by how I embed the Libmicrohttpd.
When I look at the examples, they mostly include "platform.h" in the header.
But, if I look at the docs:

``` c
/**
 * @file platform.h
 * @brief platform-specific includes for libmicrohttpd
 * @author Christian Grothoff
 *
 * This file is included by the libmicrohttpd code
 * before "microhttpd.h"; it provides the required
 * standard headers (which are platform-specific).<p>
 *
 * Note that this file depends on our configure.ac
 * build process and the generated config.h file.
 * Hence you cannot include it directly in applications
 * that use libmicrohttpd.
 */
```

So, I don't use platform.h. I follow example from Libmicrohttpd homepage, which
they just add:

```
#include <microhttpd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
```

Evgeny Grin replied, he thinks they (as Libmicrohttpd maintainer) should
document it more explicitly. `platform.h ` is not installed with MHD and used
in examples only to avoid including a lot of system headers (which might be not
available on some platforms).  
On other CI testing using Debian, Fedora, OSX using either GCC or Clang, Wget2
can build successfully, but not in Mingw64. Also I tried to compile using
Mingw64 on my local system (Ubuntu), and the result is same like on CI.  
Evgeny said, according to quoted error, I might have some configuration problem.
He asks me to ensure me to properly set `--build=` and `--host= ` configure
parameters. Then try to make by `make all V=1` or use configure parameter
`--disable-silent-rules` to get more verbose output.  
This is my build configuration on `.gitlab-ci.yml`:

``` yaml
script:
  - apt-get -y install gcc-mingw-w64-x86-64 binutils-mingw-w64-x86-64 mingw-w64-x86-64-dev pkg-co
+nfig-mingw-w64-x86-64 win-iconv-mingw-w64-dev wine wget
  - cd ..
  - export PREFIX=x86_64-w64-mingw32
  - export CC=$PREFIX-gcc
  - export CXX=$PREFIX-g++
  - export CPP=$PREFIX-cpp
  - export RANLIB=$PREFIX-ranlib
  - export PATH="/usr/$PREFIX/bin:$PATH"
  - export CFLAGS="-O2 -Wall -Wno-format -lpthread"
  - export WINEPATH="/usr/$PREFIX/bin;/usr/$PREFIX/lib"
  # Install Libmicrohttpd from source
  - wget http://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.55.tar.gz
  - tar zxf libmicrohttpd-0.9.55.tar.gz && cd libmicrohttpd-0.9.55/
  - ./configure --build=x86_64-pc-linux-gnu --host=$PREFIX --enable-shared --enable-static --prefix=/usr/$PREFIX --disable-silent-rules
  - make clean
  - make -j$(nproc)
  - make install
  - cd - && cd wget2
  - ./bootstrap
  - export CFLAGS="-O2 -Wall -Wno-format"
  - export WINEPATH="/usr/$PREFIX/bin;/usr/$PREFIX/lib;$PWD/libwget/.libs"
  - ./configure --build=x86_64-pc-linux-gnu --host=$PREFIX --enable-shared --enable-static --disable-silent-rules
  - make clean
  - make -j$(nproc)
  - make check -j$(nproc) LOG_COMPILER=wine
```

And the result:

```
Making all in tests
make[2]: Entering directory '/builds/gnuwget/wget2/tests'
/bin/bash ../libtool  --tag=CC   --mode=compile x86_64-w64-mingw32-gcc -DHAVE_CONFIG_H
-DDATADIR=\"/builds/gnuwget/wget2/data\" -DSRCDIR=\"/builds/gnuwget/wget2/tests\"
-DEXEEXT=\".exe\" -I. -I..  -I. -I../include/wget -I../lib -I../lib -fvisibility=hidden
-DBUILDING_LIBWGET -DWGETVER_FILE=\"../include/wget/wgetver.h\"
-I/usr/x86_64-w64-mingw32/include -DNDEBUG -O2 -Wall -Wno-format -Wno-attributes -fno-PIC -MT
libtest_la-libtest.lo -MD -MP -MF .deps/libtest_la-libtest.Tpo -c -o libtest_la-libtest.lo `test
-f 'libtest.c' || echo './'`libtest.c
libtool: compile:  x86_64-w64-mingw32-gcc -DHAVE_CONFIG_H
-DDATADIR=\"/builds/gnuwget/wget2/data\" -DSRCDIR=\"/builds/gnuwget/wget2/tests\"
-DEXEEXT=\".exe\" -I. -I.. -I. -I../include/wget -I../lib -I../lib -fvisibility=hidden
-DBUILDING_LIBWGET -DWGETVER_FILE=\"../include/wget/wgetver.h\"
-I/usr/x86_64-w64-mingw32/include -DNDEBUG -O2 -Wall -Wno-format -Wno-attributes -fno-PIC -MT
libtest_la-libtest.lo -MD -MP -MF .deps/libtest_la-libtest.Tpo -c libtest.c  -DDLL_EXPORT -DPIC
-o .libs/libtest_la-libtest.o
<command-line>:0:9: error: expected identifier or '(' before string constant
libtest.c: In function '_http_server_start':
libtest.c:595:4: warning: this 'if' clause does not guard... [-Wmisleading-indentation]
    if (getnameinfo(addr, addr_len, NULL, 0, s_port, sizeof(s_port), NI_NUMERICSERV) == 0)
    ^~
libtest.c:597:5: note: ...this statement, but the latter is misleadingly indented as if it is
+guarded by the 'if'
     if (SERVER_MODE == HTTP_MODE)
     ^~
Makefile:2512: recipe for target 'libtest_la-libtest.lo' failed
```

I still don't have a clue about this build error.

Evgeny gives some clues.  I can skip explicit specifications of CC, CXX (not
needed for Libmicrohttpd), CPP and RANLIB as all of them should be detected by
configure if I specify correctly `--build` and `--host`.  
He asks why do I specify `-lpthread` flag for Libmicrohttpd? Libmicrohttpd on
w32 uses native w32 threads by default.  Also Libmicrohttpd should not require
Wine to build. I will fix this, both `-lpthread` and wine. But, I just remember
why I add `-lpthread` as CFLAGS option. When I omit that option, I got this
error messages when I build Libmicrohttpd:

```
libtool: link: x86_64-w64-mingw32-gcc -DWINDOWS -O2 -Wall -Wno-format
-fno-strict-aliasing -o .libs/authorization_example.exe authorization_example.o
../../src/microhttpd/.libs/libmicrohttpd.dll.a -lws2_32
-L/usr/x86_64-w64-mingw32/lib
libtool: link: x86_64-w64-mingw32-gcc -DWINDOWS -O2 -Wall -Wno-format
-fno-strict-aliasing -o .libs/upgrade_example.exe upgrade_example.o
../../src/microhttpd/.libs/libmicrohttpd.dll.a -lws2_32
-L/usr/x86_64-w64-mingw32/lib
upgrade_example.o:upgrade_example.c:(.text+0x74): undefined reference to
`pthread_create'
upgrade_example.o:upgrade_example.c:(.text+0x82): undefined reference to
`pthread_detach'
collect2: error: ld returned 1 exit status
Makefile:796: recipe for target 'upgrade_example.exe' failed
make[4]: *** [upgrade_example.exe] Error 1
make[4]: *** Waiting for unfinished jobs....
make[4]: Leaving directory '/builds/dstw/libmicrohttpd-0.9.55/src/examples'
Makefile:932: recipe for target 'all-recursive' failed
make[3]: *** [all-recursive] Error 1
make[3]: Leaving directory '/builds/dstw/libmicrohttpd-0.9.55/src/examples'
Makefile:414: recipe for target 'all-recursive' failed
```

Evgeny helps with fixed this error by make a change to Libmicrohttpd codebase on
git master branch. He also gives suggestions. I don't need examples to test wget
so I can disable them by `--disable-examples` configure switch. In addition, I
can disable build of documentation by `--disable-doc` to speedup Libmicrohttpd
building.  
But all of them should be not related to error. He then give me an important
point to start resolving my error result [0]. Based from his guidance, this is
what I get:

```
Making all in tests
make[2]: Entering directory '/home/didik/wget2/tests'
/bin/bash ../libtool  --tag=CC   --mode=compile x86_64-w64-mingw32-gcc -DHAVE_CONFIG_H
-DDATADIR=\"/home/didik/wget2/data\" -DSRCDIR=\"/home/didik/wget2/tests\" -DEXEEXT=\".exe\" -I.
-I..  -I. -I../include/wget -I../lib -I../lib -fvisibility=hidden -DBUILDING_LIBWGET
-DWGETVER_FILE=\"../include/wget/wgetver.h\"   -O2 -Wall -Wno-format --save-temps
-Wno-attributes -fno-PIC -Wall -Wextra -Wformat=2 -fdiagnostics-color=always -Wno-format
-I/usr/x86_64-w64-mingw32/include -I/usr/x86_64-w64-mingw32/include -DNDEBUG -O2 -Wall
-Wno-format --save-temps -Wno-attributes -fno-PIC -MT libtest_la-libtest.lo -MD -MP -MF
.deps/libtest_la-libtest.Tpo -c -o libtest_la-libtest.lo `test -f 'libtest.c' || echo
'./'`libtest.c
libtool: compile:  x86_64-w64-mingw32-gcc -DHAVE_CONFIG_H -DDATADIR=\"/home/didik/wget2/data\"
-DSRCDIR=\"/home/didik/wget2/tests\" -DEXEEXT=\".exe\" -I. -I.. -I. -I../include/wget -I../lib
-I../lib -fvisibility=hidden -DBUILDING_LIBWGET -DWGETVER_FILE=\"../include/wget/wgetver.h\" -O2
-Wall -Wno-format --save-temps -Wno-attributes -fno-PIC -Wall -Wextra -Wformat=2
-fdiagnostics-color=always -Wno-format -I/usr/x86_64-w64-mingw32/include
-I/usr/x86_64-w64-mingw32/include -DNDEBUG -O2 -Wall -Wno-format --save-temps -Wno-attributes
-fno-PIC -MT libtest_la-libtest.lo -MD -MP -MF .deps/libtest_la-libtest.Tpo -c libtest.c
-DDLL_EXPORT -DPIC -o .libs/libtest_la-libtest.o
In file included from /usr/share/mingw-w64/include/objbase.h:66:0,
                 from /usr/share/mingw-w64/include/ole2.h:17,
                 from /usr/share/mingw-w64/include/wtypes.h:12,
                 from /usr/share/mingw-w64/include/winscard.h:10,
                 from /usr/share/mingw-w64/include/windows.h:97,
                 from /usr/share/mingw-w64/include/winsock2.h:23,
                 from /usr/share/mingw-w64/include/ws2tcpip.h:17,
                 from /usr/x86_64-w64-mingw32/include/microhttpd.h:108,
                 from libtest.c:48:
/usr/share/mingw-w64/include/objidl.h:12275:2: error: expected identifier or '(' before string
constant
 } DATADIR;
  ^
```

This lead me to `/usr/share/mingw-64/include/objidl.h`:

```
line  | contents
------|------------------------------------
12271 | typedef IDataObject *LPDATAOBJECT;
12272 | typedef enum tagDATADIR {
12273 |     DATADIR_GET = 1,
12274 |     DATADIR_SET = 2
12275 | } DATADIR;
```

Evgeny explains well to me, he thinks the real problem is: after preprocessing
source by precompiler, compiler get:

```
typedef enum tagDATADIR {
     DATADIR_GET = 1,
     DATADIR_SET = 2
} "/usr/some/dir";
```

I need to either `undef DATADIR ` before including problematic header or find
the way how not to include problematic header. Alternatively, I can replace
macro `DATADIR ` by something like `MY_DATADIR`, but it will require more code
changes on Wget2 side.  
I use his last option. I tried to change `DATADIR ` macro on Wget2 using other
`DATADIR` names so they don't clash and it solves the problem.  Actually, there
are not so many code which need to be changed, but I will open issue first on
Wget2 about this [1]. After confirmed this was valid issue by Tim Ruehsen, I
made a merge request [2] and it was accepted.  
Another progress I made this week is adding CI testing to simulate Wget2 build
without Libmicrohttpd installed. This is just a copy of main build process but
without `make check` part, so the Wget2 binary can be built with skipped
testing. I do it by made a branch without Libmicrohttpd installed.

### 12th Week

This week I focused my work on task adding HTTP digest authentication test. I
found an issue regarding HTTP digest authentication not work properly. I tried
to analyzed it first because throw this issue to public. After I sure that my
issue reproducible and make sense, I create issue and merge request respectively
[3][4]. The merge request was accepted.

### 13th Week

At this point I made a merge request regarding my work on this project [4].
After get reviewed by mentors and other contributors, it finally merged. To
complete final evaluations, I must submit final report which contains summary of
my work and patches I made. It could be link to Github Repositository, Github
Gist, Blog Post, Google Drive, etc, which can be accessed from public. I choose
to use Google Drive to save my report. Here is the link [5] to my work if you
want to check.

### Final Evaluations

After making it until 13th week of works, it was time to get announcement. I got
email that said I was passed on this final evaluations. As always mentors give
me feedback.  
He said, I did a good job with the project this year. There are a few small
issues still lying, but they can be easily cleaned up very quick. Over the
period, I have definitely grown as a programmer, however I should be a little
more active in the community.

### Conclusion

GSoC give me priceless experience about how to getting involved in open source
community. I have enjoyed working with my organization over the last few months
and I will continue to work with them and make sure my code that I have worked
so hard on over these last few months is integrated into their code base. I
will try to keep active in community and contributing more while GSoC passed.
I hope my small contribution was useful for all.

Reference(s):  
[0] [https://sourceforge.net/p/msys2/discussion/general/thread/c1aa51f9/](https://sourceforge.net/p/msys2/discussion/general/thread/c1aa51f9/)  
[1] [https://gitlab.com/gnuwget/wget2/issues/250](https://gitlab.com/gnuwget/wget2/issues/250)  
[2] [https://gitlab.com/gnuwget/wget2/merge_requests/264](https://gitlab.com/gnuwget/wget2/merge_requests/264)  
[3] [https://gitlab.com/gnuwget/wget2/issues/237](https://gitlab.com/gnuwget/wget2/issues/237)  
[4] [https://gitlab.com/gnuwget/wget2/merge_requests/265](https://gitlab.com/gnuwget/wget2/merge_requests/265)  
[5] [https://drive.google.com/drive/folders/0B4yUFvLvAUANLU10X19qckZ6UkE?usp=sharing](https://drive.google.com/drive/folders/0B4yUFvLvAUANLU10X19qckZ6UkE?usp=sharing)
