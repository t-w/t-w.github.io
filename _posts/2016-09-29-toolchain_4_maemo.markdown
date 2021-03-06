---
layout: post
title: "Toolchains for Maemo"
date: 2016-09-29
categories: cross-compilation toolchain arm n900 maemo
tags: cross-compilation toolchain arm n900 maemo
---
This time about some legacy stuff for a nice device I still own (Nokia N900).

Decided to play with porting some software to it and here's where the fun began...
Trying to create a reasonable toolchain I ran on several possibilities, not sure
which is the best one (can do what I need, is reasonable to use ;)

### Available toolchains
Apparently there are several, working in slightly (or significantly) different way
(the same with features they offer). So what's available:

1. [Scratchbox][sb.1]
2. [Scratchbox2][sb2.1] [git repo archive][sb2.2]
on [gitorious archive][gitorious] which contains lots of N900/Maemo related projects)
3. [Maemo SDK+][sdk+.1] (based on Scratchbox2)
4. [Sourcery CodeBench][codebench.1]
5. [Maemo5 "final" SDK][maemo.org.sdk]

### Related tutorials / documents / presentations
- [The Scratchbox development environment][sb.doc.1]
- [Scratchbox FAQ][sb.faq]

### Other links
- [maemo.org/development/][maemo.org.dev]


[sb.1]:          http://scratchbox.org/
[sb.faq]:        http://scratchbox.org/faq/
[sb.doc.1]:      http://free-electrons.com/docs/scratchbox/
[sb2.1]:         https://www.freedesktop.org/wiki/Software/sbox2/
[sb2.2]:         https://gitorious.org/scratchbox2/scratchbox2.git
[gitorious]:     https://gitorious.org/
[sdk+.1]:        http://maemo-sdk.garage.maemo.org/
[codebench.1]:   https://www.mentor.com/embedded-software/sourcery-tools/sourcery-codebench/editions/lite-edition/
[maemo.org.dev]: http://maemo.org/development/
[maemo.org.sdk]: http://wiki.maemo.org/Documentation/Maemo_5_Final_SDK_Installation
