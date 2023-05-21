---
layout: post
title: "Debian 11 on a diskless Intel PC (i386/amd64) with root filesystem on NBD"
date: 2022-03-09
categories: debian
tags: debian diskless pxe nbd
---

## Motivation
A definitive failure of the 16G SSD (PATA!) in my ancient Dell Vostro A90 nearly lead
to trashing the machine. Not that it is used that much but occasionally it still worked well
(eg. as a low-end machine for testing software performance or even a SNES emulation system).

At present, a disk with such interface (miniPCI-e PATA) is nearly impossible
to acquire. Few are available for some ridiculus price on the other side of the Atlantic (USA)
and have even more ridiculus shipment cost (more than the disk itself...).

Such catastrophies occasionally motivates to make something not necessarily very useful (as is
that ancient machine...) - but gives an opportunity to play with features of today's operating systems.
An so it was this time...

A long time ago (20 years or so...), I was building custom diskless linuxes for computer labs
at my Alma Mater. Having a usual infrastructure in place, I was using read-only nfsroot, at some point
improved with some hacks like layered filesystem (`unionfs/aufs`) - an approach that is used with
some live CDs / USBs and some years later became popular with Docker containers. The purpose was
similar - to keep the image (filesystem) of the original OS intact while letting work
with relatively large permissions and have a fresh OS with each restart (to keep security
and reliability on a reasonable level).

As preparing such an nfsroot system was taking quite a bit of time (`dhbootstrap`, manual
scripting the init process etc.), I looked for other ways to keep my Vostro alive.
Not using NFS (which, AFAIK, at least in a standard unpatched Linux kernel, is the only _network_
filesytem that can be used as the root filesystem), the other way is to have a network disk / block device.
(Obviously) excluding enterprise solutions (not really great for home...), left me with iSCSI
and [Network Block Device (NBD)][nbd]. I knew the first one a bit, as (also many years ago)
I have built the faculty's first SAN storage system on a Linux machine with a disk array
exported with [The Linux iSCSI Target][linux_iscsi_target] (meaning server, "_initiator_ / _target_"
are legacy terms from SCSI).
This still seemed a bit too complex for a home solution. Also, according to some opinions,
NBD is way simpler and - what goes along - lighter and faster.

So - NBD...

## Network configuration
Network services that needs to be available to go any further: DHCP, TFTP, DNS.
Detailed configuration of these are out of scope (esp. that each of these can be provided by
several different software packages). So please, have a look elsewhere for this
(also for firewall / iptables etc.). I will give only some details regarding this particular setup.

A little more will be about NBD which is the main focus here.

### DHCP
Must have a static IP configuration for the client host, with the filename pointing to `pxelinux.0`
on your tftp server. (Unless you have a more complex config. - but then you know what to do...).

It must provide a proper configuration: the default router for your network and the DNS
(otherwise Debian installer with not reach its package repositories).

### TFTP
Must be set up and available for the client (ie. firewall). It will contain `pxelinux.0` with
its modules, configuration, Debian's network installer, later also Debian's kernel and initrd
for the installed system - as we always boot from the network(!).

### DNS
It should contain proper name resolution for the server(s) and the clients
(if only this one is being sent by DHCP - it must also resolve properly all other internet domains).


## Create and export a disk image using NDB
The NBD server allows to export a file (which can a regular file on a filesystem or,
of course, can be another block device, like an LVM volume) to network clients.

So, there are 2 things to do:

1. Creating a file that will serve as the exported NDB image.
For instance, for a 5G file:
```
dd if=/dev/zero of=/home/nbd/vostro-debian.img bs=1M count=5120
```

2. Configure the NBD server to expose the image to the client(s)
Configuration of the `nbd-server` is fairly trivial. The following example is for Debian.
- create a file `/etc/nbd-server/conf.d/`, eg. `/etc/nbd-server/conf.d/mydebian.conf`
```
[mydebian]
    exportname = /home/nbd/mydebian.img
    readonly = false
```

