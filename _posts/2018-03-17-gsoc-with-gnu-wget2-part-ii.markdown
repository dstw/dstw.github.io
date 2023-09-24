---
layout: post
title: "GSoC with GNU Wget2 - Part II"
date: 2018-03-17 03:21:00 +0700
comments: true
categories: open-source programming collaboration gsoc
---

This is story about my second journey in the Google Summer of Code (GSoC) 2017.
From the previous phase, I made some basic progress. I also got many unresolved
tasks to be done. Started on this week, I try to follow Darshit Shah advice to
increase my report from weekly to daily. However, in reality, it was difficult
to create daily report, there must be day that I report nothing because what I
do was actually stuck in my problem.  
What I have got so far:

Started working on `wget_test_start_server()` to use Libmicrohttpd as HTTP
server. Workflow to resolve this:

- Remove initial process for HTTP server socket
- Create `_http_server_start()` function, wrapper for Libmicrohttpd. There is
  also function `answer_to_connection()` which use to create proper HTTP
  response
- Use select method (`MHD_USE_SELECT_INTERNALLY`) for threading model in
  Libmicrohttpd to get better compatibility
- `http_server_port` seized automatically using Libmicrohttpd function by
  passing `MHD_DAEMON_INFO_BIND_PORT` or `MHD_DAEMON_INFO_LISTEN_FD` parameter to
  `MHD_get_daemon_info()`
- Using iteration to parse urls data in `answer_to_connection()`. This
  guarantee we can pass any variadic data to Libmicrohttpd and prevent
  segmentation fault
- Fix `answer_to_connection()` function to create proper HTTP response:
  - Handle arguments (query string) on URL
  - Handle redirection
  - Handle chunked transfer encoding
  - Handle directory creation
  - Handle URL with IRI object
  - Handle URL with IDN hostname
  - Handle URL with query string which contains space
  - Handle If-Modified-Since header
- Fix segmentation fault error when build on Fedora/Clang CI runner. This
  caused by former variable `http_server_tid` that forget to removed

Things which would be done in the upcoming week:

* There are still problem occurred when running `make check-valgrind` on some
  tests. This prevent me to pass the CI test
* Fix all remaining unfinished test covered by Libmicrohttpd code:
  - test-wget1.c: add capability for HTTP 406 Range Not Satisfiable
  - test-auth-basic.c: add basic authentication test
  - test-metalink.c: looping
  - test-i-https.c: add HTTPS test
* Fix Libmicrohttpd depedency on Wget2, just required when need to run test
  suite

### 5th Week

I started my week with ask a question into my mentors. I try to resolve the
check-valgrind problems. What I do so far, I split tests into two groups: pass
and fail. I pick one from the fail group, take one from example,
test-directory-clash. From the valgrind log it shows:

```
==30425== ERROR SUMMARY: 24 errors from 2 contexts (suppressed: 0 from 0)
==30425==
==30425== 4 errors in context 1 of 2:
==30425== Thread 2:
==30425== Invalid read of size 1
==30425==    at 0x556F570: __strcmp_sse2_unaligned (strcmp-sse2-unaligned.S:24)
==30425==    by 0x4E6D1C2: wget_strcmp (utils.c:82)
==30425==    by 0x40CF61: try_connection (wget.c:1088)
==30425==    by 0x40D1E6: establish_connection (wget.c:1151)
==30425==    by 0x40EA0D: downloader_thread (wget.c:1626)
==30425==    by 0x52BA6B9: start_thread (pthread_create.c:333)
==30425==    by 0x55D73DC: clone (clone.S:109)
==30425==  Address 0xa716033 is 163 bytes inside a block of size 180 free'd
==30425==    at 0x4C2EDEB: free (in
/usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==30425==    by 0x4E5EDD0: wget_iri_free (iri.c:178)
==30425==    by 0x408267: host_remove_job (host.c:347)
==30425==    by 0x40EC4E: downloader_thread (wget.c:1698)
==30425==    by 0x52BA6B9: start_thread (pthread_create.c:333)
==30425==    by 0x55D73DC: clone (clone.S:109)
==30425==  Block was alloc'd at
==30425==    at 0x4C2DB8F: malloc (in
/usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==30425==    by 0x4E6F2C2: wget_malloc (xalloc.c:85)
==30425==    by 0x4E5EE69: wget_iri_parse (iri.c:231)
==30425==    by 0x4E601AC: wget_iri_parse_base (iri.c:623)
==30425==    by 0x407FC6: host_add_robotstxt_job (host.c:288)
==30425==    by 0x40ACE0: add_url_to_queue (wget.c:438)
==30425==    by 0x40C273: main (wget.c:853)
```

I try to modify the test, adding option --no-robots. The result, valgrind check
exited normally. Therefore, I make a conclusion and guest that the robots.txt
processor on Wget2 and my Libmicrohttpd code in libtest.c trigger memory leak.
I have no idea about this. I can just guest that this is related to libgcrypt
error previously said by Christian Grothoff. Although I still found that
libgcrypt error in passed test.

