# How to get a real-time Linux running on a Raspberry Pi using Buildroot

We will use Xenomai and a RPi 1 because RPi 2 is not yet officially supported by Xenomai. The examples below uses RPi B but you could adjust the steps to any RPi of the same family.

The best resources about Xenomai are obviously the http://xenomai.org website in its globality so I will not post the entire list of webpages I read to achieve my goal and write this memo, just take into account that I read them all.

Xenomai offers two versions, the 2.0 which is now deprecated, and the 3.0 we should use. Unfortunately, I failed by installing the 3rd version and FYI, I did not find any example of the net of god people having installed it successfully. So this is a guide to install Xenomai 2.6 Cobalt core (ie. a dual kernel configuration).

In this configuration, the hardware is abstracted by an Adeos/I-pipe layer. Above this, there are two concurrent kernels, Linux as always (not real-time) and Xenomai. Linux is completed by a standard userspace so we can have a standard shell and any utility we are used to work with. Xenomai is completed with "skins" that represent the API to use to create real-time tasks. Between both cores, there is an interrupt shield that filters signals between them in order to make sure that a non-real-time task will not be scheduled over a critical one.

This is a step by step guide.

Let's start by creating and filling a directory dedicated to this experimentation:
```
WORKDIR=/home/${USER}/real-time-rpi
mkdir -p $WORKDIR
cd $WORKDIR
```

1- Get Linux
============

First get Linux from the RPi community and focus on the last stable branch supported by Xenomai, 3.8.y:
```
git clone -b rpi-3.8.y git://github.com/raspberrypi/linux.git linux-rpi/
```
and checkout a new branch based on this one to receive future patches:
```
git checkout -b rpi-xen-3.8.y
```

2- Get Xenomai
==============

Then, besides the Linux repository, get your own copy of Xenomai 2.6, the last stable branch is 2.6.4:
```
git clone -b v2.6.4 git://git.xenomai.org/xenomai-2.6.git xenomai-2.6/
```
There we are in detached HEAD state so create your own branch to work into:
```
git checkout -b 2.6.4
```

Now you can browse Xenomai sources and observe in the folder xenomai-2.6/ksrc/arch/arm/patches three patches (for different Linux versions) named `ipipe-core-<linux_version>-<arch>-<patch_version>.patch`. There are the only available Linux kernel versions known to be working for ARM architecture. But, because RPi is not fully supported, there are two other patches available in the raspberry/ directory with -pre and -post appended. Furthermore, they are only available for 3.8.13 kernel version, so we will have to use this version and not another close one.

3- Get Buildroot
================

I am sure Buildroot may be used to automate the full process of patching/compiling but because Raspberry Pi kernels have to be patched thrice in a particular order, I could not get it working with the last stable branch of Buildroot. Anyway, to find documentation about this project: https://buildroot.org/.
```
git clone -b 2015.11.x git://git.buildroot.net/buildroot buildroot/
```

To lighten a bit the commands we type, the following three variables are set:
```
BUILDROOT=${WORKDIR}/buildroot
LINUX=${WORKDIR}/linux-rpi
XENOMAI=${WORKDIR}/xenomai-2.6
```

4- Produce the toolchains and the rootfs
========================================

Buildroot has the same manner of Linux (and this is very appreciable) to deal with default configuration files. They are located in buildroot/configs/. We will use the raspberrypi_defconfig file as a base.
```
cd $BUILDROOT
make raspberrypi_defconfig
make menuconfig
```
Now apply the following changes:
```
Toolchain
  |- linux version (3.8.13)
  |- Custom kernel headers series (3.8.x)
Kernel (Unselect)
```
And finally build it:
```
make
```
The toolchain generated by buildroot is located under ${BUILDROOT}/output/host/usr/bin/ and is named `arm-buildroot-linux-uclibcgnueabihf-`.

5- Configure your kernel for a non-real-time experience
=======================================================

Buildroot-compiled 3.8.13 Linux kernel does not boot on the Raspberry Pi because the package providing the boot files provides the device tree sources but supposes the device tree compiler exists. With this fresh installation, and using raspberrypi_defconfig file, there is no flattened device tree support so the device tree compiler is not built when building the kernel. Hence, we cannot build the provided device tree sources (although the DTC sources are present in the Linux kernel tree under scripts/dtc/). So we need to enable it manually. But to be able to configure our Kernel using the menuconfig command, we need a valid toolchain. That is why we compiled first the toolchain using Buildroot before starting a full build with the Kernel. To do this configuration:
```
cd $LINUX
make ARCH=arm CROSS_COMPILE=${BUILDROOT}/output/host/usr/bin/arm-buildroot-linux-uclibcgnueabihf- bcmrpi_quick_defconfig
make ARCH=arm CROSS_COMPILE=${BUILDROOT}/output/host/usr/bin/arm-buildroot-linux-uclibcgnueabihf- menuconfig
```
In this menu apply the following change:
```
Boot options
  |- Flattened Device Tree support (Select)
```

