USB Armory Debian Jessie image creator
======================================

This script will create an image file containing a fresh, minimal Debian Jessie installation.  This image can be written directly to a microSD card that then can be inserted in your USB Armory, and the USB Armory booted.  This program can run on a Fedora workstation or server, provided that the `debootstrap` program is installed, as well as a few other utilities (see below).

See below for usage instructions.

Usage
-----

As root:

1. Check this project out using Git in a folder on your computer.
2. Create a source directory where build sources will be integrated.
   * At this point, you can optionally provision your own kernel sources, by placing a Linux source tree in a `linux` subdirectory of this source directory.   Make sure those sources correspond to Linux 4.0 or later.
3. Create a build directory where build products will be staged.
4. Run the following command:

    path/to/usbarmory-debian-builder/build-debian-image-for-usbarmory \
        <SOURCEDIR>   \
        <IMAGEFILE>   \
        <BUILDDIR>    \
        <DISTRO>      \
        host | device \
        [ path to SSH public key file ]

The parameters are:

* `SOURCEDIR`: the source directory you created.
* `IMAGEFILE`: the resulting image file.
* `BUILDDIR`: the build directory you created.
* `DISTRO`: this ought to be `jessie`
* `host` or `device`: the initial mode for the system.
   * Host mode will boot in a mode that is compatible with the USB converter, DisplayLink devices, USB network devices, and USB HID (keyboard / mouse) devices.
   * Device mode will boot in a mode that is compatible with operation as a USB device connected to a computer.  In this mode, you can SSH into the device as root by (1) setting up a static IP address `10.0.0.2/255.255.255.0` on the USB adapter that shows up on your computer after plugging in, (2) SSHing to `root@10.0.0.1`.
* path to SSH public key file: if specified (this is optional), then the contents of this file will be embedded into the `root` user SSH authorized keys database, thereby ensuring that only the possessor of the corresponding private key can SSH into the USB Armory.  If this file is not specified, then the `root` user password will be blank and you will be able to SSH into the device directly.

After setup is done, you can use `dd` to transfer the image(s) to the appropriate media for booting.  See below for examples and more information.

Requirements
------------

These are the programs you need to execute this script:

* debootstrap
* GnuPG (`gpg`)
* GNU wget
* rsync
* parted
* sha256sum
* tar
* xz

Transferring the images to media
--------------------------------

You can transfer the resulting disk image to your USB Armory's microSD card.  The usual `dd if=/path/to/built/image of=/dev/path/to/disk/device oflag=sync conv=sparse bs=32M` advice works fine.  Here is an example showing how to write an image file that was just created to `/dev/sde`:

    dd if=/path/to/image/file of=/dev/sde oflag=sync conv=sparse bs=32M

Taking advantage of increased disk space in the target media
------------------------------------------------------------

Once you have booted your USB Armory device and successfully logged into it, you can enlarge the root partition by:

1. Using `fdisk` (see online instructions on the Web on how to do that) to enlarge the partition `/dev/mmcblk0p1`.
2. Rebooting the device.
3. Using `resize2fs` on the block device `/dev/mmcblk0p1`.

License
-------

GNU GPL v3.
