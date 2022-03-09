---
layout: post
title: "Diskless Debian 11 on an Intel PC (i386/amd64) with PXE and rootfs on NBD (INCOMPLETE)"
date: 2022-03-09
categories: debian
tags: debian diskless pxe nbd
---

### Motivation
Definitive failure of an ancient 16G SSD (PATA!) in Dell Vostro a90 nearly lead
to trashing the machine. Not that it is used that much but occasionally it still worked well
(eg. as SNES emulation system or as low-end machine for testing software performance).

At present, the disk with such an interface (miniPCI-e PATA) is nearly impossible
to get. Those available are in the US for some ridiculus price and have even more ridiculus
shipment cost (more than the disk itself...).

Such catastrophies occasionally motivates to make something not necessarily very useful (as is
that ancient machine...) - but are opportunity to play with features of today's operating systems.
An so was this time...

A long time ago (~20 years or so), I was building custom diskless linuxes for computer labs
at my Alma Mater. Having usual infrastructure in place, I was using read-only nfsroot, at some point
improved with some hacks like layered filesystem (unionfs/aufs) - an approach that was/is used with
some live CDs / USBs and some years later became popular with Docker containers. The purpose was similar
- keep image of the original OS intact while letting work with relatively large permissions
and have fresh a OS with each restart (to keep security and reliability on reasonable level).

However, preparing such nfsroot system was taking quite a bit of time (dhbootstrap, manual
scripting for init etc.) - so I looked at other ways (not sure if faster. But other...).

Not using NFS (which AFAIK is the only _network_ filesytem that can be used as root filesystem for linux),
the other way is to have a network disk / block device.
Excluding enterprise solutions (not really great for home...), left me with iSCSI and Network Block Device
(NBD). I knew the first one a bit, as I have built the faculty's first SAN storage system on a Linux machine
with disk array exported with Open-iSCSI target (meaning server, initiator/target was a legacy from SCSI).
It seemed a bit too complex for a home solution, I also read that NBD is way simpler and - what goes along -
lighter and faster.

So - NBD...

### Network configuration
Network services that needs to be in place to go any further: DHCP, TFTP, DNS
(maybe it can be done without the last one, but...).
Detailed configuration of these are rather out of scope (esp. that there each of these can be provided by
several different software packages) - so please have a look elsewhere for this.
I will give only some details regarding this particular setup (also for firewall / iptables etc.).

A little more will be about NBD which is the main focus here.

### Create an empty file for OS image
Eg. 5G:
```
dd if=/dev/zero of=/home/nbd/vostro-debian.img bs=1M count=5120
```

# DHCP
Must have static IP configuration for the host, with filename poiting to pxelinux.0
on you tftp server.
(Unless you have more complex config. - but then you know what to do...).

Must configure properly the default router and DNS (otherwise installer with not reach Debian
package repositories).

# TFTP
Just setup on your server and make it available to the client (firewall). Will contain pxelinux.0 with
its modules, configuration, debian's network installer, then also Debian's kernel and initrd
(for the installed system - as we always boot from the network).

# DNS
Should contain name resolution for server and the clients (if you specify only this in DHCP - also
resolve properly other internet names).

# The NBD server
The NBD server let's you export a file (which can a regular file on a filesystem or,
of course, be another block device) to the network clients. Configuration of the nbd-server
is fairly trivial (example for Debian) - create a file `/etc/nbd-server/`, eg. `/etc/nbd-server/mydebian.conf`
```
[mydebian]
    exportname =  /home/nbd/mydebian.img
    readonly = false
```

`mydebian` is the name of the export (used then with `nbd-client` with `-N` option),
while the `exportname` is the path to the exported image/file (exportname??? WTF?).

Your client will need to access the nbd's port (which by default is tcp/10809).
(Re)start the `nbd-server` - and you're done (you may eventually check system logs).


### Network installer
The first step has to be providing OS installer available on the network, so that booting the machine
using network card (and PXE protocol).

First - you have to get the network installer for the OS (Debian in this case).
For i386, it can be found at:
http://ftp.pl.debian.org/debian/dists/bullseye/main/installer-i386/current/images/netboot/