Evgeny Grin gives a review about my works, he direct pointed to the code I
attached for review purpose.

``` c
static char *_scan_directory(const char* data)
{
      char *path = strchr(data, '/');
      if (path != 0) {
              return path;
      }
      else
              return NULL;
}
```

He asks why do I use underscore as prefix for functions. Usually it is used by
libraries to avoid name conflict. First, I think that I need to remove the
underscore, then Tim Ruehsen also give comment that this is also useful in
projects where there is more than one C file. They use it to make clear that a
function/variable is static (not consequently everywhere, though). So, I leaved
the underscore as is.

``` c
static char *_parse_hostname(const char* data)
{
      if (!wget_strncasecmp_ascii(data, "http://", 7)) {
              char *path = strchr(data += 7, '/');
              return path;
      } else
              return NULL;
}

static char *_replace_space_with_plus(char *data)
{
      if (strchr(data, ' ') != 0) {
              char *result = data;
              char *wk, *s;

              wk = s = strdup(data);

              while (*s != 0) {
                      if (*s == ' '){
                              *data++ = '+';
                              ++s;
                      } else
                              *data++ = *s++;
              }
              *data = '\0';
              free(wk);
              return result;
      } else
              return data;
}
```

He asks me about the reason for using `strdup()`/`free()`. I checked string
twice (by `strchr()` and by my custom iterations). It just waste of CPU time.
He gives simpler implementation:

``` c
{
  while(0 != *data)
  {
    if (' ' == *data)
      *data = '+';
    data++;
  }
  return data;
}
```

I applied the changes to my code.  
Evgeny also give note that according to current HTTP specification '+' must NOT
be used as replacement for '&nbsp;' (space) in URLs. When I learn about this problem, it
leads me to here [0]. Then if it need to be applied, I need to modify test file:
test-base.c to not use '+', and use %2B instead.  
Tim added that the '+' was always just for the query part. He asks to Evgeny
what document are you exactly referring to to. Not that he is against dropping
the '+' rule, but what consortium is not accepted as normative by everyone,
while IETF is. He is unsure about what 'spec' to follow. So, I keep my changes
of how I treat the '&nbsp;' (space).

``` c
static int print_out_key(void *cls, enum MHD_ValueKind kind, const char *key,
                                              const char *value)
{
      if (key && url_it == 0 && url_it2 == 0) {
              wget_buffer_strcpy(url_arg, "?");
              _replace_space_with_plus((char *) key);
```

Evgeny said that we are not allowed to modify any content pointed by pointer to
const. By dropping 'const' qualifiers are violating API. In other words: I was
modifying internal structures that are not expected to be modified and the
result is unpredictable.

``` c
              wget_buffer_strcat(url_arg, key);
              if (value) {
                      wget_buffer_strcat(url_arg, "=");
                      _replace_space_with_plus((char *) value);
                      wget_buffer_strcat(url_arg, value);
              }
      }
      if (key && url_it != 0 && url_it2 == 0) {
              wget_buffer_strcat(url_arg, "&");
              _replace_space_with_plus((char *) key);
              wget_buffer_strcat(url_arg, key);
              if (value) {
                      wget_buffer_strcat(url_arg, "=");
                      _replace_space_with_plus((char *) value);
                      wget_buffer_strcat(url_arg, value);
              }
      }
      url_it++;
    return MHD_YES;
}
```

I fixed them then.  
He also gives me some questions:

* Is `url_arg` a global variable?
* Do I use single thread only?
* Why not to pass pointer to `url_arg` as `cls`?
* What about `url_it` and `url_it2`? It is hard to guess meaning from name.

If I need to pass all of them, I was advised to define some structure with three
pointers and pass pointer to structure.

