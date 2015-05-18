USB Armory Debian Jessie image creator
======================================

This script will create an image file containing a fresh, minimal Debian Jessie installation.  The built image can be written directly to a microSD card that then can be inserted in your USB Armory, and the USB Armory booted.  This program can run on a Fedora workstation or server, provided that the `debootstrap` program is installed, as well as a few other utilities (see below).  An image built by this program is mostly compatible with the instructions presented by the USB Armory wiki.

See below for usage instructions.

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
* bzip2

Usage
-----

As root:

1. Check this project out using Git in a folder on your computer.
2. Create a source directory where build sources will be integrated.
   * At this point, you can optionally provision your own kernel sources, by placing a Linux source tree in a `linux` subdirectory of this source directory.   Make sure those sources correspond to Linux 4.0 or later.
   * The same applies to U-Boot -- you can provision your U-Boot sources in a subdirectory `u-boot` within the source directory.  Make sure those sources correspond to U-Boot 2015.04 or later.
3. Create a build directory where build products will be staged.
4. Run the following command:

```
    path/to/usbarmory-debian-builder/build-debian-image-for-usbarmory \
        <SOURCEDIR>   \
        <IMAGEFILE>   \
        <BUILDDIR>    \
        <DISTRO>      \
        host | device \
        [ path to SSH public key file ]
```

The parameters are:

* `SOURCEDIR`: the source directory you created.
* `IMAGEFILE`: the resulting image file.
* `BUILDDIR`: the build directory you created.
* `DISTRO`: this ought to be `jessie`
* `host` or `device`: the initial mode for the system.
   * Host mode will boot in a mode that is compatible with the USB converter, DisplayLink devices, USB network devices, and USB HID (keyboard / mouse) devices.
   * Device mode will boot in a mode that is compatible with operation as a USB device connected to a computer.
* path to SSH public key file: if specified (this is optional), then the contents of this file will be embedded into the `root` user SSH authorized keys database, thereby ensuring that only the possessor of the corresponding private key can SSH into the USB Armory.  If this file is not specified, then the `root` user password will be blank and you will be able to SSH into the device directly.

There are a number of overridable environment variables at the top of the script.  You can prefix the command line that runs the script with variable definitions of the form `variable=value` to override those values, thus avoiding having to modify the script in order to change those values.  Always make sure to run this program in a sanitized environment so that variables injected by potential attackers cannot modify the behavior of this program.

The program will download the necessary sources from the Internet (unless you have provided those sources manually) and then proceed to verify that the signatures of the downloaded sources are valid.  At no point is unverified software installed or executed in your computer.

After setup is done, you can use `dd` to transfer the image(s) to the appropriate media for booting (which must be minimum 4 GB in size).  See below for examples and more information.

Transferring the images to media
--------------------------------

You can transfer the resulting disk image to your USB Armory's microSD card.  The usual `dd if=/path/to/built/image of=/dev/path/to/disk/device oflag=sync conv=sparse bs=32M` advice works fine.  Here is an example showing how to write an image file that was just created to `/dev/sde`:

    dd if=/path/to/image/file of=/dev/sde oflag=sync conv=sparse bs=32M

Booting and accessing the image for the first time
--------------------------------------------------

You should know that:

During the first boot, `systemd-journald` will fail to start because there will not be an `/etc/machine-id` file which the journal expects to find in the root volume.  This should correct itself after the first boot is complete, once the service `systemd-machine-id-setup.service` completes its first and only run.

SSH host keys will also be absent during the very first phases of booting, because the keys need to be created upon first boot.  This is done by the `ssh-gen-host-keys.service` service, which starts right before the OpenSSH server starts itself.

To access the device, it depends on which mode you selected for the first boot during the build of the image.

In host mode: plug it into a USB hub using the female-to-female adapter, then plug to the hub a DisplayLink monitor and a keyboard.  Optionally, plug a network card into the USB hub as well.  Then plug the power socket on the adapter, and watch the device boot onscreen.  If you did not specify an SSH public key to access the device when you built the image, you can now log in as root (the device will be password-free).  Network connectivity using USB network adapters can be accomplished by using `/etc/network/interfaces` configuration stanzas (with DNS defined in `/etc/resolv.conf`).  Here is an example configuration file for a hypothetical USB network device thaty you can suit to taste and drop in `/etc/network/interfaces.d/eth0`:

