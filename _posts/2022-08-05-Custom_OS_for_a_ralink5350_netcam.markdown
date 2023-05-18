---
layout: post
title: "Building a custom OS image for Belkin NetCam (an unfinished draft)"
date: 2022-08-05
categories: embedded
tags: embedded netcam openwrt kernel devicetree
---


A few years ago, I bought a Belkin Netcam (the exact model id: F7D7601v1), which,
despite of annoying cloud-only availability, served decently. For some time at least...

It was until the moment when Belkin withdraw its support for these cameras (_add a link_).
Normally such a thing means, that if the device breaks or there is a software bug
(eg. a security vulnerability) then your are on your own.
But we are in the new, wonderful era of cloud-connected and cloud-managed IoT,
so now the end of support means disabling the cloud service, which, of course,
made the device useless as owner/user cannot connect to it, even on a local
(cable) network.

I find this kind of practices particularly despicable, especially in times when
the environment is becoming our priority. We need some strict legislation that
will not allow companies doing such things for free - a device for which
the support ends should get a firmware update which gives full control of
the device to the user. Also, an SDK with documentation should be provided so
that user can continue using the hardware without any support.

But enough politics.... The situation pissed me off enough to make an attempt
to hack or just replace the existing system with something that would give me
back control over the device I own.

And here we are...

## Accessing the device
What I wrote above is not entirely true - the device could have been accessed
(_link to the vuln_), but in a very limited way. Basically, in the setup mode
(selected with a switch), the camera would open a web service, which was normally
used by a dedicated mobile phone app to configure the device.
Of course, no official information on how to use its API...

The service, as also described in several places, can be used to execute
a command on the remote service eg. start `telnetd` which gives direct access
to terminal console on the NetCam's system.

The system however contains very small number of utilities - no file utilities
(`tar`), also transferring a file from/to was a challenge. The only available
thing was busybox' `tftp` client, which was not getting along with `tftpd-hpa`
that I had on my system. I connected the camera directly to the 2nd network card
so that I could isolate it completely from the outside world and monitor all
eventual network traffic.

### Tftp
Having a tftp client and tftp server, things should be simple - right?
Not quite this time...

The files received by the `tftpd-hpa` from the camera had constantly the very
useful length of zero. Traffic monitoring with `wireshark` revealed that
the file was actually send through the network. So the problem was on
the receiver(???).

The reason for this was as follows: there was a timeout on the client waiting
for tftpd's answer for the request (RRQ / _add link to RFC_) and the busybox'
tftp client is not patient enough and sends subsequent RRQs (multiple!).
The `tftpd-hpa` is `fork()`-style network daemon, so it just spawns a new
process for each incoming request. In consequence, the first transfer actually
succeeds, but the following ones do not (because the client ends its work when
it sends the last piece of the data). The `tftpd-hpa` opens the destination
file before receiving any data (after just the request (WRQ)) what in the case
of the following processes _truncates_ the file already received and written
by the `tftpd-hpa` process serving the first request.

For a UDP-based protocol (like tftp), the application has to manage state of
the connection - and it seems to be buggy in `tftpd-hpa`...
Subsequent requests for uploading the same file from the same client (meaning:
IP + port !) should not be served (should be dropped!) until the already
started file transfer is finished.

(It was a while ago and I cannot recall if I tested `atftpd` - it works on
threads and may behave better. However, a brief look at the code suggests
otherwise - it does not check for on-going transfers, just starts new threads...)

I could start playing with proper configuration (name resolution seemed to be
the source of the delay), but this struck me as an absurd. I have a device which
I quickly need to test, and I cannot use just its IP to do basic things...

#### Tftpd in Python
Modifying `tftpd-hpa` did not seem feasible - the change would be tricky,
it would require some central place for keeping the list of active transfers
and some inter-process communication (monitoring when transfers ends etc.).
`Atftpd` - seems more feasible (one process with threads) but still similar
problem - the software itself is complex, require proper configuration etc.
while I need just the simplest thing possible (without disturbing my system).

Since the tftp protocol is 'trivial', I decided to deal with the problem
writing the simplest possible tftp server (and client), so that if any other
problem occurs, I would have a complete control and a simple code to
experiment with.

The `tftpd.py` is very limited: it serves just 1 request at a time (what
solves the problem described above) and it does not support block size
negotiation (works only with generic 512-byte long blocks).
But, finally, it lets to transfer files to/from the camera (from its original OS).