though in fact the content of:
http://ftp.pl.debian.org/debian/dists/bullseye/main/installer-i386/current/images/netboot/debian-installer/i386/
is enough (talking about the whole subtree of course).

It has to be put on your tftp server and properly configured:
- DHCP - has to be pointing to `pxelinux.0` as boot file.
- `pxelinux.cfg/default` is the default config - either
The simpliest way is to put it as the only thing on the tftp (to the main directory. Normally, you do not have to do
anything more then - but in this case you _have_ to do at least one more thing - option to boot your Debian after
installation. So some customization will be required.

I have decided to put is into some subdir as have more things on tftp / pxelinux. You can eg. put the installer into
a subdirectory. eg. `debian-installer/11/i386` (under your tftp server path!).
and have the following section in your PXELINUX config:
```
LABEL PXE_Debian_11_i386_Installer
   CONFIG debian-installer/11/i386/debian-installer/i386/pxelinux.cfg/default debian-installer/11/i386/debian-installer/i386/
```
The network debian installer is not booting well from a subdirectory - it requires the following modifications:
1. Adding the symbolic link:
```
$ cd debian-installer/11/i386/debian-installer/i386/pxelinux.cfg/
$ ln -s ../ debian-installer
```
2. Change the Debian's `pxelinux.cfg/default` to use relative paths, ie replace:
```
path debian-installer/i386/boot-screens/
include debian-installer/i386/boot-screens/menu.cfg
default debian-installer/i386/boot-screens/vesamenu.c32
```
with 
```
path boot-screens/
include boot-screens/menu.cfg
default vesamenu.c32
```

3. Link the proper modules
If you are not using Debian's version of PXELinux, the modules in `boot-screens/` might not work properly.
You may have to link there the ones from your pxelinux installation.

There are other ways to do it, maybe even better ones to have the installer in a subdirectory and as
a submenu. This way seemed to me the least invasive - others will probably require editing more files
as they have hardcoded paths (this actually could be done better in Debian).





### initrd
The generic initrd does not support NBD and has to be modified. The fastest way I found is to add
a simple script setting-up the NBD:
```
#!/bin/sh

# setup networking
#modprobe r8169
ip link set enp4s0 up
ip link set enp4s0 arp on
ip addr add 192.168.1.10 dev enp4s0
ip route add 192.168.1.0/24 dev enp4s0

# setup nbd
tftp -g -r vostro/bin/nbd-client -l /bin/nbd-client 192.168.1.2
chmod a+rx /bin/nbd-client
modprobe nbd
nbd-client -N mydebian 192.168.1.2 /dev/nbd0
```


add 3 lines to argument parsing code in `/init`:
```
[...]
        nbdroot=*)
                NBDROOT="${x#nbdroot=}"
                ;;
        nfsroot=*)
[...]
```
and execute the `/init-nbd` script in `/init`:
```
[...]
export starttime

if [ "$NBDROOT" ]
then            
        sh /init_nbd
fi
        
if [ "$ROOTDELAY" ]; then
        sleep "$ROOTDELAY"
fi
[...]
```

The `nbd-client` was a bit tricky - the one used for installation could not be used
as it was linked with libraries which are not available inside the initrd image.
The solution was to build a custom `nbd-client` on minimal Debian (with only
gcc, make and libglib2-dev) - so the it is linked to he very basic stuff:
```
$ ldd nbd-3.24/nbd-client
        linux-gate.so.1 (0xf7fce000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7d9b000)
        /lib/ld-linux.so.2 (0xf7fd0000)
```
This is why I decided to just put it on the tftp server and download - so that I can
eventually update it without rebuilding the whole initrd.


### Conclusions
While there are a few tricky places, it seems that Debian (and other distros) could
provide way to install system on NBD device - though it require extracting 
the kernel and initrd and copying it to tftp server. An `scp` from installed system to server
could be sufficient solution (much customization has to be done on server side anyway).


### Useful links
- [NBD][nbd]
- [NBD on github][nbdgit]


[nbd]:    https://nbd.sourceforge.net
[nbdgit]: https://github.com/NBD/nbd