`mydebian` is _the name of the export_ (later used with `nbd-client` with `-N` option),
while the `exportname` is the path to the exported image/file (why they named it `exportname` - no idea...).

The client will need to access the nbd's port (which, by default, is `tcp/10809`).
(Re)start the `nbd-server` - and you're done (you may eventually check the system logs).

Note that by default (set in the global config of the `nbd-server`: `/etc/nbd-server/config`),
the `nbd-server` will run as user `ndb`/group `ndb` - the ownership and permissions to the image file
must be properly set.

## A (fully) network installation
The first step is to provide the OS installer available on the network - so that booting the machine
using network card and the PXE protocol will start Debian installer.

The network installer for the Debian i386 can be found under [`netboot/`][netboot] path in Debian repositories,
though, in fact, the content of [`debian-installer/i386/`][netboot_i386] is enough (talking about the whole subtree of course).

The downloaded contents has to be put on the tftp server and properly configured, ie. the DHCP info sent to the client 
must be pointing to `pxelinux.0` as boot file. Then, in the same directory, `pxelinux.cfg/default` lies the default config.

The simplest way is to put it as the only thing on the tftp server (to the tftp's main export directory) - 
you do not have to do anything more then - Debian installer should load and start.

I have decided to put the installer into some subdirectory (as I have more things available on tftp / pxelinux).
The installer put into eg. `debian-installer/11/i386` (under tftp server path) require adding
the following section in your PXELINUX config:
```
LABEL PXE_Debian_11_i386_Installer
   CONFIG debian-installer/11/i386/debian-installer/i386/pxelinux.cfg/default debian-installer/11/i386/debian-installer/i386/
```
However, the original network debian installer is not booting well from a subdirectory (why oh why???).
It requires the following modifications:
1. Adding the symbolic link:
```
$ cd debian-installer/11/i386/debian-installer/i386/pxelinux.cfg/
$ ln -s ../ debian-installer
```
2. Changing the Debian's `pxelinux.cfg/default` to use relative paths, ie. replace:
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

3. Linking the proper modules
If you are not using Debian's version of PXELinux, the modules in `boot-screens/` might not work properly.
You may have to link there the ones from your pxelinux installation.

There are other ways to do it, maybe even better ones to have the installer in a subdirectory and as
a submenu. This way seemed to me the least invasive - others will probably require editing more files
as they have hardcoded paths (this actually could be done better in Debian).

(Note that Debian installer prepared in this way can be used to install a regular local debian - you just
won't need any CD/DVD/USB image anymore.)

### NBD and Debian installer
While the original Debian installer provides loading nbd modules as an option, there are _no tools allowing
to actually configure and use an nbd device_ (what just makes the nbd modules useless...). This is the major chore
during the installation - it is necessary to get the `nbd-client` binary which will work within
the installation system (which is _not_ the same package as in the regular system - as the installer
does not contain most of the libraries). I extracted the binary from package `nbd-client-udeb_3.21-1_i386.udeb`
(note the [`udeb`][udeb], which is micro-deb or [debian installer package][di_components]).
You can find one on any Debian mirror, eg. [this one][icm_debian_nbd].

The binary has to be transferred to the installation system - tip: check what tools
are available (ftp/tftp/curl/scp/etc) and just make it available for download. Other way would be
a modification of the installator's `initrd` (this would make it permanent, useful in case of creating other
similar installations) - but I haven't considered it worthwhile.

`nbd-client` copied to the Debian installer system allows finally to setup the exported block device:
```
nbd-client -N mydebian nbdserverhostname /dev/nbd0
```
Then, such device can be partitioned, formatted and used by the installer as target.

(In case you cannot partition it from installer - it can also be done locally on the server
using a loop device with whatever program you want. The Debian installer require just specifying
mount points and filesystems).

### Installation of the system
... is as usual. Except for the bootloader - installing it obviously does not make any sense
as the network bootloader (`pxelinux`) will be loaded from network.

Before finishing the installation, remember to copy the kernel and initrd from `/target/` to your server
- they will be needed to boot the new system (NBD and root filesystem will be available _after_
loading kernel and initrd!). If you forget it - again, you can just mount the filesystem with a loop device.

After finishing the installation - a few more things has to be done manually to boot the new system.

## Booting the newly installed Debian on NBD
Configuration for PXElinux:
```
LABEL Debian
  LINUX mydebian/boot/vmlinuz
  INITRD mydebian/boot/initrd.img
  APPEND ip=dhcp nbdroot=192.168.1.1,mydebian,nbd0 root=/dev/nbd0p1
```
The `vmlinuz` and `initrd.img` are symbolic links to the original files copied from `/boot`
of the new system, so that the links can be easily changed on upgrades (or issues) - but of course
you may prefer overwriting the files or editing the config file instead.

(The `nbdroot` parameter is not fully used in the first prototype version of initrd, ie. data provided is not parsed.
It is corrected in the improved version of initrd, see after the next section).

### initrd with nbd support (quick'n'dirty prototype)
The generic initrd does not support NBD and has to be modified. The fastest way I found is to add
a simple script `/init_nbd` setting-up the NBD:
```
#!/bin/sh

# setup networking
#modprobe r8169
ip link set enp4s0 up
ip link set enp4s0 arp on
ip addr add 192.168.1.10 dev enp4s0
ip route add 192.168.1.0/24 dev enp4s0

# download nbd client
tftp -g -r vostro/bin/nbd-client -l /bin/nbd-client 192.168.1.1
chmod a+rx /bin/nbd-client

# setup nbd
modprobe nbd
nbd-client -N mydebian 192.168.1.1 /dev/nbd0
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
as it was linked with libraries which are not available inside the initrd image:
```
# ldd /sbin/nbd-client
        linux-gate.so.1 (0xb7f3f000)
        libgnutls.so.30 => /lib/i386-linux-gnu/libgnutls.so.30 (0xb7cf5000)
        libnl-genl-3.so.200 => /lib/i386-linux-gnu/libnl-genl-3.so.200 (0xb7cec000)
        libnl-3.so.200 => /lib/i386-linux-gnu/libnl-3.so.200 (0xb7cc7000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7adf000)
        libp11-kit.so.0 => /lib/i386-linux-gnu/libp11-kit.so.0 (0xb798a000)
        libidn2.so.0 => /lib/i386-linux-gnu/libidn2.so.0 (0xb7968000)
        libunistring.so.2 => /lib/i386-linux-gnu/libunistring.so.2 (0xb77e6000)
        libtasn1.so.6 => /lib/i386-linux-gnu/libtasn1.so.6 (0xb77cf000)
        libnettle.so.8 => /lib/i386-linux-gnu/libnettle.so.8 (0xb7784000)
        libhogweed.so.6 => /lib/i386-linux-gnu/libhogweed.so.6 (0xb773b000)
        libgmp.so.10 => /lib/i386-linux-gnu/libgmp.so.10 (0xb76ab000)
        libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xb7689000)
        /lib/ld-linux.so.2 (0xb7f41000)
        libffi.so.7 => /lib/i386-linux-gnu/libffi.so.7 (0xb767f000)
        libdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xb7679000)