``` c
static int answer_to_connection (void *cls,
                                      struct MHD_Connection *connection,
                                      const char *url,
                                      const char *method,
                                      const char *version,
                                      const char *upload_data, size_t *upload_data_size, void **con_cls)
{
      struct MHD_Response *response;
      int ret;

      url_arg = wget_buffer_alloc(1024);
      MHD_get_connection_values (connection, MHD_GET_ARGUMENT_KIND, print_out_key, NULL);

      url_it2 = url_it;
      wget_buffer_t *url_full = wget_buffer_alloc(1024);
      wget_buffer_strcpy(url_full, url);
      if (url_arg)
              wget_buffer_strcat(url_full, url_arg->data);
      if (!strcmp(url_full->data, "/"))
              wget_buffer_strcat(url_full, "index.html");
      url_it = url_it2 = 0;
      unsigned int itt, found = 0;
      for (itt = 0; itt < nurls; itt++) {
              char *dir = _scan_directory(url_full->data + 1);
              if (dir != 0 && !strcmp(dir, "/"))
                      wget_buffer_strcat(url_full, "index.html");

              char *host = _parse_hostname(url_full->data);
              if (host != 0 && !strcmp(host, "/"))
                      wget_buffer_strcat(url_full, "index.html");

              wget_buffer_t *iri_url = wget_buffer_alloc(1024);
              wget_buffer_strcpy(iri_url, urls[itt].name);
              MHD_http_unescape(iri_url->data);

              if (urls[itt].code != NULL &&
                      !strcmp(urls[itt].code, "302 Redirect") &&
                      !strcmp(url_full->data, iri_url->data))
              {
                      response = MHD_create_response_from_buffer(strlen("302 Redirect"),
                                      (void *) "302 Redirect", MHD_RESPMEM_PERSISTENT);
                      for (int itt2 = 0; urls[itt].headers[itt2] != NULL; itt2++) {
                              const char *header = urls[itt].headers[itt2];
                              if (header) {
                                      char *header_value = strchr(header, ':');
                                      char *header_key = wget_strmemdup(header, header_value - header);
                                      MHD_add_response_header(response, header_key, header_value + 2);
                                      ret = MHD_queue_response(connection, MHD_HTTP_FOUND, response);
                                      wget_xfree(header_key);
                                      itt = nurls;
                                      found = 1;
                              } else
                                      itt = nurls;
                      }
              } else if (!strcmp(url_full->data, iri_url->data)) {
                      response = MHD_create_response_from_buffer(strlen(urls[itt].body),
                                      (void *) urls[itt].body, MHD_RESPMEM_PERSISTENT);
```

He asks me again to ensure that content of `urls[itt].body` will be valid until
end of this connection. Otherwise I must use `MHD_RESPMEM_MUST_COPY` as last
parameter. I followed his advice and fix my code later.

Two generic advises that very useful he gave to me:

* Avoid using global variables.
* Use better names for variables. It it hard to understand what mean
  `iri_url`, `itt` and `itt2`.

### 6th Week

I found some difficulties when deal with global variables. I throw a question
to mentors. Based on explanation from Evgeny before, I was given a suggestion
to avoid using global variables. Then, I modify the code to this:

``` c
// for passing URL query string
struct query_string {
 wget_buffer_t
     *params;
 int
     *it;
};

static int print_out_key(void *cls, enum MHD_ValueKind kind, const char *key,
                        const char *value)
{
    struct query_string *query = cls;

    if (key && query->it == 0) {
        wget_buffer_strcpy(query->params->data, "?");
        replace_space_with_plus(key);
        wget_buffer_strcat(query->params->data, key);
        if (value) {
            wget_buffer_strcat(query->params->data, "=");
            replace_space_with_plus(value);
            wget_buffer_strcat(query->params->data, value);
        }
    }
    if (key && query->it != 0) {
        wget_buffer_strcat(query->params->data, "&");
        replace_space_with_plus(key);
        wget_buffer_strcat(query->params->data, key);
        if (value) {
            wget_buffer_strcat(query->params->data, "=");
            replace_space_with_plus(value);
            wget_buffer_strcat(query->params->data, value);
        }
    }

        query->it++;
    return MHD_YES;
}

static int answer_to_connection(void *cls,
                    struct MHD_Connection *connection,
                    const char *url,
                    const char *method,
                    const char *version,
                    const char *upload_data, size_t *upload_data_size, void **con_cls)
{
        struct query_string *query;

    // get query string
        query->params = wget_buffer_alloc(1024);
        query->it = 0;
    MHD_get_connection_values(connection, MHD_GET_ARGUMENT_KIND, print_out_key, query);

        ...

    wget_buffer_t *url_full = wget_buffer_alloc(1024);
    wget_buffer_strcpy(url_full, url);
    if (query->params->data)
        wget_buffer_strcat(url_full, query->params->data);
    wget_buffer_free(&query->params);

}
```

I need to pass struct query into `MHD_get_connection_values`. But, I never get
expected result, often ends with segmentation faults. Then, I changed variable
initialization to this:

```
struct query_string *query = {wget_buffer_alloc(1024), 0};
```

Still, the result is not what I expected.  
Tim help me with his answers. I asked to switch on warnings (e.g. `touch
.manywarnings` and do `./configure` again). Based on it, I will see something
like:

```
libtest.c: In function 'answer_to_connection':
libtest.c:357:13: warning: 'query' is used uninitialized in this function [-
Wuninitialized]
  query->str = wget_buffer_alloc(1024);
  ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~
```

I should take such warnings for serious analysis and fix them, e.g. by removing
the pointer here:

```
      struct query_str query;
...
      query.str = wget_buffer_alloc(1024);
      query.it1 = 0;
```

I just didn't allocate my previous `*query`. I should use a stack allocation as
above. And that is what all I need. Now, I can pass the variable through the
functions. No more global variable needed.  
Actually I already enable the manywarnings option, nonetheless I can
interpreting the messages. Now, I learned something.

### 7th Week

__Questions about CI testing__

Previously I said that some CI runner still fail such as Gitlab MingW64 and
Travis OSX where other test passed.

