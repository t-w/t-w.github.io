---
layout: post
title: "Software debugging tools"
date: 2016-09-28
categories: software debugging tools
tags: software debugging tools
---
Another info gather post, this time about misc. debugging and reverse engineering
software tools (debuggers, tracers, call loggers, network sniffers etc.)
Just a messy (and unfinished) list for now (may be updated later...)

### Software debugging tools

U-xs (GNU/Linux, *BSD, ...):

- `gdb`, `ddd`
- `pstack`
- `strace, ktrace, ltrace, ftrace, latrace, mutextrace, xtrace, etrace`
- LTT (Linux Trace Toolkit) / LTTng: `trace`, `traceview`
- truss, sotruss
- gprof (call graph profile data)
- `valgrind`
- dtrace
- systemtap
- dprobes, kprobes
- perf-tools (kprobe, io/open/execsnoop, functrace, ...)
- ptrace, python-ptrace
- cheat engine (for games but not only...)
- debugfs
- kdump
- gcore (dumping process mem / core)
- [DynamoRio][dynamorio]
- [Flawfinder][flawfinder]
- cppcheck
- `ld` / `ldd`
  - `LD_PRELOAD`
  - `LD_BIND_NOW` (and -Wl, -znow) - resolve functions on load (not on the 1st call/lazy way)
  - `ldd` and `LD_DEBUG`

Windows:
- sysinternals tools: filemon, process explorer, ...
- SoftICE (DOS, Win31, 95, .. , XP)
- OllyDbg
- x64dbg

Java:

- [FindBugs][findbugs]
- [Jlint][jlint]

### Network monitoring
- `tcpdump`
- `wireshark`

### Disassemblers, hex editors, analysers
- IDA (Interactive disassembler) (and IDA Pro - commercial...)
- Biew / Beye (u-x), Hiew (DOS / Windows)
- ht

### Decompilers
- [Boomerang][5.1.]
- [hex-rays][5.2.]
- [Reverse engineering compiler][5.3.]

### Other bin utils
- `objdump`
- `nm`
- `elfedit`
- `strip`
- c++filt

### On-line tools
- [Compiler explorer][compiler_explorer]

### Links
- [Disassemblers, decompilers x64 (Wiki)][1.]
- [Wikipedia, Category: Debugging][2.]
- [Sysinternals][6.]

Related talks:
- [The Journey of a source line: how your code is translated into a controlled flow of electrons][anowak_talk_cern]
- [The bits between the bits: How we get to main()][bitsbetweenbits]
  - C/C++ startup code, linking, symbol resolution, debugging

[1.]: https://en.wikibooks.org/wiki/X86_Disassembly/Disassemblers_and_Decompilers
[2.]: https://en.wikipedia.org/wiki/Category:Debugging
[6.]: https://technet.microsoft.com/en-us/sysinternals/bb545021.aspx

[5.1.]: http://boomerang.sourceforge.net/
[5.2.]: https://www.hex-rays.com/products/decompiler/
[5.3.]: http://www.backerstreet.com/rec/rec.htm

[compiler_explorer][https://godbolt.org/]

[dynamorio]: http://dynamorio.org/
[flawfinder]: https://dwheeler.com/flawfinder/
[findbugs]: http://findbugs.sourceforge.net/
[jlint]: http://jlint.sourceforge.net/

[anowak_talk_cern]: https://mediastream.cern.ch/MediaArchive/Video/Public2/weblecture-player/index.html?year=2018&lecture=668207&ftime=00:00:05#
[bitsbetweenbits][https://yotube.com/watch?v=dOfucXtyEsU]
