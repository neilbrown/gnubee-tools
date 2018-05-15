GNUBEE Tools
============

This package contains tools specifically for working with
the GnuBee NAS platform - See www.gnubee.org.

Currently the primary focus is building a firmware
image to boot into the Linux kernel.

It is assumed that an ext4 root filesystem already exists; this can be
on an SD card, USB storage, or SATA device (including md-raid).
This filesystem must be named "GNUBEE-ROOT" and `findfs` must be able
to find it.

This package contains config files and scripts to build a kernel
and an initramfs, in a format that can be written to flash.

Preparation
-----------

To build the firmware, you will need a GnuBee running Debian.
Various tools for the initramfs are simply copied from the running
system rather than being built separately: lazy, but fast.

On the GnuBee you will need various packages installed.  To
install them run:

	sudo apt-get install make gcc busybox libnl-genl-3-dev

On whichever machine you want to build the kernel (which could be a
GnuBee) you will need:

	sudo apt-get install git make gcc bc libssl-dev	u-boot-tools

Other packages for the GnuBee that are optional but give more functionality
in the initramfs are:

	sudo apt-get install mdadm dropbear cryptsetup-bin \
		mtd-utils u-boot-tools


Building
--------

To build a firmware image you can

	git clone https://github.com/neilbrown/gnubee-tools.git
	git clone https://github.com/neilbrown/linux.git -b gnubee/v4.4
	cd linux
	../gnubee-tools/scripts/gbmake firmware gbpc1-900-4.4

If you run this on the GnuBee itself (with Debian installed), no other
configuration is needed.

If you run it on another machine, you will need ssh access to the
GnuBee.  This needs to be configured in `gnubee-tools/config`, which
can be copied from `config.sample`.  In particular, `CROSS_COMPILE`
and `GNUBEE_SSH_HOST` must be set.

The above command will build firmware for a GnuBee running at 900MHz.
If yours runs a 880MHZ you can ask for `gbpc1-880-4.4`.  If you build
on the GnuBee itself you can leave the model information out and
`gbmake` will guess.  If you have a PC2, you still need to specify the
gbpc1 as I haven't added config info for the PC2 yet.

I find that the `gbmake` step takes 104 minutes on a GnuBee, and a
little over 4 minutes on my quad-core 16GB RAM desktop.  The `git
clone` of Linux takes roughly forever on the GnuBee due to limited
memory -- consider doing this elsewhere and coping the result over.

If you want to just run `gbmake firmware` without the full path, you
can `ln -s` the script to a `bin` directory.  Don't copy it as it
won't work like that.

Installing
----------

The image created is placed in `gnubee-tools/build/gnubee.bin`.  If
you have this on the GnuBee and have `mtd-utils` installed, you can
flash it with `flashcp -v gnubee.bin /dev/mtd3`.  Alternatively you
can copy it to a VFAT filesystem on USB storage, and boot with that
storage plugged in.

First Boot
----------

The firmware image is quite large, partly because it contains all
modules - that seems the easiest way to distribute them.  On first
boot you will find `/lib/modules` is a tmpfs filesystem containing all
the modules, and so wasting some of your precious memory.  If you run
`/lib/modules/keep`, the modules will be copied into your root
filesystem.  Subsequent boots will not have /lib/modules mounted.

/lib/modules will also contain a copy of `swconfig` which is used to
manage the internal network switch.  The code in the initramfs will
already have configured this, but it might be helpful to have the
binary to play with.