__Gitlab MingW64:__ I have no idea about how to install Libmicrohttpd on
MingW64 with Debian images. Previously, I can install Libmicrohttpd on Fedora
images, but currently Wget2 use Debian. I ask if someone can give me a hint
about how to install packages on Debian MingW64.  
Tim said he doubt there is a MinGW64 Libmicrohttpd package for Fedora (nor for
Debian). But, I always could build my own library, e.g. via `git clone`
Libmicrohttpd, cd into the directory, build with MinGW64 (and optionally `make
install`). Then set the right library paths for Wget2's `./configure` run as
always.  
This might be tedious since there could be some pitfalls not easy to debug.  His
suggestion would be to leave this step out until I am ready with all the other
stuff. But when I try, I should do it locally first and note the steps I did.  
On Travis Debian test, when I just use `apt install libmicrohttpd-dev` in
travis script, it installed Libmicrohttpd <= 0.9.51 which causes problems. So,
I do some workaround with install Libmicrohttpd 0.9.55 from source. Here is the
output of git diff from gitlab-ci and travis.

```
diff --git a/.gitlab-ci.yml b/.gitlab-ci.yml
index e8d1946..8e77d8e 100644
--- a/.gitlab-ci.yml
+++ b/.gitlab-ci.yml
@@ -34,6 +34,8 @@ variables:
 #  * ASan, UBSan, Msan
 #  * distcheck
 Debian GNU/Linux build:
+  before_script:
+    - apt-get update -qq && apt-get install -y -qq libmicrohttpd-dev
   image: $CI_REGISTRY/$BUILD_IMAGES_PROJECT:$DEBIAN_BUILD
   script:
     - ./bootstrap && touch .manywarnings
@@ -55,6 +57,8 @@ Debian GNU/Linux build:
       - tests/*.log

 clang/Fedora:
+  before_script:
+    - dnf -y install libmicrohttpd-devel
   image: $CI_REGISTRY/$BUILD_IMAGES_PROJECT:$FEDORA_BUILD
   script:
     - export CC=clang
diff --git a/.travis_setup.sh b/.travis_setup.sh
index 916dbdb..6e5041b 100755
--- a/.travis_setup.sh
+++ b/.travis_setup.sh
@@ -14,7 +14,13 @@ if [[ "$TRAVIS_OS_NAME" = "osx" ]]; then
      brew install xz
      brew install lbzip2
      brew install lzip
+     brew install libgcrypt
+     brew install libmicrohttpd
      brew link --force gettext
 elif [[ "$TRAVIS_OS_NAME" = "linux" ]]; then
+     sudo apt-get -y install wget
+     wget http://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.55.tar.gz
+     tar zxf libmicrohttpd-0.9.55.tar.gz && cd libmicrohttpd-0.9.55/
+     ./configure --prefix=/usr && make -j$(nproc) && sudo make install
      pip install --user cpp-coveralls
 fi
```

I then ask that my snippet above is appropriate or there is any other better
solutions, and Tim said it is fine to use it.

__Travis OSX:__ Failure regarding test with HTTPS support. I have checked
that even I provide all dependencies for TLS/SSL support like gnutls and
libgcrypt, it still fail. I don't know if it fails because how my code read
certificates provided or other reasons.  
Tim comments is the problem occurs only on the OSX VM. This looks like
Libmicrohttpd is not starting correctly. The same versions (gnutls 3.5.14 and
Libmicrohttpd 0.9.55) work fine here on Debian unstable. I should try to get
more logging output.  
Evgeny also give comments, more output from Libmicrohttpd could give more
information.  What he noticed in the meantime is warning `passing const char *
to parameter of type char * discards qualifiers` which means that could
modifying read-only memory. I should try to resolve it not by casting to `char
*`.  
Then I have a question, how could I get more verbose messages log of
Libmicrohttpd? I tried to use `MHD_OPTION_EXTERNAL_LOGGER` like in example
provided in `msgs_i18n.c`.

``` c
httpsdaemon = MHD_start_daemon(MHD_USE_SELECT_INTERNALLY | MHD_FEATURE_MESSAGES |
                               MHD_USE_TLS | MHD_USE_ERROR_LOG,
               port_num, NULL, NULL, &_answer_to_connection, NULL,
               MHD_OPTION_EXTERNAL_LOGGER, &error_handler, NULL,
               MHD_OPTION_HTTPS_MEM_KEY, key_pem,
```

But, it doesn't generated any useful messages. It Just tell me that the HTTPS
server failed in the runtime.  
Evgeny replied, most probably I was trying to use Libmicrohttpd build without
HTTPS support. I should check original package. Alternatively, I could try
`MHD_is_feature_supported(MHD_FEATURE_SSL)` and skip HTTPS tests if HTTPS is not
supported by Libmicrohttpd. I can also display warning about it during
configure.  Before implementing such check, find first Libmicrohttpd version
where `MHD_is_feature_supported()` was added.  
I asked about which kind of problems I have with Libmicrohttpd 0.9.51. Evgeny
said most recent versions of Libmicrohttpd are stable, there are only a few
"problematic" versions. I replied with show the build log:

