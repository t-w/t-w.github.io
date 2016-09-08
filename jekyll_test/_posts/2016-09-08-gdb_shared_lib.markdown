---
layout: post
title: "Debugging inside shared libraries with GNU debugger (gdb)"
date: 2016-09-08
categories: software development debugging shared libraries gdb
---
Since there seems to be no post with a reasonably condensed info on this topic
(in kind of solution based format, not complete-and-hard-to-find-detail-you-need documentation)
I thought gathering these misc. pieces might be useful for others.

Debugging execution of your program INSIDE a shared library you need a bit more than just compiling it with `-g`. Let's see a simple case - a program using a call to a function from `SDL2_mixer`, as an example.

Initially requesting step-into results in step-over:

{% highlight c %}
78              audio_mix_chunks[i] = Mix_LoadWAV(fname);
(gdb) s
80              if (audio_mix_chunks[i] == NULL)
{% endhighlight %}


Well, that's not what we want...

Author's newbie experience leads to conclusion that two things are probably missing:
debug information in the binary (library in this case), and the source code for it.

1. Debug information for the library
In case of Debian - you just have to install `libXYZ-dbg` package (where XYZ is the library you want to examine). So:
{% highlight c %}
$ aptitude install libsdl2-mixer-dbg
{% endhighlight %}

and we have now:

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

We have arrived to another code module, part of dynamically linked `libSDL2_mixer`.

Now, we can the the line number and the name of the source file, but gdb does not show any code...

This leads us to the 2nd missing thing:

2. The source code of the library

We know more less how to get it ;-), eg.:

{% highlight c %}
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
- and still no clue...

Since `libSDL2_mixer` is clearly dependent on `libSDL2` I decide to try:

{% highlight c %}
$ apt-get install libsdl2-dbg
{% endhighlight %}

- and now(!):
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

{% highlight c %}
$ mkdir /tmp/buildd ; cd /tmp/buildd/
$ apt-get source libsdl2-mixer-dev
$ apt-get source libsdl2-dev
{% endhighlight %}

This time we should see what's inside the libraru, for example:

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

Later I learned that there is another (better!) way to find out the source files
and their paths that `gdb` expects:

{% highlight c %}
(gdb) info sources

(a looong list...)

/tmp/buildd/libsdl2-mixer-2.0.0+dfsg1/music.c, /tmp/buildd/libsdl2-mixer-2.0.0+dfsg1/mixer.c,

(another looong list...)
{% endhighlight %}

Information on the blog 4. suggests that it is possible also to specify gdb the directory with sources doing eg:

{% highlight c %}
(gdb) dir /var/tmp/sources/libc6/eglibc-2.15/malloc/
{% endhighlight %}

even saving in a file:

{% highlight c %}
$ echo dir /var/tmp/sources/libc6/eglibc-2.15/malloc/ > gdb.setup
$ gdb program -c core -x gdb.setup
{% endhighlight %}

but (if I understand it well...) it specifies a _single_ directory, not complete directory tree with all sources of a library that can be used easily by gdb.


Useful related links:
1. [Debian maintainer's guide][1.]
2. [Debian Policy Manual][2.]
3. [Few GDB Commands – Debug Core, Disassemble, Load Shared Library (Blog article)][3.]
4. [Make system library source code available to gdb on Ubuntu][4.]

[1.]: https://www.debian.org/doc/manuals/maint-guide/advanced.en.html
[2.]: https://www.debian.org/doc/debian-policy/ch-sharedlibs.html
[3.]: http://www.thegeekstuff.com/2014/03/few-gdb-commands/
[4.]: http://trail-of-a-programmer.blogspot.com/2014/11/make-system-library-source-code.html


https://sourceware.org/gdb/onlinedocs/gdb/Files.html
https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html


Misc.
- (gdb) info sharedlibrary