Save your configuration and copy it at the right location
```
cp .config arch/arm/configs/bcmrpi_quick_xeno_defconfig
```

6- Compile this non-real-time kernel and boot the RPi
=====================================================

Let's go back to Buildroot and apply some changes to the configuration to select our defconfig file, and finally, produce all the files to boot the RPi.
```
cd $BUILDROOT
make menuconfig
```
Select back the Kernel option and apply this configuration:
```
Kernel
  |- Kernel version (Local directory)
  |- Path to the local directory (Absolute path to the linux-rpi directory, the value of $LINUX)
  |- Kernel configuration (Using an in-tree defconfig file)
  |- Defconfig name (bcmrpi_quick_xeno)
  |- Kernel binary format (zImage)
```
Then launch the build:
```
make
```

During the build, you might setup your micro SD card to this kind of layout (I personnally use gparted for that):
```
| Partition | File system | Label  | Size   | Flags    |
| --------- | ----------- | ------ | ------ | -------- |
| 1         | fat16       | boot   | 32MiB  | boot,lba |
| 2         | ext2        | rootfs | 300MiB | -        |
```

After 30 to 45 minutes on a decent computer you should get all the files you need to boot your RPi, Buildroot outputs them in buildroot/output/images directory:
```
cd ${BUILDROOT}/output/images
```
My SD card is mounted automatically at /media/${USER}/boot and /media/${USER}/rootfs, eventually adapt thoses values to your computer setup (may need root accesses):
```
cp rpi-firmware/* zImage /media/${USER}/boot/
tar -C /media/${USER}/rootfs/ -xvf rootfs.tar
```
Before unmounting, you have to add the following to the cmdline.txt file (at the beginning of the line, *all on the same line*): "`console=ttyAMA0,115200 init=/bin/sh `" and then:
```
umount /media/${USER}/{boot,rootfs}
```
Configure your favorite serial communication program to read on the right tty at 115200 bauds, without any flow control.

At this point, you should have a booting RPi running... nothing more than a bare metal distribution.

7- Let's patch that kernel
==========================

Now our setup is up. This section will use the patches from the Xenomai tree to add real-time support to our RPi kernel sources.
```
cd $LINUX
patch -p1 < ${XENOMAI}/ksrc/arch/arm/patches/raspberry/ipipe-core-3.8.13-raspberry-pre-2.patch
patch -p1 < ${XENOMAI}/ksrc/arch/arm/patches/ipipe-core-3.8.13-arm-4.patch
patch -p1 < ${XENOMAI}/ksrc/arch/arm/patches/raspberry/ipipe-core-3.8.13-raspberry-post-2.patch
```
Once the kernel is patched, we have to launch Xenomai "setup" script on thoses sources that way:
```
${XENOMAI}/scripts/prepare-kernel.sh --arch=arm --linux=$LINUX --adeos=${XENOMAI}/ksrc/arch/arm/patches/ipipe-core-3.8.13-arm-4.patch
```

A priori, everything is already set up in the kernel configuration, but some entries may corrupt Xenomai so we have to unselect them (to understand why, please refer to Xenomai official documentation). So finalize the kernel configuration this way:
```
make ARCH=arm CROSS_COMPILE=${BUILDROOT}/output/host/usr/bin/arm-buildroot-linux-uclibcgnueabihf- menuconfig
```
```
CPU Power Management
  |- CPU Frequency scaling
       |- CPU Frequency scaling (Unselect)
  |- CPU idle PM support (Unselect)
Kernel hacking
  |- Latency measuring infrastructure (Select)
```

The following is not fundamentally needed but will significantly speed up your boot time ! (from > 10s to < 1s) 
```
Power management options
  |- Suspend to RAM and standby (Unselect)
Device Drivers
  |- Sound card support (Unselect)
  |- USB support (Unselect)
File systems
  |- Second extended fs support (Select)
  |- The Extended 4 (ext4) filesystem (Unselect)
  |- Network File Systems (Unselect)
```

