---
layout: post
title: "GNU Global in Emacs -> ggtags"
date: 2016-11-06
categories: emacs
tags: emacs gnu global ctags etags ggtags tags
---

For quite a while I was looking for reasonable way to use Emacs as a real IDE,
in particular I needed something to work with large and complex codebase
what requires tools to efficiently traverse and find things wherever they are.
I have tried ctags/etags, and while it is useful to some extent giving quick
access to definitions, still [it does not provide way to find all occurences
of a variable or a function][ctags]. So I was stuck with grep and friends...

Today accidentally I have found [this post][cscope.stackex] with one but great answer,
which in several lines provided me solution I was looking for. Basically instead of ctags/etags
it is better to use GNU Global with ggtags (or helm-gtags).
I just had to do few simple steps:

- install gnu global (in Debian `$ apt-get install global`)
- install [MELPA][melpa] following [the instruction][melpa.install]
- having MELPA in my Emacs I could do `package-list-packages` and having packages list
  select and install `ggtags`

And that's all.

If everything went fine then you have new commands in Emacs starting from `ggtags-`

So a test - I chose some medium project in C and tried:

- `ggtags-mode`
- `ggtags-create-tags` - specifying directory with the code

then finally `ggtags-grep` and any variable/function name shows all occurences in all files
allowing easily to navigate between them all!



### Credits
Big thanks for [tuhdo] who's info pages are really great help!

### Update
There are other possibilities to install ggtags from other Emacs package
repositories, ie. [ELPA][elpa.ggtags].
(Actually it would be nice if it was included as .deb in debian ;-)

# Related links
- [CScope and Emacs][cscope_emacs]
- [C/C++ IDE in Emacs][c-ide.emacs]
- [Post about Emacs package repositories][elpa_melpa] 


[ctags]:          http://stackoverflow.com/questions/30753882/how-to-use-ctags-to-list-all-the-references-of-a-symboltag-in-vim
[cscope.stackex]: http://emacs.stackexchange.com/questions/9499/emacs-cscope-integration-basics
[melpa]:          http://melpa.org/#/
[melpa.install]:  http://melpa.org/#/getting-started
[tuhdo]:          http://tuhdo.github.io/index.html
[cscope_emacs]:   https://www.emacswiki.org/emacs/CScopeAndEmacs
[c-ide.emacs]:    http://tuhdo.github.io/c-ide.html
[elpa_melpa]:     http://emacs.stackexchange.com/questions/268/what-are-the-practical-differences-between-the-various-emacs-package-repositorie
[elpa.ggtags]:    https://elpa.gnu.org/packages/ggtags.html