```
    auto eth0
    allow-hotplug eth0
    iface eth0 inet static
      hwaddress aa:bb:cc:dd:ee:ff
      address 1.2.3.4
      netmask 255.255.255.0
      gateway 1.2.3.5
```

In device mode: plug it to your host computer.  Set up a static IP address `10.0.0.2/255.255.255.0` on the USB adapter that shows up on your computer after plugging in (`ip link` will show it).   Simply `ssh root@10.0.0.1` into the device.  If you did not an SSH public key to access the device when you built the image, this will log you in on the spot.  if you did not, you must have the private key corresponding to that public key in order to log in.  More (generic) information on how to connect to the device is available on the [Host communication page of the USB Armory wiki](https://github.com/inversepath/usbarmory/wiki/Host-communication) -- specifically in the sections titled *Setup & Connection Sharing: Linux / Mac OS X* (the other sections are not relevant to this image).

The image of the device has no locales set up.  To set up a basic UTF-8 U.S. English locale, you can run as `root` on the device:

```
    echo en_US.UTF-8 UTF-8 >> locale.gen
    locale-gen
    update-locale LANG=en_US.UTF-8
```

Taking advantage of increased disk space in the target media
------------------------------------------------------------

Once you have booted your USB Armory device and successfully logged into it, you can enlarge the root partition by:

1. Using `fdisk` (see online instructions on the Web on how to do that) to enlarge the partition `/dev/mmcblk0p1`.
2. Rebooting the device.
3. Using `resize2fs` on the block device `/dev/mmcblk0p1`.

Alternatively, you can create another partition on the trailing unallocated space of your MMC block device, and mount them through `/etc/fstab` or expose them to the host operating system (see below).  Remember to reboot after creating new partitions.

Switching between host mode and device mode
-------------------------------------------

The image ships with device tree files for host mode (makes the device into a regular computer with an USB host) and for device mode (makes the device appear as a set of USB peripherals when plugged into a regular computer).

To switch to host mode when in device mode, while logged in as `root`, run the command `/usr/local/bin/hostmode`.  Then reboot the device.  This makes the device able to plug into a USB hub with its USB female-to-female adapter, and you can plug monitors, network devices, mice and keyboards to your USB hub.

To switch to device mode when in host mode, while logged in as `root`, run the command `/usr/local/bin/devicemode`.  Then reboot the device.  This makes the device appear as a combination serial device, network device and USB mass storage device.

Taking advantage of USB mass storage emulation in device mode
-------------------------------------------------------------

When logged into the USB Armory running in device mode, you can expose a block device to the host computer by running the following code:

```
    f=$(find /sys | grep gadget/lun0/file | head -1)
    echo /dev/path/to/block/device > "$f"
```

This causes said block device to be "plugged into" your host computer.  Echoing the empty string to that control file causes the block device to be "unplugged from" your host computer.

Under no circumstances should you echo the path of the root volume block device to that sysfs control file, because that will cause corruption of your USB Armory's root volume.

This feature works thanks to the magic of `g_multi.service`, which detects device mode and loads the `g_multi` kernel module, with the appropriate parameters to make USB mass storage work.

Troubleshooting using the console
---------------------------------

An USB to TTL cable, hooked to the appropriate header pinouts on the board, will show you the device's console as it boots.  Run `minicom -D /dev/ttyUSB0` on your host computer after plugging the USB to TTL cable to your host computer.  Then you can connect the USB Armory to the cable's pins.  You'll see the console start in a few seconds.

The pinout for your USB Armory is detailed on page https://github.com/inversepath/usbarmory/wiki/GPIOs .  A breakdown of those pins:

* Pin 1 (squarish looking) goes to ground.
* Pin 2 goes to USB +5V
* Pin 5 goes to the receive pin of your TTL cable.
* Pin 6 goes to the transmit pin of your TTL cable.

(If these summarized pinouts are wrong, please send a pull request with the corrections.)

License
-------

GNU GPL v3.
