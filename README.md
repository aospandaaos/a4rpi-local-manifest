# Building Android for Raspberry Pi 4B

# General notes
This is project experimental; it is not guaranteed to build on all possible
systems and it is not guaranteed to run on all Raspberry Pi 4 configurations.
You need a fair amount of knowledge of Linux and Android to get this to work.
Be warned.

My build system:
 - Ubuntu 20.04

My Raspberry Pi configuration:
 - Raspberry Pi 4B 4GB
 - 7" HDMI touch screen 1024x600. There are many available. The one I use
   is from Elecrow


## Preparation

Make sure that you have a system capable of building AOSP in a reasonable
amount of time, as described here:
[https://source.android.com/source/building.html](https://source.android.com/source/building.html).

Then set it up following the steps given here:
[https://source.android.com/source/initializing.html](https://source.android.com/source/initializing.html)

Then install these extra packages
```
$ sudo apt install bmap-tools
```

Choose a directory for the project, e.g. $HOME/rpi. The structure is going to be

```
rpi
  |-- u-boot
  |-- android-kernel-rpi-5.15
  |-- aosp
```

## U-Boot
We need a toolchain to build U-Boot, for example from Bootlin.
Go to [https://toolchains.bootlin.com/](https://toolchains.bootlin.com/)

Select arch=aarch64 and libc=glibc, then "Download stable"

Set the shell to find that toolchain:
```

$ PATH=$HOME/aarch64--glibc--stable-2022.08-1/bin:$PATH
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-
```

Download U-Boot:
```
$ cd rpi
$ git clone https://gitlab.denx.de/u-boot/u-boot -b v2022.10
```

Build U-Boot:
```
$ cd rpi/u-boot
$ make rpi_4_defconfig
$ make
```

Check
```
$ ls -l u-boot/u-boot.bin
-rwxrwxr-x 1 chris chris 628136 Jan  5 17:30 u-boot/u-boot.bin
```

## Raspberry Pi firmware
Get the official Raspberry Pi firmware (about 27GB !):
```
$ cd rpi
$ git clone https://github.com/raspberrypi/firmware.git
```

## Kernel
Download the kernel (about 19 GB):
```
$ cd rpi
$ mkdir android-kernel-rpi-5.15
$ cd android-kernel-rpi-5.15
$ repo init -u https://github.com/android-rpi/kernel_manifest -b arpi-5.15
$ repo sync
```

Build it:
```
$ build/build.sh
```

Check (your file sizes may be slightly different to mine)
```
$ ls -l out/arpi-5.15/dist
total 26888
-rw-rw-r-- 1 chris chris       22 Dec 28 15:33 abi.prop
-rw-rw-r-- 1 chris chris    52457 Dec 28 15:33 bcm2711-rpi-400.dtb
-rw-rw-r-- 1 chris chris    52325 Dec 28 15:33 bcm2711-rpi-4-b.dtb
-rw-rw-r-- 1 chris chris       42 Dec 28 15:35 gki_aarch64_modules
-rw-rw-r-- 1 chris chris 10770878 Dec 28 15:35 Image.gz
-rw-rw-r-- 1 chris chris 15072266 Dec 28 15:35 kernel-headers.tar.gz
-rw-rw-r-- 1 chris chris  1561436 Dec 28 15:35 kernel-uapi-headers.tar.gz
-rw-rw-r-- 1 chris chris      572 Dec 28 15:35 test_mappings.zip
-rw-rw-r-- 1 chris chris     3914 Dec 28 15:33 vc4-kms-v3d-pi4.dtbo
```

## Download AOSP

Select the version of AOSP (tested using android-13.0.0_r15) 
```
$ mkdir rpi/aosp
$ cd rpi/aosp
$ repo init -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r15
```

If you want to reduce the size of the download and you don't care too much
about the git history you can do a shallow clone by setting the depth to 1:
```
$ repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r15
```

Add the A4Rpi repositories:
```
$ git clone https://github.com/aospandaaos/a4rpi-local-manifest .repo/local_manifests -b android13
```

Then sync everything (downloads 130 GB data, or 70 GB if you did a shallow clone)
```
$ repo sync -c
```


## Build AOSP

Start a *new shell*

```
$ source build/envsetup.sh
$ lunch
```
Select either rpi4_tablet-userdebug or rpi4_auto-userdebug

Start the build
```
$ m
```

The command 'm' is a wrapper for 'make', with the additional benefit that it
will use all available CPU cores

Even so, the build will take an hour or two...


## Write to SD card

You will need a micro SD card of at least 8 GB. Ideally it should
be class 10 or better.

Plug the card into your card reader. Run command `lsblk` to find which
device it is. Then run the script below, giving the device name as the
parameter. For example, if the card reader is `/dev/mmcblk0`, the
command would be:
```
$ cd rpi/aosp
$ scripts/write-sdcard-rpi4.sh /dev/mmcblk0
```
Note: I use bmap-tool to write the card because it is faster and
safer than dd. If you don't want to use bmap-tool, go ahead and edit the script
to switch back to dd.

When done, plug the card into your Raspberry Pi and boot up


## ADB

To connect ADB via Ethernet, plug your Raspberry Pi into your Ethernet
LAN and wait for it to be given an IP address. Then from $ANDROID_BUILD_TOP type
this command:

```
$ adb connect Android.local:5555
```