## Backup
`dd` and `tftpd.py` let me to make backup of raw `mtd` devices.
I wanted also to transfer all the files (for simple analysis on a PC).
But... no `tar` or similar...

Since one of the ways to deal with the camera was modifying the original system,
I decided to find a way to build software for the system. First, I was going to
try to build a custom toolchain (finding the same `gcc`, `uLibc`), but since
these things are old, I realized that it is rather overshot for the problem.
If I am going to build a custom toolchain, than rather with a newer GCC etc.

Fortunately, I have found the original SDK(!), with original (patched!) sources
of the kernel, software, documentation etc. (!).
It was lacking `gcc` - but I managed to find it too([buildroot-gcc342.tar.bz2][gcc342]).
This opened a way to build (or at least - rebuild) the software to patch or
add some missing features.

I rebuild `busybox` (adding more utilities), `dropbox` (to eventyally replace
`telnetd`) and `uvc_stream` (for custom streaming).
Everything worked - the only thing is that to run `uvc_stream`, we need to free
the `/dev/video0` device, used by customized `goahead` webserver which was
also streaming MJPG from the camera. This had however limited use - after short
time the camera is rebooted (some watchdog process is checking if `goahead`
is working, if not it reboots the system).

But - I could make a backup of the files, and have a closer look what is there.
It is tightly constructed system (I am used to rather larger systems...), with
some scripting and configuration that was ensuring that the system was a working
agent in operational state (as foreseen for distance usage with connection to
the company's cloud). Modifying this could possibly be done, by I was stuck on
the workflow - I would need build and to prepare proper system image and run it. 

The only way to do it (without overwriting the firmware - which could "brick"
the device) was to access `Uboot`, and that required a ...

## Serial console
As an eletronics rookie, I grabbed my old T23p with a physical 9-pin serial
(RS-232) port, connected the pins, started and setup `minicom`, reset the camera
and... got trash on the screen. It was clearly visible that there is a connection,
data goes through but it was distorted. Initially, I thought that the problem was
the amature connection, but after testing 2 cables and several ways of connecting
them to the camera, nothing helped.

Not having any way for testing it more, I have ordered several things that could
help (like some simple soldering station, "golden" pins and some cheap logic
analyzer, among others small pieces).

Soldering things did not help. So the next thing was to check the logic analyzer.
I got a simple device that could be easily used with `sigrok` / `pulseview`
(just selecting the proper driver ... ).
`Pulseview` has nice features of decoding certain type of bus / wire protocols,
which fortunately includes UART. I started gathering the data - and got surprised.
The data sent from the camera was correct (I could read the text, `uboot` stuff
and such). So the problem was at the receiving side...

Started digging, people suggested that `minicom` is not good, better use `kermit`.
OK. tried that (`ckermit`) - the same thing. Tried `screen -L /dev/ttyS0 57600`,
the same thing. Digging more, I have finally found out the reason: RS-232 serial
_is not_ the same thing as TTL serial (_links!_). It has different voltage (!)
and even reversed interpretation of high/low levels(!). I was actually lucky that
I haven't damaged the camera's serial lines with higher voltage of RS-232...
It is suprising how difficult was to dig out this information - while it should
be in wikipedia(!) as very basic stuff to know!

Anyway - I knew what the problem was, I just needed a TTL serial in the PC...
I could order one (on USB), but having a few Arduinos and a RasberryPie around,
I decided to use one of these. RaspberryPie seemed the simplest - I just disabled
console in the system (RPie's system is usually configured to provide the console
for itself), connected the camera, started  `screen -L /dev/ttyAMA0 57600`
on the RPie - ... and it worked! I got to...

## UBoot and buildroot 
Here's were the fun began (finally). `Uboot`'s menu allowed finally to control
and select from where the system will be loaded/started - so load and test
something else than what is in the orig. firmware.


## First custom toolchain and (embedded) system
The above opened a way for (finally) loading completely custom (or at least
customized) system to the device. I decided to try to build something new -
even just to see how it goes.

Docker provides a nice isolation for such use cases - I downloaded buildroot to
the newly created Debian container and started playing.

I've build a new toolchain, it went surprisingly easy, it took a while,
though... Then I made a setup for MIPS/ralink305x/experimental device
(_or sth like this - to check_), and build with the target of `uboot` image.

Now the big test - camera's uboot version leaves only one option for booting a custom system:
```
# tftp <address> <IP> <remotefile>
# bootm <address>
```
- but where the hell am I supposed to load the file???
I took RA5350's memory map in hand, after couple of tests I found out that `0x8200000`
is kind of an optimal address for loading the kernel, so that:
```
# tftp 0x8200000 <IP> <remotefile>
# bootm 0x8200000
```
should do the trick.

Note that:
1. both adressses has to be the same and
2. they have to be different and outside of the destination place for the kernel
   (`file image.bin` should provide this information). Otherwise the system with probably hang.
3. I think it is also wise not to load to any mtd/firmware address space
   (haven't arrived to that part yet, not sure how it works...).

And it somewhat worked. Somewhat, because it hasn't detected some stuff
(the camera) and for some reason it did not see the ramdisk with the system
that was supposed to be built along.

Suspecting that there can be custom stuff in the original firmware, I had
a look and compared the original 2.6.26 with the one from RaLink's SDK - there
is quite some stuff modified, new drivers, support for MIPS ralink boards etc.

Looking at the kernel in the latest buildroot (5.x.x), one can see immediately
that porting is not a trivial thing, a lot of things changed and I do not have
expertise to do that. It is an option - but not a quick one for sure...

# OpenWRT image for another ralink5350 device
Encouraged by http://www.wanda25.de/wansview_ncs601w.html
I have decided to give it a try. It actually loaded quite well (tftp,
no changes in flash), with networking operational (cable at least), but the OS
was seeing only half of the memory (32MB, instead of 64). This was quite
limiting in terms of tests as rootfs resides in RAM...


# Custom OpenWRT image
Since this looked promising, I had a look a the latest `openwrt` from git repo:

https://github.com/openwrt/openwrt.git
https://git.openwrt.org/openwrt/openwrt.git

and it seemed even more promising. The kernel code contained already stuff
for RT5350 so maybe it will work...

https://openwrt.org/docs/guide-developer/toolchain/buildsystem_essentials

Indeed, there were some options in the configuration of `openwrt` that allowed
to build an image similar to the one I tried above, but still it was missing
half of its memory...

So no other way - must dig deeper into kernel.
OpenWRT contains configurations and patches to support many devices which are
not available in "vanilla" Linux kernel. In particular, there are plenty of
configurations for embedded devices, with proper `dts` (devicetree source) files,
options in configuration menu, additional drivers etc.
And there is a number of devices based on RT5350 - what is a very good sign!

The only thing is either to find one matching the camera (no luck here), or
add configuration

https://openwrt.org/docs/guide-developer/adding_new_device

at least `dts` mappings and menu options: so that devices and system resource
can be mapped properly (ie. mentioned RAM size).


`... to be continued ...`



## Useful links
- [NBD][nbd]
- [NBD on github][nbdgit]
- [UDEB][udeb]
- [DI_COMPONENTS][di_components]
- iSCSI: [iSCSI initiator][open_iscsi], [iSCSI target (stgt)][stgt], [iSCSI target (istgt)][istgt]

[nbd]:    https://nbd.sourceforge.io
[nbdgit]: https://github.com/NetworkBlockDevice/nbd
[udeb]:   https://wiki.debian.org/udeb
[di_components]: https://d-i.debian.org/doc/internals/ch03.html
[netboot]: http://ftp.pl.debian.org/debian/dists/bullseye/main/installer-i386/current/images/netboot/
[netboot_i386]: http://ftp.pl.debian.org/debian/dists/bullseye/main/installer-i386/current/images/netboot/debian-installer/i386/
[icm_debian_nbd]: http://ftp.icm.edu.pl/debian/pool/main/n/nbd/

[open_iscsi]: https://www.open-iscsi.com/
[linux_iscsi_target]: https://linux-iscsi.org/wiki/iSCSI
[stgt]:  http://stgt.sourceforge.net/
[istgt]: http://www.peach.ne.jp/archives/istgt/

[init_patched]: https://github.com/t-w/debian_on_nbd/blob/main/patch/init
[script_nbd]:  https://github.com/t-w/debian_on_nbd/blob/main/patch/scripts/nbd
[nbd_client_for_installed]: https://github.com/t-w/debian_on_nbd/blob/main/patch/bin/nbd-client

[gcc342]: https://github.com/MediaTek-Labs/linkit-smart-7688-uboot/blob/master/buildroot-gcc342.tar.bz2
[mediatek_uboot]: https://github.com/MediaTek-Labs/linkit-smart-7688-uboot
