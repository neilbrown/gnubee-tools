GNUBEE Tools
============

This package contains tools specifically for working with
the GnuBee NAS platform - See www.gnubee.org.

Currently the primary focus is building a firmware
image to boot into the Linux kernel.

A normal boot requires that a root filesystem already exists;
this can be on an SD card, USB storage, or SATA device (including
md-raid or LVM2).
This filesystem must be named "GNUBEE-ROOT" and `findfs` must be able
to find it.  The default config only builds support for xfs and ext4.

This package contains config files and scripts to build a kernel
and an initramfs, in a format that can be written to flash.

If "GNUBEE-ROOT" cannot be found, the initramfs drops into a shell.
Enough tools are available to configure the network, create md arrays,
format an ext4 filesystem, and install Debian over the network.
A script called "config" is available which does much of this for you.
It will not create md arrays, but if you create one manually, it will
find it and offer to install to it.

At the initramfs shell, you can "`cat config`" to see the script or
just type "`config`" to run it.

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

	sudo apt-get install git make gcc bc libssl-dev u-boot-tools unzip

Other packages for the GnuBee that are optional but give more functionality
in the initramfs are:

	sudo apt-get install mdadm dropbear cryptsetup-bin \
		mtd-utils u-boot-tools xfsprogs lvm2 evtest

If you want to build firmware that can be used to install debian, you
also need

	sudo apt-get install debootstrap

Building
--------

To build a firmware image you can

	git clone https://github.com/neilbrown/gnubee-tools.git
	git clone https://github.com/neilbrown/linux.git -b gnubee/v4.4
	cd linux
	../gnubee-tools/scripts/gbmake firmware gbpc1-4.4

If you run this on the GnuBee itself (with Debian installed), no other
configuration is needed.

If you run it on another machine, you will need ssh access to the
GnuBee.  This needs to be configured in `gnubee-tools/config`, which
can be copied from `config.sample`.  In particular, `CROSS_COMPILE`
and `GNUBEE_SSH_HOST` must be set.

The above command will build firmware for a GnuBee PC1.
If you have a PC2, you still need to specify the
gbpc1 as I haven't added config info for the PC2 yet.

I find that the `gbmake` step takes 118 minutes on a GnuBee, and a
little over 4 minutes on my quad-core 16GB RAM desktop.  The `git
clone` of Linux takes roughly forever on the GnuBee due to limited
memory -- consider doing this elsewhere and coping the result over.
Alternately, use

	wget  https://github.com/neilbrown/linux/archive/gnubee/v4.4.zip
	unzip v4.4.zip
	cd linux-gnubee-v4.4
	../gnubee-tools/scripts/gbmake firmware gbpc1-4.4


If you want to just run `gbmake firmware` without the full path, you
can `ln -s` the script to a `bin` directory.  Don't copy it as it
won't work like that.

Note that Linux-5.1 pre-release code is also available - simply
replace '4.4' with '5.1' in the above.  With 5.1 there is the
option of "gbpc2-5.1" which supports the 3.5inch PC2 and includes
support for the 3rd network port.

Installing
----------

The image created is placed in `gnubee-tools/build/gnubee.bin`.  If
you have this on the GnuBee and have `mtd-utils` installed, you can
flash it with `flashcp -v gnubee.bin /dev/mtd3`.  Alternatively you
can copy it to a VFAT filesystem on USB storage, and boot with that
storage plugged in.  This will cause the firmware to be copied to
flash.  When one LED stops flashing and both LEDs are solid-on,
remove the USB storage and reboot into the new firmware.

First Boot
----------

The firmware image is quite large, partly because it contains all
modules - that seems the easiest way to distribute them.  On first
boot you will find `/lib/modules` is a tmpfs filesystem containing all
the modules, and so wasting some of your precious memory.  If you run
`/lib/modules/keep`, the modules will be copied into your root
filesystem.  Subsequent boots will not have /lib/modules mounted.
Note that this isn't needed if you use "`config`" to create your
debian install - that will copy the modules in for you.

When build with a kernel that suppors it, /lib/modules will also
contain a copy of `swconfig` which is used to manage the internal
network switch.  The code in the initramfs will already have
configured this, but it might be helpful to have the binary to play
with.

Install / Rescue mode
---------------------

If the firmware fails to find a filesystem with the label GNUBEE_ROOT,
it will run a shell on the serial console from which you can repair or
create such a filesystem.  If you don't have a serial cable, this
isn't much help.

If you hold the small black button during boot, the firmware will
notice and will not even look for GNUBEE_ROOT but will start a shell
and, importantly, configure the network and start the "dropbear" ssh
daemon.

The first network port (black) is by default configured with address
192.168.10.1.  The second (blue) port is configured to use DHCP to
request an address.  You can use whichever of these is more
convenient.  To login you will need to know the root password which is
"GnuBee".

You can over-ride some defaults by created a VFAT filesystem on a USB
storage device, and placing the file `gnubee-config.txt` on the root
directory.  Then plugging this device in during boot.  The file should
contains "name=value" assignments, one per line.  Following names are
meaningful.

- `CONFIGURE_NET=yes` - this is equivalent to holding the black button
   during boot

- `CONFIGURE_BLACK_IP=xx.xx.xx.xx` - If the network is being
  configure, either due to the button being pressed or due to
  CONFIGURE_NET, the Black network port is configured to the given
  IP address, and the DHCP server is not run.

- `CONSOLE_SHELL=yes` - this is equivalent to not finding GNUBEE-ROOT,
   a console shell is run, but the network is not configured.

Once you are logged in you can modify the network configuration (if,
for example, you want some other static IP, or need to specify a
gateway address) and can create and RAID arrays you want etc.
You can install Debian at this point using debootstrap.

Much of the work for configuring and installing Debian can be
performed using a script called "config".  Simply run "config" and
answer the questions.  You will soon have a minimal Debian
installation which you can boot into.  From there  you can install
anything else that you want.

Cross Compiling
---------------

If you want to compile the kernel on some other computer, you will
likely need a cross compiler.  These are available from various
places, such as described in  https://wiki.debian.org/CrossToolchains.
If you want to compile your own (as I did), here are some steps.

- Install gmp mpfr mpc devel packages
- collect the source code:

        cd /home/git
        git clone git://sourceware.org/git/binutils-gdb.git
        git clone git://gcc.gnu.org/git/gcc.git

- build a recent released version of binutils (I'm using 2.29.1)

        cd binutils-gdb
        git checkout binutils-2_29_1
        mkdir MIPS
        cd MIPS
        ../configure --target=mipsel-unknown-linux-gnu --prefix=/opt/cross
        make
        sudo make install

- build a released gcc (e.g. 7.2.0)

        cd ../../gcc
        git checkout gcc-7_2_0-release
        mkdir MIPS
        cd MIPS
        ../configure --target=mipsel-unknown-linux-gnu --prefix=/opt/cross \
          --enable-languages=c --without-headers \
          --with-gnu-ld --with-gnu-as \
          --disable-shared --disable-threads \
          --disable-libmudflap --disable-libgomp \
          --disable-libssp --disable-libquadmath \
          --disable-libatomic
        make -j
        sudo make install

- Now use `/opt/cross/bin/mipsel-unknown-linux-gnu-` as the
  CROSS_COMPILE setting in your `config` file.