```
The following NEW packages will be installed:
  libmicrohttpd-dev libmicrohttpd10
0 upgraded, 2 newly installed, 0 to remove and 25 not upgraded.
Need to get 190 kB of archives.
After this operation, 499 kB of additional disk space will be used.

0% [Working]

Get:1 http://us-central1.gce.archive.ubuntu.com/ubuntu trusty/universe amd64 libmicrohttpd10
+amd64 0.9.33-1 [41.0 kB]

======

libtest.c: In function '_http_server_start':
libtest.c:547:3: error: unknown type name 'MHD_socket'
   MHD_socket sock_fd;
   ^
```

It seems the version of Libmicrohttpd installed is 0.9.33-1. For quick
workaround I installed v0.9.55 from source and it works. He also said, indeed
v0.9.33 is too old. But it's not too hard to handle it:

``` c
#include <microhttpd.h>

#if MHD_VERSION <= 0x93302
#define MHD_socket int
#endif

```

Nice pointer. Maybe I can apply this solution later.

__Questions about many warnings on libtest.c and Libmicrohttpd__

Here some warning I get when I build `libtest.c` on my system. I mentioned my
system that I use Ubuntu 16.04 64 bit with configure command `./configure
--enable-manywarnings`

```
libtest.c: In function ‘_load_file’:
libtest.c:166:11: warning: comparison between signed and unsigned integer expressions [-Wsign-compare]
  if (size != fread(buffer, 1, size, fp)) {
          ^
libtest.c:227:29: warning: passing argument 1 of ‘_replace_space_with_plus’ discards ‘const’ qualifier from pointer target type [-Wdiscarded-qualifiers]
    _replace_space_with_plus(value);

libtest.c:194:14: note: expected ‘char *’ but argument is of type ‘const char *’
 static char *_replace_space_with_plus(char *data)
              ^
```

Once I tried to use `char *` on `_replace_space_with_plus()` so it becomes
`_replace_space_with_plus((char *) key)` but it not allowed (API violation).
Tim suggests me to give `query-params` to my "replace function" and iterate through
all chars and add these to the buffer. For example:

```
_replace_space_with_plus(query->params, key);

static void _replace_space_with_plus(wget_buffer_t *buf, const char *data)
{
        for (; *data; data++)
                wget_buffer_memcat(buf, *data == ' ' ? "+" : data, 1);
}
```

He said no need to do any optimization at this point.  
However, Evgeny tells that using implicit conversion from `const char*` to
`char*` is the same API violation. But there is a better solution. If I not
allowed to modify original string (as this this is used for other proposes as
well) and I can read original string, just make my own copy of string and
modify it. And I should free my copy when I don't need it anymore.  
Here is another warning messages:

```
libtest.c: In function ‘_print_authorization’:
libtest.c:252:63: warning: unused parameter ‘kind’ [-Wunused-parameter]
 static int _print_authorization(void *cls, enum MHD_ValueKind kind,
                                                               ^
```

It used by Libmicrohttpd API but it treated as warning by the compiler.

```
libtest.c:272:64: warning: unused parameter ‘con_cls’ [-Wunused-parameter]
      const char *upload_data, size_t *upload_data_size, void **con_cls)
```

Same as above, it treated as warning by the compiler, but actually it required.  
Solution from Tim, if I found error regarding `-Wunused-parameter` is by adding
the attribute `G_GNUC_WGET_UNUSED` to the param.  
Another warning messages:

```
libtest.c:429:13: warning: Value MHD_HTTP_REQUESTED_RANGE_NOT_SATISFIABLE is deprecated, use MHD_HTTP_RANGE_NOT_SATISFIABLE
      ret = MHD_queue_response(connection, MHD_HTTP_REQUESTED_RANGE_NOT_SATISFIABLE, response);
             ^
```

It seems about Libmicrohttpd incompatibility. I will provide version checker
(ifdef MHD version) to fix this.  
Tim directs me to the solution:

``` c
#ifdef MHD_HTTP_RANGE_NOT_SATISFIABLE
  ret = MHD_queue_response(connection, MHD_HTTP_RANGE_NOT_SATISFIABLE,
response);
#else
  ret = MHD_queue_response(connection,
MHD_HTTP_REQUESTED_RANGE_NOT_SATISFIABLE, response);
#endif
```
Very clear explanation from him.  
Some warnings regarding unmatching data type:

```
libtest.c: In function ‘_answer_to_connection’:
libtest.c:365:27: warning: comparison between signed and unsigned integer expressions [-Wsign-compare]
     for (int it2 = 0; it2 < countof(urls[it1].headers); it2++) {
                           ^
```

Tim said it just as simple as add `unsigned` keyword in front of data type,
like below:

```
for (unsigned it2 = 0; it2 < countof(urls[it1].headers); it2++) {
```