```
The solution was to build a custom `nbd-client` on minimal Debian (with only
`gcc`, `make` and `libglib2-dev`) - so that it is linked only with very basic stuff:
```
$ ldd nbd-3.24/nbd-client
        linux-gate.so.1 (0xf7fce000)
        libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7d9b000)
        /lib/ld-linux.so.2 (0xf7fd0000)
```
This is why I decided to just put it on the tftp server and download - so that I can
eventually update it without rebuilding the whole initrd...


### initrd with nbd support - more generic and parametrized
The recipe above was a quick prototype of initrd, with hardcoded values (adresses, NBD name etc.).
It was possible to improve this and do it in a less invasive and more flexible (parametrized) way.

1. Modification in [`/init`][init_patched] - add setting `NBDROOT` from the kernel parameter `nbdroot`:
```
[...]
# Parse command line options
# shellcheck disable=SC2013
for x in $(cat /proc/cmdline); do
        case $x in
[...]
        rootdelay=*)
                ROOTDELAY="${x#rootdelay=}"
                case ${ROOTDELAY} in
                *[![:digit:].]*)
                        ROOTDELAY=
                        ;;
                esac
                ;;
        nbdroot=*)
                NBDROOT="${x#nbdroot=}"
                ;;
        nfsroot=*)
