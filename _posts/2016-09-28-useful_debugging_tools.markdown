---
layout: post
title: "Debugging tools"
date: 2016-09-08
categories: debugging tools
tags: debugging tools
---
Another info gather post, this time about misc. debugging and reverse engineering tools
(debuggers, tracers, call loggers, network sniffers etc.)
Just a messy (and unfinished) list for now (more maybe later...)

### Debugging tools

U-xs (GNU/Linux, *BSD, ...):

- gdb, ddd
- strace, ktrace, ltrace, ftrace, latrace, mutextrace, xtrace
- LTT (Linux Trace Toolkit) / LTTng: trace, traceview
- truss, sotruss
- dtrace
- systemtap
- dprobes, kprobes
- ptrace, python-ptrace
- cheat engine (for games but not only...)
- debugfs
- kdump

Windows:

- sysinternals tools: filemon, process explorer, ...
- SoftICE (DOS, Win31, 95, .. , XP)
- OllyDbg
- x64dbg

### Network monitoring
- tcpdump
- wireshark

### Disassemblers, hex editors
- IDA (Interactive disassembler) (and IDA Pro - commercial...)
- Biew / Beye (u-x), Hiew (DOS / Windows)

### Decompilers
- [Boomerang][5.1.]
- [hex-rays][5.2.]
- [Reverse engineering compiler][5.3.]

### Other bin utils
- objdump
- nm
- elf
- strip

### Links
- [Disassemblers, decompilers x64 (Wiki)][1.]
- [Wikipedia, Category: Debugging][2.]
- [Sysinternals][6.]

[1.]: https://en.wikibooks.org/wiki/X86_Disassembly/Disassemblers_and_Decompilers
[2.]: https://en.wikipedia.org/wiki/Category:Debugging
[6.]: https://technet.microsoft.com/en-us/sysinternals/bb545021.aspx

[5.1.]: http://boomerang.sourceforge.net/
[5.2.]: https://www.hex-rays.com/products/decompiler/
[5.3.]: http://www.backerstreet.com/rec/rec.htm