Tim said, in general, I should stick with the Wget2 coding style for
consistency. For example, instead of `if (' ' == *data)` use `(*data == ' ')`.
He comments about function I write: `_get_file_size()`. He suggests to use
`stat()`. It's just single system call. Other function, `_load_file()`. He
recommends to use wget built in function, `wget_read_file()` as replacement.
Actually I get those functions from Libmicrohttpd implementation example of
HTTPS. Tim suggestions made all of this shorter and simpler.

__Problem with HTTP Persistent Connection__

One of my report said, I need to solve problem of `check-valgrind`. I figure out
that the problem come out from connection state of the response which need to
be "closed" after created. The solution is just to add HTTP header "Connection:
Close" in all response without exception.  
Evgeny gives me some clarifications about this. This way I am actually hiding
the problem, not fixing. Both Libmicrohttpd and wget must work perfectly nice
with keep-alive. Moreover, in Libmicrohttpd test suite we run many tests in
two modes: with "keep-alive" and "close" connections to ensure testing of all
sides. He suggest me to fix modification of read-only memory before trying to
resolve valgrind as this modification could easily trigger valgrind errors. If
I run any test in multi-thread mode, he suggests to check thread
synchronisation and concurrent access to modifiable variable.  
After learned that I should leave persistent connection to be exist between
client and server, I try to make a change, just by add HTTP header "Connection:
Close" on request that returned 404, 401 and 416 status code, just like in the
old http server code. Even more, I actually don't have to close connection
after those response codes. Otherwise, I leave the connection as is
(keep-alive). But, I still found other issues from the result.  
The test case fail if they found robots.txt in server, in other words,
robots.txt returned status 200 OK. Here the error log from running the test
with address sanitizer enabled:

```
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 85
Content-Type: text/plain
Date: Thu, 20 Jul 2017 19:17:43 GMT

21.021743.773 method 2
21.021743.773 nbytes 85 total 85/85
21.021743.773 keep_alive=1
21.021743.773 Scanning robots.txt ...
21.021743.773 host_remove_job: 0x60c00000b980
21.021743.773 host_remove_job: qsize=1 host->qsize=1
21.021743.774 [0] action=1 pending=0 host=0x60700000df40
21.021743.774 qsize=1 blocked=0
21.021743.774 pause=-1500578263774
21.021743.774 dequeue job http://localhost:41929/index.html
21.021743.774 main: wake up
=================================================================
==14544==ERROR: AddressSanitizer: heap-use-after-free on address 0x610000007de3 at pc
+0x7f16594162d5 bp 0x7f16504fec60 sp 0x7f16504fe408
READ of size 6 at 0x610000007de3 thread T1
    #0 0x7f16594162d4  (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x472d4)
    #1 0x7f1659070c91 in wget_strcmp /home/didik/wget2/libwget/utils.c:82
    #2 0x41b6d4 in try_connection /home/didik/wget2/src/wget.c:1088
    #3 0x41c081 in establish_connection /home/didik/wget2/src/wget.c:1151
    #4 0x422f06 in downloader_thread /home/didik/wget2/src/wget.c:1643
    #5 0x7f1657e176b9 in start_thread (/lib/x86_64-linux-gnu/libpthread.so.0+0x76b9)
    #6 0x7f1657b4d3dc in clone (/lib/x86_64-linux-gnu/libc.so.6+0x1073dc)

0x610000007de3 is located 163 bytes inside of 180-byte region [0x610000007d40,0x610000007df4)
freed by thread T1 here:
    #0 0x7f16594672ca in __interceptor_free (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x982ca)
    #1 0x7f165904840b in wget_iri_free /home/didik/wget2/libwget/iri.c:287
    #2 0x40d5b0 in host_remove_job /home/didik/wget2/src/host.c:347
    #3 0x423800 in downloader_thread /home/didik/wget2/src/wget.c:1715
    #4 0x7f1657e176b9 in start_thread (/lib/x86_64-linux-gnu/libpthread.so.0+0x76b9)

previously allocated by thread T0 here:
    #0 0x7f1659467602 in malloc (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x98602)
    #1 0x7f16590784da in wget_malloc /home/didik/wget2/libwget/xalloc.c:85
    #2 0x7f1659048648 in wget_iri_parse /home/didik/wget2/libwget/iri.c:350
    #3 0x7f165904da08 in wget_iri_parse_base /home/didik/wget2/libwget/iri.c:818
    #4 0x40c9cf in host_add_robotstxt_job /home/didik/wget2/src/host.c:288
    #5 0x414bba in add_url_to_queue /home/didik/wget2/src/wget.c:438
    #6 0x4192ae in main /home/didik/wget2/src/wget.c:853
    #7 0x7f1657a6682f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

Thread T1 created by T0 here:
    #0 0x7f1659405253 in pthread_create (/usr/lib/x86_64-linux-gnu/libasan.so.2+0x36253)
    #1 0x7f165906d933 in wget_thread_start /home/didik/wget2/libwget/thread.c:48
    #2 0x41a3bf in main /home/didik/wget2/src/wget.c:963
    #3 0x7f1657a6682f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)

SUMMARY: AddressSanitizer: heap-use-after-free ??:0 ??
Shadow bytes around the buggy address:
  0x0c207fff8f60: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c207fff8f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c207fff8f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c207fff8f90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c207fff8fa0: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
=>0x0c207fff8fb0: fd fd fd fd fd fd fd fd fd fd fd fd[fd]fd fd fa
  0x0c207fff8fc0: fa fa fa fa fa fa fa fa 00 00 00 00 00 00 00 00
  0x0c207fff8fd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 04 fa
  0x0c207fff8fe0: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c207fff8ff0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c207fff9000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Heap right redzone:      fb
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack partial redzone:   f4
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
==14544==ABORTING
Unexpected error code 1, expected 0 [-r -nH]
Removed test directory '.test_14531'
FAIL test-robots (exit status: 1)
```

