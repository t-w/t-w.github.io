---
layout: post
title: "Rescuing broken initramfs (Debian Buster - but might work for others)"
date: 2019-01-08
categories: debian administration
tags: boot startup initramfs initial ramdisk initrd repair
---
I have just bumped on an issue that usually was the Ubuntu's domain (in my case ;) -
my Debian 'testing' (buster) station have not started after reboot... It broke 
after installing some updates (including kernel - 4.18-2) and rebooting the system.
It stopped trying to unlock/find swap device (which was on an encryted partition).

It seems the reason for it might be the change of device id (sda became sdb), despite
the fact that I used UUIDs everywhere... Not sure if it was my fault, or Debian
left something using `sda` in configuration of encrypted devices...
Anyway - the boot just left me with the prompt of initramfs.

Knowing that the swap is the reason, I wanted to disable it (or whatever), boot 
the system and see how can I repair it (maybe just rebuild initramfs).

Not that simple... I do not have the Debian installation disk at the moment,
so the standard rescue method does not work. I have just SystemRescueCD on my network boot,
and there should be a standard way to do it with standard tools - should be enough, right?

Having some experience with rebuilding initrds in the past (eg. modifying some live systems
to boot over network using NBD), I took the standard tools: `gzip`, `xz`, `cpio` -
and got access only to something very little, containing something like microcode...

Googling I found [the Debian page about initramfs][1.], which correctly explained that
the initramfs has 2 parts - the first with microcode (cpio), and the second with "standard"
initrd (gzipped cpio).

Step by step how I managed to repair it:

1. Extract the initial ramdisk
1.1 check the size of microcode initrd:
{% highlight bash %}
$ cpio -t < initrd-4.18-2.img
kernel
kernel/x86
kernel/x86/microcode
kernel/x86/microcode/.enuineIntel.align.0123456789abc
kernel/x86/microcode/GenuineIntel.bin
28 blocks
{% endhighlight %}
1.2 get standard initrd (use the size got above in skip parameter)
{% highlight bash %}
$ dd if=initrd-4.18-2.img of=initrd-4.18-2_initramfs.img bs=512 skip=28
{% endhighlight %}
1.3 extract the initramfs (in some empty directory!)
{% highlight bash %}
$ gzip -cd initrd-4.18-2_initramfs.img | cpio -i
{% endhighlight %}
2. Fix whatever necessary (in my case I had to comment/remove swap from crypttab)
3. Rebuild initrd
{% highlight bash %}
$ dd if=initrd-4.18-2.img of=initrd-4.18-2_fixed_microcode.img bs=512 count=28
$ find . | cpio -H newc -o | gzip -c > initrd-4.18-2-fixed_initramfs.img
$ cat initrd-4.18-2_fixed_microcode.img initrd-4.18-2-fixed_initramfs.img > initrd-4.18-2-fixed.img
{% endhighlight %}

Again, I feel stronger that switching to FreeBSD might be the best option in the era with
Linuxes broken with systemd, unreliable/random disk ordering...

Also - encryption of things critical for system startup seem ridiculus - unless you are crazy
about privacy, the operating system, programs, local system configuration should be
RELIABLE, ROBUST and EASY TO ACCESS AND REPAIR(!). Encrypt only what's worthy and do not
make system startup dependent on that kind of stuff (I felt it was bad decision but thought
I'd give it a try - and in result I wasted time on reparing something that should give no trouble...).


Links
--------------------
1. [Debian wiki about initramfs][1.]

[1.]: https://wiki.debian.org/initramfs
