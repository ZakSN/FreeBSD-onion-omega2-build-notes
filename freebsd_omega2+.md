# Building FreeBSD for the Onion Omega 2+:
This document describes how to build FreeBSD -HEAD for the Onion Omega2+ single
board computer and boot it from a USB key.

## Contents:
1. Previous Work
2. Hardware
3. Software
4. Checkout source
5. Get Device Trees
6. Create A New Kernel Configuration File
	1. Modify the KERNCONF
7. Run the Build
8. Build Mfsroot
9. Prepare the Kernel For u-boot
10. Format a USB Key
11. Boot the Omega2+
12. Burning FreeBSD to the Omega2+'s On-Board Flash
 

## 1. Previous Work:
These instructions: <https://github.com/sysadminmike/freebsd-onion-omega2-build>
and (unpublished) notes provided by Jakob Alvermark.

## 2. Hardware:
- Onion Omega2+
- Omega Expansion Dock

## 3. Software:
- devel/uboot-mkimage
- archivers/lzma
- FreeBSD source tree
- freebsd-wifi-build tree (<https://github.com/freebsd/freebsd-wifi-build>)

This build will be using the freebsd-wifi-build system. The freebsd-wifi-build
scripts assume that the source tree is the current directory, and that root and
obj are built relative to the current directory. More information about the 
freebsd-wifi-build system: <https://github.com/freebsd/freebsd-wifi-build/wiki>

## 4. Checkout Source:
```shell
$ mkdir ~/omega2p && cd ~/omega2p
$ git clone https://github.com/freebsd/freebsd.git
$ git clone https://github.com/freebsd/freebsd-wifi-build.git
```

## 5. Get Device Trees:
```shell
$ cd ~/omega2p/src/sys/gnu/dts/mips
$ fetch https://raw.githubusercontent.com/WereCatf/source/image/target/linux/\
ramips/dts/OMEGA2.dtsi
$ fetch https://raw.githubusercontent.com/WereCatf/source/image/target/linux/\
ramips/dts/OMEGA2.dts
$ fetch https://raw.githubusercontent.com/WereCatf/source/image/target/linux/\
ramips/dts/OMEGA2P.dts
```

## 6. Create a New Kernel Configuration File:
Kernel configuration is based on `MT7628_FDT`.
```
$ cd ~/omega2p/src/sys/mips/conf
$ cp MT7628_FDT ONIONOMEGA2P
```
### 6.1. Modify The KERNCONF:

In ONIONOMEGA2P change the .dts file used:
```
makeoptions	FDT_DTS_FILE=OMEGA2P.dts
```

To enable booting from a USB key change root device name:
```
options	ROOTDEVNAME=\"ufs:/dev/da0s2a\"
```

Comment out debugging options in ~/omega2p/src/sys/mips/mediatek/std.mediatek
```
#options 	INVARIANTS
#options 	INVARIANT_SUPPORT
#options 	WITNESS
#options 	WITNESS_SKIPSPIN
#options 	DEBUG_REDZONE
#options 	DEBUG_MEMGUARD
```

and uncomment some necessary options:
```
options 	SOFTUPDATES	# Enable FFS soft updates support
options 	UFS_DIRHASH	# Improve big directory performance
options 	MSDOSFS		# Enable support for MSDOS filesystems
```

## 7. Run the build:
Kernel configuration and .dts files must be specified for the build:
```
$ cd ~/omega2p/src
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink buildworld
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink buildkernel
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink installworld
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink installkernel
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink distribution
```

This will produce a kernel in `~/omega2p/tftpboot`.

## 8. Build Mfsroot:
```
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink mfsroot
$ X_DTS_FILE=OMEGA2P.dts KERNCONF=ONIONOMEGA2P\
    ../freebsd-wifi-build/build/bin/build ralink fsimage
```

This will produce a root filesystem in `~/omega2p/mfsroot/ralink`.

## 9. Prepare the Kernel For u-boot:
```
$ cd ~/omega2p/tftpboot
```

Check kernel entry point address:
```
$ readelf -h kernel.ONIONOMEGA2P
```

Look for the following line:
```
  Entry point address:               0x80001120
```
record the entry point address. In this case it is `0x80001120`

Convert the kernel to binary:
```
$ objcopy -O binary kernel.ONIONOMEGA2P kernel.ONIONOMEGA2P.kbin
```

Compress the binary, using the ports version of lzma:
```
$ /usr/local/bin/lzma e kernel.ONIONOMEGA2P.kbin kernel.ONIONOMEGA2P.kbin.lzma
```

Make a u-boot image:
```
$ mkimage -A mips -O linux -T kernel -C lzma \
    -a 0x80001120 -e 0x80001120 -n "FreeBSD" \
    -d kernel.ONIONOMEGA2P.kbin.lzma kernel.ONIONOMEGA2P.lzma.uImage
```

note that the -a and -e parameters use the entry point address recorded earlier

## 10. Format a USB Key:
The USB key needs a FAT partition for the kernel and a UFS partition for the 
root filesystem. Assuming that the USB key is attached as `/dev/da0`:
Clear the existing partition table:
```
# gpart destroy -F da0
```

Create a new partition table:
```
# gpart create -s mbr da0
```

Create a 16MB FAT partition:
```
# gpart add -t \!12 -s 16m da0
```

Create a FreeBSD slice in the remaining space and add a BSD partition table for 
it:
```
# gpart add -t freebsd da0
# gpart create -s bsd da0s2
```

Create a UFS partition on the slice:
```
# gpart add -t freebsd-ufs da0s2
```

Create a FAT filesystem and a UFS filesystem:
```
# newfs_msdos /dev/da0s1
# newfs /dev/da0s2a
```

Copy the Kernel to the USB Key:
```
# mount -t msdosfs /dev/da0s1 /mnt
# cp ~/omega2p/tftpboot/kernel.ONIONOMEGA2P.lzma.uImage /mnt
# umount /dev/da0s1
```

Copy the Root Filesystem to the USB Key:
```
# mount /dev/da0s2a /mnt
# cp ~/omega2p/mfsroot/ralink/ /mnt
# umount /dev/da0s2a
```

## 11. Boot the Omega2+:
Plug the USB key into the port on the expansion dock. Plug a micro-USB cable 
into the expansion dock and connect it to the host computer. The expansions 
dock's micro-USB port provides power and a USB serial port. If no other USB 
serial devices are connected to the host the expansion dock should show up as
`/dev/cuaU0`. To connect at 115200 baud:
```
# cu -s 115200 -l cuaU0
```

Hold down the reset button on the expansion dock and switch it on. The 
following menu should appear in the terminal:
```
Please select option: 
   [ Enter ]: Boot Omega2.
   [ 0 ]: Start Web recovery mode.
   [ 1 ]: Start command line mode.
   [ 2 ]: Flash firmware from USB storage. 

press 1 to start the u-boot command line. From the u-boot command line:

Omega2 # usb reset

to re-scan for USB devices. Ensure that the kernel image is there:

Omega2 # fatls usb 0:1 
*
*
*
**
  1221598   kernel.onionomega2p.lzma.uimage 

1 file(s), 0 dir(s)
```

Load the kernel:
```
Omega2 # fatload usb 0:1 0x80800000 kernel.onionomega2p.lzma.uimage
```

Once the kernel has been loaded boot it:
```
Omega2 # bootm 0x80800000
```

The Omega2+ should boot to the login prompt. Log in with user root.

## 12. Burning  FreeBSD to the Omega2+'s On-Board Flash:
The Omega2+ has 32MB of internal flash memory. The kernel can be burned into
this flash memory allowing one to boot FreeBSD without the detour through
u-boot. The process is quite simple as u-boot provides an automatic interface
to do this. This is documented on Onion's site:

<https://onion.io/2bt-firmware-flashing-from-usb-storage/>

Note that in the above link 'firmware' refers to Onion's flavour of Linux, not
u-boot.

The same process can be used to flash FreeBSD's kernel. Mount the USB key's 
FAT partition and rename the kernel to `omega2.bin`
```
# mount -t msdosfs /dev/da0s1 /mnt
# mv /mnt/kernel.ONIONOMEGA2P.lzma.uImage /mnt/omega2.bin
# umount /dev/da0s1
```

Connect to the Omega2+ over serial and boot into the u-boot menu as described
in the previous section (Boot the Omega2+). This time press 2 instead of 1. The
Omega2+ should load the file `omega2.bin`, flash it and then reset. If 
everything went well FreeBSD should now boot every time the Omega2+ is powered
on.

NOTE: At the time of this writing Onion's fork of u-boot is deeply flawed. The
repo is available here:

<https://github.com/OnionIoT/omega2-bootloader>

This is a quote from the above repo's main README:

>Sometimes burning stalls and the process hangs. Oddly enough, the only common
denominator seems to be the presence of symbol '6' in the string representation
of the size of the executable. Thank GOD, it's not '666' :) If that happens to
you, don't panic. Just add (or remove) some unimportant functionality, such as
printouts for example, that will change the size, and chances are the process
of burning will proceed as expected.

During testing this issue was encountered. As the README suggests the solution
was to change the size of the kernel to be burned. The kernel's size was 
changed by adding the following line:
```c
const char waste_of_space[1000] ={0};
```

The kernel was then recompiled, renamed and successfully flashed.