What I can see is the process want to use an allocated memory that has been
freed before. But, I still don't know what exact variable/pointer that trigger
the problem.  
Evgeny give comments about this. It should be pretty straightforward. I should
check what is allocated at `host_add_robotstxt_job()->wget_iri_parse`
`iri.c:350` in main thread. Find out how it's passed to `downloader_thread()`.
Find out how it's freed by `host_remove_job()->wget_iri_free()`. Find out why
it's still used in
`downloader_thread()->establish_connection()->try_connection()` at
`wget.c:1088`.  
Tim give me a clue, this is definitely a bug in Wget2 itself. No matter what
the test server is doing, this should not happen. He will try to reproduce this
issue to investigate later.  
Because I feel a little bit rush, I have create a merge request regarding this
[1]. I thought I was doing the right thing, until I realize, I made a mistake.
Instead of posting issue, I made a merge request, which actually I try to
modify code I quite not understand. Hence, I get some cautions about this. But,
because of this, now I know the right procedure how to handle problem in
community. Mentioning an issue itself is good way to contribute to community,
because valid issue will lead our community to find the solution early.

__Questions about chunked tranfer encoding__

In test case that involving chunked transfer encoding, I found that they just
freeze without another reaction, holding up until I send interrupt signal.

```
HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Length: 15
Transfer-Encoding: chunked
Content-Type: text/plain
Date: Thu, 20 Jul 2017 19:30:55 GMT

21.023055.800 method 1 0 0:
21.023055.800 a nbytes 15 body_len 15
21.023055.800 chunk size is 3
21.023055.800 write full chunk, 3 bytes
21.023055.801 chunk size is 2
21.023055.801 write full chunk, 2 bytes
```

I don't know why in the old http server code, they just can close the connection
implicitly after generate the response.  
Evgeny replied I should check with debugger first step-by-step chunked download
process.  
I was looking in to the manual and examples that we should use
`MHD_create_response_from_callback()` to handle chunked transfer encoding
connection.  My question, how do we generate a data using
`MHD_ContentReaderCallback`? In the examples there just an empty data
(`chunked_example.c`) and infinite data (`minimal_example_comet.c`)? Say we
have a string as the buffer, something like:

```
static ssize_t
data_generator (void *cls, uint64_t pos, char *buf, size_t max)
{
  if (max < 80)
    return 0;

  strcpy(buf, "3\r\nan\r\n2\r\example\r\n");

  return 80;
}
```

I also asks, how do I arrange it properly? So the result is like expected
chunked transfer encoding file. I have tried using that example and the
result is infinite streaming file. I try using return value
`MHD_CONTENT_READER_END_OF_STREAM` and it give me nothing, just debug
information showing chunked transfer encoding.  
Evgeny said, it was indeed not clear from example how to use chunked transfer.
He give a hand by updated chunked_example.c in official git repository.  I'd
tried `chunked_example.c` again and it was successfully run. But, when I modify
using encoded data like:

```
static const char response_data[] = "3\r\nan\r\n2\r\example\r\n";
```

The result was:

```
3
an
2
example
```

I was expected a result something like plain: "an example" as the output.  
Evgeny explained, I don't need to handle chunked encoding in application.
Libmicrohttpd automatically wrap supplied data into chunks with local header and
footer.  If I want to generate more chunks - simply provide smaller amount of
data with each call of callback. Each part of data will be send as separate
chunk.  
I'm actually stuck in this. I asked how do we present a connection with multiple
chunks combined into one complete data? Like what he said, I want to generate
more chunks. I ask if the code below related:

```
/* Pseudo code.        *
if (data_not_ready)
  {
    // Callback will be called again on next loop.
    // Consider suspending connection until data will be ready.
    return 0;
  }
* End of pseudo code. */
```

I wonder what should I do with "data_not_ready"? My goal is to simulate chunked
transfer encoding between client and server using Wget2 and Libmicrohttpd
respectively. Currently, in the Wget2 test suite, it uses encoded data like:

```
"3\r\nabc\r\n2\r\nde\r\n"
```

With Libmicrohttpd, I don't need to provide encoded data like above. It
automatically handled by Libmicrohttpd. So I need to change data provided in
`test-chunked.c`.  
Evgeny said, I don't need to change data provided by callback.
I need to change amount of data provided by each call of callback. For example:

