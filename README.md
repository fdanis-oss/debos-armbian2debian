# Add new platform to debos: from existing distribution to full debos platform for LePotato or OrangePi 0 Plus 2

The [LePotato](https://libre.computer/products/boards/aml-s905x-cc/) and [OrangePi Zero Plus 2](http://www.orangepi.org/OrangePiZeroPlus2/) boards are already supported by Armbian.<br>
The Armbian distribution is a full image with a lot of tools, it takes more than 1GB, and with specific Armbian's tools, kernel and bootloader.

But how to get a minimal Debian upstream image with only the packages we want? Debos is the perfect tool to do this.

In the first part we will create a minimal Debian image using debos, and reuse the kernel and u-boot from Armbian.<br>
Later, we will see how to remove this Armbian dependency.

The recipes, overlays and scripts used in this blog can be found at [debos-armbian2debian](https://gitlab.collabora.com/fdanis/debos-armbian2debian)

## Install Debos

This is really simple as an official container is provided for it:
```
docker pull godebos/debos
```

## Create Debos recipe using Armbian kernel and u-boot

Start a very simple recipe, minimal debian sid image. As both boards use arm64 architecture this will start in the same manner.
```
architecture: arm64

actions:
  - action: debootstrap
    suite: sid
    components:
      - main
    mirror: https://deb.debian.org/debian
    variant: minbase

  - action: apt
    packages: [ sudo, adduser, systemd-sysv, initramfs-tools, u-boot-tools, u-boot-menu, util-linux ]

  - action: run
    chroot: true
    command: echo debian-sid-arm64 > /etc/hostname

  - action: run
    chroot: true
    command: echo "127.0.1.1\tdebian-sid-arm64" >> /etc/hosts

  - action: overlay
    source: overlays/etc
    destination: /etc

  - action: run
    chroot: true
    script: scripts/setup-user.sh
```
This will install the base of our system and create a default user (login: user, password: user). The _etc_ overlay will add the default user to sudoers.
> with latest debos version, this can be shared by our recipes if we save it separetely, let's call it `debian-minimal.yaml`, and replace all things above in board's recipes by:
> ```
>architecture: arm64
>
>actions:
>  - action: recipe
>    recipe: debian-minimal.yaml
>```

Then, we should add the Armbian repository to be able to install kernel and u-boot for our boards:
```
  - action: overlay
    source: overlays/armbian
```
And install latest kernel and u-boot packages, for LePotato:
```
  - action: apt
    packages: [ linux-image-next-meson64, linux-dtb-next-meson64, linux-u-boot-lepotato-next ]
```
or for OrangePi:
```
  - action: apt
    packages: [ linux-image-next-sunxi64, linux-dtb-next-sunxi64, linux-u-boot-orangepizeroplus2-h5-next ]
```

Create the partitions and deploy our system on it:
```
  - action: image-partition
    imagename: debian-sid-arm64.img
    imagesize: 1GB
    partitiontype: msdos
    mountpoints:
      - mountpoint: /
        partition: root
    partitions:
      - name: root
        fs: ext4
        start: 2MB
        end: 100%
        flags: [ boot ]

  - action: filesystem-deploy
    description: Deploying filesystem onto image
```

U-boot-menu expects FDT directory name to include the kernel version as returned by _linux-version_.<br>
Armbian's kernel package does not provide this directory, so we need to add a link to allow _u-boot-update_ to correctly setup _/boot/extlinux/extlinux.conf_.<br>
For LePotato:
```
  - action: run
    chroot: true
    command: ln -s linux-image-next-meson64 /usr/lib/linux-image-$(linux-version list)
```
or for OrangePi:
```
  - action: run
    chroot: true
    command: ln -s linux-image-next-sunxi64 /usr/lib/linux-image-$(linux-version list)
```

Call _u-boot-update_ to generate the u-boot menu
```
  # Update U-Boot menu after creation of image partitions and filesystem
  # deployment to get correct root information from /etc/fstab
  - action: run
    description: Update U-Boot menu
    chroot: true
    command: u-boot-update
```

Install uboot in our image.<br>
**:warning: Armbian encode a version number in path to u-boot binaries, 5.75 at time I write this blog, this may need to be changed.**

For LePotato:
```
  - action: run
    chroot: false
    command: if=${ROOTDIR}/usr/lib/linux-u-boot-next-lepotato_5.75_arm64/u-boot.bin of=${IMAGE} conv=fsync,notrunc bs=1 count=444

  - action: run
    chroot: false
    command: if=${ROOTDIR}/usr/lib/linux-u-boot-next-lepotato_5.75_arm64/u-boot.bin of=${IMAGE} conv=fsync,notrunc bs=512 skip=1 seek=1
```
or for OrangePi:
```
  - action: raw
    origin: filesystem
    source: /usr/lib/linux-u-boot-next-orangepizeroplus2-h5_5.75_arm64/sunxi-spl.bin
    offset: 8192 # bs=8k seek=1

  - action: raw
    origin: filesystem
    source: /usr/lib/linux-u-boot-next-orangepizeroplus2-h5_5.75_arm64/u-boot.itb
    offset: 40960 # bs=8k seek=5
```

And finally create block map file and compress final image
```
  - action: run
    description: Create block map file
    postprocess: true
    command: bmaptool create debian-sid-arm64.img > debian-sid-arm64.img.bmap

  - action: run
    description: Compressing final image
    postprocess: true
    command: gzip -f debian-sid-arm64.img
```

The final recipe should look like this (for the OrangePi):
```
architecture: arm64

actions:
  - action: debootstrap
    suite: sid
    components:
      - main
    mirror: https://deb.debian.org/debian
    variant: minbase

  - action: apt
    packages: [ sudo, adduser, systemd-sysv, initramfs-tools, u-boot-tools, u-boot-menu, util-linux ]

  - action: run
    chroot: true
    command: echo debian-sid-arm64 > /etc/hostname

  - action: run
    chroot: true
    command: echo "127.0.1.1\tdebian-sid-arm64" >> /etc/hosts

  - action: overlay
    source: overlays/etc
    destination: /etc

  - action: run
    chroot: true
    script: scripts/setup-user.sh

  - action: overlay
    source: overlays/armbian

  - action: apt
    packages: [ linux-image-next-sunxi64, linux-dtb-next-sunxi64, linux-u-boot-orangepizeroplus2-h5-next ]

  - action: image-partition
    imagename: debian-sid-arm64.img
    imagesize: 1GB
    partitiontype: msdos
    mountpoints:
      - mountpoint: /
        partition: root
    partitions:
      - name: root
        fs: ext4
        start: 2MB
        end: 100%
        flags: [ boot ]

  - action: filesystem-deploy
    description: Deploying filesystem onto image

  - action: run
    chroot: true
    command: ln -s linux-image-next-sunxi64 /usr/lib/linux-image-$(linux-version list)

  # Update U-Boot menu after creation of image partitions and filesystem
  # deployment to get correct root information from /etc/fstab
  - action: run
    description: Update U-Boot menu
    chroot: true
    command: u-boot-update

  # Armbian encode a version number in path to u-boot binaries, 5.75 at time I write this blog, this may need to be changed
  - action: raw
    origin: filesystem
    source: /usr/lib/linux-u-boot-next-orangepizeroplus2-h5_5.75_arm64/sunxi-spl.bin
    offset: 8192 # bs=8k seek=1

  - action: raw
    origin: filesystem
    source: /usr/lib/linux-u-boot-next-orangepizeroplus2-h5_5.75_arm64/u-boot.itb
    offset: 40960 # bs=8k seek=5

  - action: run
    description: Create block map file
    postprocess: true
    command: bmaptool create debian-sid-arm64.img > debian-sid-arm64.img.bmap

  - action: run
    description: Compressing final image
    postprocess: true
    command: gzip -f debian-sid-arm64.img
```
Save it to orangepi0p2.yaml.<br>
Recipes, overlays and scripts for both boards can be found at [debos-armbian2debian/1-ArmbianKernel](https://gitlab.collabora.com/fdanis/debos-armbian2debian/tree/master/1-ArmbianKernel)

Last things to do before testing it is to build the image and flash it on a SDCard.
```
$ docker run --rm --interactive --tty --device /dev/kvm --user $(id -u) --workdir /recipes --mount "type=bind,source=$(pwd),destination=/recipes" --security-opt label=disable godebos/debos orangepi0p2.yaml
$ DEV=/dev/your_sd_device
$ sudo bmaptool copy --no-sig-verify --no-verify debian-sid-arm64.img.gz $DEV
```

Those recipes need to be updated on Armbian version change.

## Moving to "full" Debian image

Both boards have upstream support for the kernel and u-boot, so we should be able to replace Armbian packages by Debian one (with some tweaks at least).

### 1st, replace Armbian kernel by Debian's one

This is pretty simple as this just need to replace `linux-image-next-…, linux-dtb-next-…` by `linux-image-arm64` in our recipes. ☺

### 2nd, replace Armbian u-boot by Debian's one

This will be more tricky.<br>
LePotato board already has a specific Debian package for u-boot, but needs some tweak to install it.<br>
OrangePi Zero Plus 2 has u-boot upstream support but not yet its Debian package.

For both boards, as Armbian repository is not needed anymore, we should remove the _overlay_ action installing _overlays/armbian_ in our recipes.<br>
We should also remove the action performing the _link_ hack for u-boot-menu (_ln -s linux-image-next-… /usr/lib/linux-image-$(linux-version list)_).

#### LePotato

Replace `linux-u-boot-lepotato-next` by `u-boot-amlogic` in our recipe.<br>
Remove _run_ actions performing u-boot installation in our recipes.

We will need to retrieve the `u-boot.bin` provided by `u-boot-amlogic` package. Mount the SDCard and copy `usr/lib/u-boot/libretech-cc/u-boot.bin`<br>
Add an action to extract it from the image at the end of our recipe.
```
  - action: run
    description: Extract bootloader u-boot.bin
    chroot: false
    command: cp ${ROOTDIR}/usr/lib/u-boot/libretech-cc/u-boot.bin ${ARTIFACTDIR}/u-boot.bin
```

Recipes, overlays and scripts can be found at [debos-armbian2debian/2-FullDebian](https://gitlab.collabora.com/fdanis/debos-armbian2debian/tree/master/2-FullDebian)

Build and flash the new image to the SDCard.

Now, we should build the vendor specific tools for u-boot, and use them to finish the u-boot installation.

As quoted from README.libretech-cc in u-boot-amlogic package, "Amlogic doesn't provide sources for the firmware and for tools needed to create the bootloader image, so it is necessary to obtain them from the git tree published by the board vendor".<br>
See https://github.com/u-boot/u-boot/blob/master/board/amlogic/p212/README.libretech-cc#L41 to build the bootloader for SDcard (u-boot.bin.sd.bin).

Flash the bootloader to the SDCard.
>```
> DEV=/dev/your_sd_device
> dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=512 skip=1 seek=1
> dd if=fip/u-boot.bin.sd.bin of=$DEV conv=fsync,notrunc bs=1 count=444
>```

#### OrangePi Zero Plus 2

At time I write this, u-boot-sunxi debian package does not include the u-boot binary for OrangePi Zero Plus 2 H5 board.<br>
We will see how to add the board to u-boot-sunxi package and cross-build it before installing it in our image.

> ##### Retrieve and hack U-boot Debian package source
>
> Retrieve latest Debian package source for u-boot:
> ```
> > git clone https://salsa.debian.org/debian/u-boot.git
> > cd u-boot
> ```
>
> Update the Debian targets to add the board to the sunxi package:
> ```
> > echo "arm64    sunxi    orangepi_zero_plus2    u-boot.bin spl/sunxi-spl.bin u-boot-nodtb.bin arch/arm/dts/sun50i-h5-orangepi-zero-plus2.dtb" >> debian/targets
> ```
>
> Add a new entry to debian/changelog:
> ```
> > dch --local test "Enable orangepi_zero_plus2 target in u-boot-sunxi."
> ```
>
> ##### Set-up cross-build for U-boot Debian package
>
> This part is based on schroot creation to build debian packages using sbuild (cf. https://wiki.debian.org/CrossCompiling#Set_up_a_chroot and https://wiki.debian.org/sbuild), and it only needs to be done once.
>
> To create the schroot, do the following:
> ```
> > sudo sbuild-createchroot --make-sbuild-tarball=/srv/chroot/unstable-amd64.tar.gz unstable `mktemp -d` http://deb.debian.org/debian
> > sudo sbuild-adduser $USER
> ```
> Allow access to your home directory from schroot
> ```
> > sudo sed -i s/profile=sbuild/profile=default/ /etc/schroot/chroot.d/unstable-amd64-sbuild-<uniqueID>
> ```
>
> *logout* and *re-login* or use `newgrp sbuild` in your current shell
>
> ##### Build U-boot Debian package
>
> Finally build the new version of u-boot-sunxi (from u-boot source directory):
> ```
> > schroot --chroot unstable-amd64-sbuild --user root --directory $PWD
> # dpkg --add-architecture arm64
> # apt update
> # apt install build-essential crossbuild-essential-arm64 libc6:arm64
> # apt build-dep -aarm64 u-boot
> # CONFIG_SITE=/etc/dpkg-cross/cross-config.arm64 DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -aarm64 -b
> # exit
> ```

Copy the new u-boot-sunxi package to `overlays/u-boot/u-boot-sunxi-test.deb`<br>
Copy the following action before the _apt_ action installing _linux-u-boot-orangepizeroplus2-h5-next_:
```
  - action: overlay
    source: overlays/u-boot
    destination: /root/u-boot
```

Thanks to apt-get we can directly use the bootloader package's path in _apt_ action.<br>
Replace `linux-u-boot-orangepizeroplus2-h5-next` by `arm-trusted-firmware, device-tree-compiler, /root/u-boot/u-boot-sunxi-test.deb` in our recipe.

Replace _run_ action performing u-boot installation in our recipes, by u-boot-sunxi specific one:
```
  - action: run
    chroot: true
    command: TARGET=/usr/lib/u-boot/orangepi_zero_plus2/ /usr/bin/u-boot-install-sunxi64 ${IMAGE}
```

Recipes, overlays and scripts can be found at [debos-armbian2debian/2-FullDebian](https://gitlab.collabora.com/fdanis/debos-armbian2debian/tree/master/2-FullDebian)

Build and flash the new image to the SDCard.