[...]
```
2. Add the [`/scripts/nbd`][script_nbd] script (configuring and mounting rootfs on NBD).

3. Put [`nbd-client`][nbd_client_for_installed] binary into `/bin/` (or better `/sbin/`)
on initrd (no need to download it separately).

This way, the nbd device can be configured in the `pxelinux` configuration using the kernel parameter `nbdroot`
(as shown in example above).

(This version of initrd is still not perfect, but it could already be proposed
to implement in Debian. It would make creating systems on NBD much simpler and faster).


## Result and conclusions
My Vostro works fine running Debian on NBD since few months.
The NBD, operating through a 100Mb network card, does not seem to be much slower
than the weird SSD - the machine itself is very slow in today's standards (Intel Atom).
In the meantime, I've made the same setup for another machine (and may do similar things
for some others).

As you can see, while there are a few tricky places, Debian (and probably other distros too)
can be installed and used with root filesystem on an NBD device. If Debian developers
remove the weird obstacles (like the lack of the `nbd-client` in the installer system
and a lack of proper code for setting up an NBD device in the generic init scripts),
Debian could be ready to install and use from NBD nearly as easy as in other types
of installation.

Such installation has some drawbacks:
- it require a bit in infrastructure available (as described)
- it require copying the kernel and initrd to the tftp server on every update(!),
along with updating either symlinks or the pxelinux configuration.

But it also has many advantages, eg. having an image of the system in a file gives
the same flexibility as with a virtual machine: backup is just a file copy,
templating, reuse for any experiments is equally simple - but it runs on a bare metal hardware(!).
Also - the use of some "special" machines (like the one described above) can be extended,
made more robust and easier to recover.

## Follow-up and updates
I have contacted [debian-user][https://lists.debian.org/debian-user/2023/05/msg00853.html]
about the issues and, following the advice, the maintainer of the `nbd-client`.

New things to know:
### nbd-client package
... (deb, not udeb) contains a start-up script and a iniramfs hook for putting
the binary into the initrd (while working on the target system).

I have tested these and unfortunately, there are still issues:
- `nbd-client` binary used for initrd is the one from deb package (linked to libraries
unavailable inside initrd, see details above)
- the nbd startup script is not in the right place:
  it is in `/usr/share/initramfs-tools/scripts/local-top/` while `/init` expects it
  to be in `/usr/share/initramfs-tools/scripts/`
- the nbd startup script do not override the `mountroot` shell function, what is, again,
  expected by `/init` (in which this function is just empty) and, in result, the rootfs is
  just not mounted during the startup...).


### Debian installer
... has the parameter `modules=partman-nbd`, which, according to
`/usr/share/doc/nbd-client/README.Debian.gz` should let configure nbd from
Debian installer.


### to be continued...


## Useful links
- [NBD][nbd]
- [NBD on github][nbdgit]
- [UDEB][udeb]
- [DI_COMPONENTS][di_components]
- iSCSI: [iSCSI initiator][open_iscsi], [iSCSI target (stgt)][stgt], [iSCSI target (istgt)][istgt]
- Discussions on Debian mailing lists:
  - [debian nbd discussion1][debian_nbd_discussion1]
  - [debian nbd discussion2][debian_nbd_discussion2]
  - [debian nbd discussion3][debian_nbd_discussion3]

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

[debian_nbd_discussion1]: https://lists.debian.org/debian-boot/2011/06/msg00401.html
[debian_nbd_discussion2]: https://lists.debian.org/nbd/2019/06/msg00022.html
[debian_nbd_discussion3]: https://lists.debian.org/nbd/2009/07/msg00008.html