``` c
if (buf_size < (param->response_size - pos))
  size_to_copy = buf_size;
else
  size_to_copy = param->response_size - pos;

if (2 < size_to_copy)
  size_to_copy = 2;

memcpy (buf, param->response_data + pos, size_to_copy);
```

Or use more complex algorithm with variable size depending on position.
In this case chunks may have different size.  
It's now clear. I just need to divide data into two smaller chunks.

Another case is, there is a garbage data like `FFFFFFF4\nGarbage` need to
produced. I need to simulate it with Libmicrohttpd.  
Evgeny answered, depending on where I want to put this data. If I want to put it
in reply body - just use `FFFFFFF4\nGarbage` as reply.  
But, I still don't get it. Where is the reply body?  
He said reply body is in response data sent by Libmicrohttpd.  
If I wrote it in response text, it just literally send me same text as before
processed. Instead, I want to create response with chunk size `FFFFFFF4`, like I
wrote in the provided response text (before data encoded).  
He then said, if I want to have chunk size `FFFFFFF4`, I have to provide data
with size `FFFFFFF4`. Libmicrohttpd do not generate invalid HTTP data. For any
test with invalid reply he suggests me to use custom simple hard coded
pseudo-server instead of Libmicrohttpd.

Another question is how can I passed a variable through callback.

```
wget_buffer_t *chunked_data = wget_buffer_alloc(1024);
wget_buffer_strcpy(chunked_data, "some char");
response = MHD_create_response_from_callback(MHD_SIZE_UNKNOWN,
                                            1024,
                                            &_callback,
                                            chunked_data,
                                            NULL);
```

My goal is to pass "chunked_data" as response_data for the callback.
Evgeny updated chunked example once more.
From his point of view, it is pretty simple and clear.

Another question, and maybe just it is my misunderstanding, about how the
response text provided. It requires to use char array, while I just have char
pointer.

```
char *body = "abcde";
static char simple_response_text[6];
strcpy(simple_response_text, body);
simple_response_text[sizeof(simple_response_text) - 1] = '\0';
```

Then I copy the char pointer into char array using `strcpy`. Though I still need
to provide the length of the char. As a workaround, I just hard coded the char
length. I ask if there is any better solution.  
Evgeny give some explanations, char size is always one byte (it could have
different size, but almost all real platforms use one byte). I should not
assign pointer to static string (which is `const char *`) to non-const pointer.
`strcpy() ` copies termination null as well, no need to terminate it again.
Libmicrohttpd response do not need null-terminated strings, as HTTP could send
any data, not only strings. I can use array variable as pointer to first array
member.  
Hence I need to read more about C basics, including static strings, arrays,
pointers, pointers manipulations, pointers arithmetics, strings manipulations
and memory management.

### 8th Week

I have work on `wget_test_start_server()` to use Libmicrohttpd as HTTP server.
To resolve this, I fix `answer_to_connection()` function to create proper HTTP
response to:

* Handle arguments (query string) on URL
* Handle redirection
* Handle directory creation
* Handle URL with IRI object
* Handle URL with IDN hostname
* Handle URL with query string which contains space
* Handle If-Modified-Since header
* Handle Byte Range request
* Handle HTTP basic authentication
* Handle HTTPS

Pass CI test on some system like Debian/GCC and Fedora/Clang.
Fix compiler warning `-Wunused-param` caused by Libmicrohttpd API.
Use persistent connection (Connection: keep-alive) instead multiple.
connection (Connection: close). This prevent hiding several problems of the
connection between HTTP server and client.  
At this point I think that the new HTTP server with Libmicrohttpd for test suite
has already covered all existing test. Later, I can add non existing test that
ready on Wget2 and Libmicrohttpd. Such as add HTTP digest authentication test.
Another unresolved tasks are:

* Fix connection handler using chunked tranfer encoding
* Fix CI testing on Gitlab MingW64 and Travis OSX

### Second Period Evaluations

Just like previous time, we as student are required to submit evaluations. The
evaluations itself was just like the one before, it asks us to fill feedback
form to the organization.  
The results then announced. Mentors give me chance to pass this period
evaluations. They send me messages that my communication has definitely
improved over time. They think this project is now again moving on the right
track towards completion.

### Conclusion

In this period I faced again with many challenges that test my technical
skills. My communication abilities are also tested a lot. I must be good in
conveying problems to my mentor so there is no miscommunication. Mentors also
helped me a lot in solving the problems I have encountered. They give me a trust
to finish this project. I should be able to take advantage of this chance to
give the best of me in contributing to this project.

Reference(s):  
[0] [https://stackoverflow.com/questions/1005676/urls-and-plus-signs](https://stackoverflow.com/questions/1005676/urls-and-plus-signs)  
[1] [https://gitlab.com/gnuwget/wget2/merge_requests/250](https://gitlab.com/gnuwget/wget2/merge_requests/250)
