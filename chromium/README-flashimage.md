---
Author: Stephane Desneux <sdx@iot.bzh>
title: How to flash AGL raw images on SDCard
---
# AGL - Using raw disk images

## Purpose

Here are the commands to run on a Linux host to create a bootable SDcard from a full image file and boot a Renesas R-Car Gen3 board (Starter Kit Pro / M3ULCB).

Requirements:

* bmaptools
* SDcard (>2GB) inserted and available at $DEVICE
* the M3 board is preconfigured to boot on SDCard

TLDR; quick instructions for impatients:

* Grab the raw image (\*.raw.xz) and the associated bmap file (\*.raw.bmap)
* Run:

    ```
    $ sudo bmaptool copy *.raw.xz $DEVICE
    ```

* Eject SDCard, insert in M3 board and turn it on.
* Enjoy!

## Install bmap-tools (recommended)

Bmap-tools is a generic tool for creating the block map (bmap) for a sparse file and copying fioles using the block map. The idea is that large files, like raw system image files, can be copied or flashed a lot faster and more reliably with bmaptool than with traditional tools, like "dd" or "cp".

Bmap-tools sources are available on [github/01org/bmap-tools].
[Full documentation](https://source.tizen.org/documentation/reference/bmaptool) is also available (a bit old, but still relevant).

*Note: Even if Bmap-tools is not strictly required for operation, it's highly recommended. You can still skip this section if you do not wish to install bmap-tools or don't find any package for it*

### RPM-based distribution

Bmap-tools is available as a noarch package here: [bmap-tools-3.3-1.17.1.noarch.rpm](http://iot.bzh/download/public/tools/bmap-tools/bmap-tools-3.3-1.17.1.noarch.rpm)

For example, on Opensuse 42.X:

```
$ zypper in http://iot.bzh/download/public/tools/bmap-tools/bmap-tools-3.3-1.17.1.noarch.rpm
```

### Debian-based distribution

bmap-tool is available in Debian distribution (not tested).

```
$ apt-get install bmap-tools
```

## Download AGL image and bmap file

Download the image and the associated bmap file:

* the raw image (*.raw.xz)
* the bmap file (*.raw.bmap)

## Write a SDcard

1. Insert a SDcard (minimum 2GB)

2. Find the removable device for your card:

    The following commands which lists all removable disks can help to find the information:

    ```
    $ lsblk -dli -o NAME,TYPE,HOTPLUG | grep "disk.*1$"
    sdk  disk       1
    ```

    Here, the device we'll use is /dev/sdk.

    Alternatively, a look at the kernel log will help:

    ```
    $ dmesg | tail -50
    ...
    [710812.225836] sd 18:0:0:0: Attached scsi generic sg12 type 0
    [710812.441406] sd 18:0:0:0: [sdk] 31268864 512-byte logical blocks: (16.0 GB/14.9 GiB)
    [710812.442016] sd 18:0:0:0: [sdk] Write Protect is off
    [710812.442019] sd 18:0:0:0: [sdk] Mode Sense: 03 00 00 00
    [710812.442642] sd 18:0:0:0: [sdk] No Caching mode page found
    [710812.442644] sd 18:0:0:0: [sdk] Assuming drive cache: write through
    [710812.446794]  sdk: sdk1
    [710812.450905] sd 18:0:0:0: [sdk] Attached SCSI removable disk
    ...
    ```

    For the rest of these instructions, we assume that the variable $DEVICE contains the name of the device to write to (/dev/sd* or /dev/mmcblk*). Export the variable:

    ```
    $ export DEVICE=/dev/[replace-by-your-device-name]
    ```

3. If the card is mounted automatically, unmount it through desktop helper or directly wih the command line:

    ```
    $ sudo umount ${DEVICE}*
    ```

4. Write onto SDcard

    Using bmap-tools:

    ```
    $ sudo bmaptool copy *.raw.xz $DEVICE
    bmaptool: info: discovered bmap file 'XXXXXXXXX.raw.bmap'
    bmaptool: info: block map format version 2.0
    bmaptool: info: 524288 blocks of size 4096 (2.0 GiB), mapped 364283 blocks (1.4 GiB or 69.5%)
    bmaptool: info: copying image 'XXXXXXXX.raw.xz' to block device '/dev/sdk' using bmap file 'XXXXXXXX.raw.bmap'
    bmaptool: info: 100% copied
    bmaptool: info: synchronizing '/dev/sdk'
    bmaptool: info: copying time: 4m 26.9s, copying speed 5.3 MiB/sec
    ```

    Using standard dd command (more dangerous):

    ```
    $ xz -cd *.raw.xz | sudo dd of=$DEVICE bs=4M; sync
    ```

## Configure M3 board for boot on SDcard if needed

1. Connect serial console on M3 board and start a terminal emulator on the USB serial port. 
    Here, we use 'screen' on device /dev/ttyUSB0 but you could use any terminal emulator able to open the serial port at 115200 bauds (minicom , ...) 

    ```
    $ screen /dev/ttyUSB0 115200
    ```

2. Power up the board

3. Break at uboot prompt (press any key)

4. Set the following uboot variables:

    **WARNING: don't make a big copy/paste or some garbage characters may be sent to the  console. Instead, copy one or two lines at a time.**

    ```
    setenv board m3ulcb

    setenv set_bootkfile 'setenv bootkfile Image'

    setenv bootkaddr 0x48080000

    setenv set_bootdfile 'setenv bootdfile Image-r8a7796-${board}.dtb'

    setenv bootdaddr 0x48000000

    setenv bootargs_console 'console=ttySC0,115200 ignore_loglevel'

    setenv bootargs_video 'vmalloc=384M video=HDMI-A-1:1920x1080-32@60'

    setenv bootargs_extra 'rw rootfstype=ext4 rootwait rootdelay=2'

    setenv bootargs_root 'root=/dev/mmcblk1p1'

    setenv bootmmc '0:1'

    setenv bootkload_sd 'ext4load mmc ${bootmmc} ${bootkaddr} boot/${bootkfile}'

    setenv bootdload_sd 'ext4load mmc ${bootmmc} ${bootdaddr} boot/${bootdfile}'
    
    setenv bootload_sd 'run set_bootkfile; run bootkload_sd; run set_bootdfile; run bootdload_sd'

    setenv bootcmd 'setenv bootargs ${bootargs_console} ${bootargs_video} ${bootargs_root} ${bootargs_extra}; run bootload_sd; booti ${bootkaddr} - ${bootdaddr}'
    ```

    Then save environment

    ```
    saveenv
    ```

## Boot the board

At uboot prompt, type:

    ```
    run bootcmd
    ```

    Alternatively, simply start the board.

**NOTE: Due to initial operations, first boot can be long (3 to 4 minutes waiting for a timeout on a systemd service). Homescreen may also have problems to start. These issues are known. On next boots, the system should run as expected.**

