---
layout: post
title: "Debugging inside common shared libraries with GNU debugger (gdb)"
date: 2016-09-08
categories: debugging shared libraries gdb
tags: debugging shared libraries gdb
---
There seems to be no post with reasonably condensed information
on this topic (in a solution/howto format, not
complete-and-hard-to-find-details-you-need documentation),
so I thought gathering these misc. pieces might be useful.

To debug execution of your program _inside_ a shared library
we need a bit more than just compiling it with `-g`.
Let's see a simple case - a program using a call to a function
from `SDL2_mixer`, as an example.

Initially requesting step-into results in step-over:

{% highlight c %}
78              audio_mix_chunks[i] = Mix_LoadWAV(fname);
(gdb) s
80              if (audio_mix_chunks[i] == NULL)
{% endhighlight %}

and, well, that's not what we want...

Two things are probably missing:
debug information in the binary (library in this case), and the source code for it.

### Debug information for the library

In case of Debian - you just have to install `libXYZ-dbg` package (where XYZ is the library you want to examine). So:

{% highlight bash %}
$ aptitude install libsdl2-mixer-dbg
{% endhighlight %}

and we have:

{% highlight c %}
(gdb) s
Mix_LoadWAV_RW (src=0x7fffffffe270, freesrc=4203968) at mixer.c:573
573     mixer.c: No such file or directory.
(gdb) s
581     in mixer.c
(gdb) s
587     in mixer.c
(gdb) s
596     in mixer.c
{% endhighlight %}

Now, we can see the line number and the name of the source file - but gdb
does not show any code... In this case the reason is that this external code
is not a part of our code, but is one of the sources of a dynamically linked
library (here `SDL2_mixer`).

This leads us to the 2nd missing thing:

### The source code of the library

We know more less how to get it ;-), eg.:

{% highlight bash %}
$ apt-get source libsdl2-mixer-dev
{% endhighlight %}

what results in my case in sources in `libsdl2-mixer-2.0.0+dfsg1/`.

But where should we put them so that `gdb` finds them?

{% highlight c %}
(gdb) s
Mix_LoadWAV_RW (src=0x7fffffffe270, freesrc=4203968) at mixer.c:573
573     mixer.c: No such file or directory.
(gdb) info frame
Stack level 0, frame at 0x7fffffffe140:
 rip = 0x7ffff6b04be0 in Mix_LoadWAV_RW (mixer.c:573); saved rip = 0x4092b6
 called by frame at 0x7fffffffe280
 source language c.
 Arglist at 0x7fffffffe130, args: src=0x7fffffffe270, freesrc=4203968
 Locals at 0x7fffffffe130, Previous frame's sp is 0x7fffffffe140
 Saved registers:
  rip at 0x7fffffffe138
{% endhighlight %}

-- and still no clue...

Since `libSDL2_mixer` is clearly dependent on `libSDL2` we can try:

{% highlight bash %}
$ apt-get install libsdl2-dbg
{% endhighlight %}

-- and now(!):
{% highlight c %}
78              audio_mix_chunks[i] = Mix_LoadWAV(fname);
(gdb) s
SDL_RWFromFile (a=0x7fffffffe160 "data/snd/S00.wav", b=0x43b7bf "rb")
    at /tmp/buildd/libsdl2-2.0.2+dfsg1/src/dynapi/SDL_dynapi_procs.h:386
386     /tmp/buildd/libsdl2-2.0.2+dfsg1/src/dynapi/SDL_dynapi_procs.h: No such file or directory.

(gdb) info frame
Stack level 0, frame at 0x7fffffffe140:
 rip = 0x7ffff6d925b0 in SDL_RWFromFile
    (/tmp/buildd/libsdl2-2.0.2+dfsg1/src/dynapi/SDL_dynapi_procs.h:386); saved rip = 0x4092a9
 called by frame at 0x7fffffffe280
 source language c.
 Arglist at 0x7fffffffe130, args: a=0x7fffffffe160 "data/snd/S00.wav", b=0x43b7bf "rb"
 Locals at 0x7fffffffe130, Previous frame's sp is 0x7fffffffe140
 Saved registers:
  rip at 0x7fffffffe138
{% endhighlight %}

Finally gdb is informing us nicely where it expects the source code of the libraries - so we can provide it:

{% highlight bash %}
$ mkdir /tmp/buildd ; cd /tmp/buildd/
$ apt-get source libsdl2-mixer-dev
$ apt-get source libsdl2-dev
{% endhighlight %}

This time we should see what's inside the library, for instance:

{% highlight c %}
78              audio_mix_chunks[i] = Mix_LoadWAV(fname);
(gdb) s
SDL_RWFromFile (a=0x7fffffffe160 "data/snd/S00.wav", b=0x43b7bf "rb")
    at /tmp/buildd/libsdl2-2.0.2+dfsg1/src/dynapi/SDL_dynapi_procs.h:386
386     SDL_DYNAPI_PROC(SDL_RWops*,SDL_RWFromFile,(const char *a, const char *b),(a,b),return)
(gdb) s
SDL_RWFromFile_REAL (file=0x7fffffffe160 "data/snd/S00.wav", mode=0x43b7bf "rb")
    at /tmp/buildd/libsdl2-2.0.2+dfsg1/src/file/SDL_rwops.c:461
461         if (!file || !*file || !mode || !*mode) {
(gdb) s
459     {
(gdb) s
461         if (!file || !*file || !mode || !*mode) {
(gdb) s
526             FILE *fp = fopen(file, mode);
(gdb) n
528             if (fp == NULL) {
{% endhighlight %}

There is also another (better!) way to find out the source files
and their paths, as expected by `gdb`:

{% highlight c %}
(gdb) info sources

(a looong list...)

/tmp/buildd/libsdl2-mixer-2.0.0+dfsg1/music.c, /tmp/buildd/libsdl2-mixer-2.0.0+dfsg1/mixer.c,

(another looong list...)
{% endhighlight %}

-- although it is necessary to go through a looong list of files.

Information on [another blog][4.] suggests that it is possible also to specify gdb the directory with sources doing eg:

{% highlight c %}
(gdb) dir /var/tmp/sources/libc6/eglibc-2.15/malloc/
{% endhighlight %}

even saving in a file:

{% highlight c %}
$ echo dir /var/tmp/sources/libc6/eglibc-2.15/malloc/ > gdb.setup
$ gdb program -c core -x gdb.setup
{% endhighlight %}

but (if I understand it well...) it specifies a _single directory_, not complete _directory tree_ with all sources of a library that can be used easily by gdb.


Useful related links
--------------------
1. [Debian maintainer's guide][1.]
2. [Debian Policy Manual][2.]
3. [Few GDB Commands – Debug Core, Disassemble, Load Shared Library (Blog article)][3.]
4. [Make system library source code available to gdb on Ubuntu][4.]
5. Related GDB docs:
    - [Commands to specify files][5.]
    - [Debugging Information in Separate Files][6.]


[1.]: https://www.debian.org/doc/manuals/maint-guide/advanced.en.html
[2.]: https://www.debian.org/doc/debian-policy/ch-sharedlibs.html
[3.]: http://www.thegeekstuff.com/2014/03/few-gdb-commands/
[4.]: http://trail-of-a-programmer.blogspot.com/2014/11/make-system-library-source-code.html
[5.]: https://sourceware.org/gdb/onlinedocs/gdb/Files.html
[6.]: https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html

Misc.
-----
*  useful `gdb` commands
    *  `(gdb) info sharedlibrary`
    *  `(gdb) info sources`
    *  `(gdb) info frame`
