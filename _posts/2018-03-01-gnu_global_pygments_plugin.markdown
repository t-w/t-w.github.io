---
layout: post
title: "Support for more languages in GNU Global with Pygments plugin"
date: 2018-03-01
categories: gnu_global
tags: gnu global pygments python
---

### GNU global extension to support other languages
What I found missing in the [GNU global][emacs.ggtags] was support for more
programming languages, natively very few are supported (maily C, C++).
I have just found [a plugin (global pygments plugin)][global_pygments_plugin]
which is doing very smart job using [Pygments][pygments] (a syntax highlighter
for Python) to extract tags. This means that any language supported by Pygments
(there is a lot) can be used also with GNU global (and with GGtags - in Emacs!).

Installation is pretty straightforward, just follow the procedure
(I have additionally built a `.deb` package with checkinstall), create
`.globalrc` (eg. sample provided is working well) - and that's it.

For the moment I use this with Python - works great.

[emacs.ggtags]: https://t-w.github.io/emacs/2016/11/06/emacs_gnu_global_ggtags/
[global_pygments_plugin]: https://github.com/yoshizow/global-pygments-plugin
[pygments]:     http://pygments.org/