Do not forget to copy your new configuration into the right directory or Buildroot will simply not compile Xenomai !
```
cp .config arch/arm/configs/bcmrpi_quick_xeno_defconfig
```

8- Produce the real-time distribution
=====================================

We are almost ready to test Xenomai, just go back to Buildroot tree and edit one last time the configuration:
```
cd $BUILDROOT
make menuconfig
```
```
Toolchain
  |- Enable WCHAR support (Select) /* Authorizes latencytop to be selected */
Target packages
  |- Debugging, profiling and benchmark
       |- latencytop (Select) /* If this command runs, then you have a functionnal Xenomai */
  |- Real time
       |- Xenomai Userspace (Select)
           |- Custom Xenomai version (2.6.4)
           |- Install testsuite (Select)
           |- Native skin library (Select)
           |- POSIX skin library (Select)
```

Now take your precautions that the kernel will be rebuilt:
```
rm -rf output/build/linux-custom/
```

And finally, run:
```
make
```

You may just copy the new Kernel and deploy the rootfs:
```
cd ${BUILDROOT}/output/images
cp zImage /media/${USER}/boot/
tar -C /media/${USER}/rootfs/ -xvf rootfs.tar
umount /media/${USER}/{boot,rootfs}
```

At boot time you should see lines close to:
```
[    0.459641] I-pipe: head domain Xenomai registered.
[    0.464541] Xenomai: hal/arm started.
[    0.468441] Xenomai: scheduling class idle registered.
[    0.473578] Xenomai: scheduling class rt registered.
[    0.484089] Xenomai: real-time nucleus v2.6.4 (Jumpin' Out) loaded.
[    0.490430] Xenomai: debug mode enabled.
[    0.494806] Xenomai: starting native API services.
[    0.499651] Xenomai: starting POSIX services.
[    0.504163] Xenomai: starting RTDM services.
```

You may check that this node is present:
```
/ # ls -l /dev/rtheap
crw-------    1 root     root       10, 254 Jan  1 00:00 /dev/rtheap
```

And also run latency or latencytop and if I-Pipe is present you should not have any error. But once you have launched it you cannot kill it, so reboot. But we do not care because it is booting in less than one second!

Here we built a very simple distribution with almost no support for current usage. But the system can make hard real-time. Although, if you need support for more drivers/packages, go back to the configuration steps of Linux/Buildroot and add support for whatever feature you need :) If you add Kernel features, do not forget to replace the defconfig file and be sure the kernel will be rebuilt by Buildroot by cleaning first.

9- Compile Xenomai support to you own programs
==============================================

```
cd $XENOMAI
mkdir out-rpi/
PATH="${BUILDROOT}/output/host/usr/bin:$PATH"
./configure --host=arm-buildroot-linux-uclibcgnueabihf CFLAGS='-march=armv6' LDFLAGS='-march=armv6'
make DESTDIR=$(pwd)/out-rpi install
```

10- Test empirically your hardware latency
==========================================

The most straightforward method to test the maximum latency of such an installation is to produce an interruption at a known pace and toggle an LED each time. Before you get familiar with Xenomai API, you may just get this elementary program from Pierre Ficheux's Github (https://github.com/pficheux/raspberry_pi):
```
cd ..
mkdir xenomai_rpi_gpio
cd xenomai_rpi_gpio
wget https://raw.githubusercontent.com/pficheux/raspberry_pi/master/Xenomai/xenomai_rpi_gpio/Makefile
wget https://raw.githubusercontent.com/pficheux/raspberry_pi/master/Xenomai/xenomai_rpi_gpio/xenomai_rpi_gpio.c
```

The tricky make line you can then reuse with your own program follows:
```
PATH="${XENOMAI}/out-rpi/usr/xenomai/bin:$PATH"
DESTDIR=${XENOMAI}/out-rpi/ make XENO=${XENOMAI}/out-rpi/usr/xenomai ARCH=ARM SRC=${XENOMAI}/ksrc
```

If you come back to this chapter later, remind that your PATH shall also contain the path to your Xenomai folder (as it is done in the previous section).

Just copy the created file and you are ready to probe GPIO 25. Add some load with dd, for example, to see how your system reacts. The system load is available with the "top" command after you "mount -t proc /proc proc".



If you discover any mistake in this how-to do not hesitate to drop me an email !
Finally, if needed, the .config from buildroot and the bcmrpi_quick_xeno Linux defconfig file are present at the root of this repository, and the tarballs for the boot and the rootfs partitions are available in the images subdirectory, with the binary of the xenomai_rpi_gpio program.
