# LFS 11.1 systemd with EFI and pacman

### Installing LFS systemd edition with EFI support and maintain it with pacman package manager ###

Based on [Linux From Scratch](https://www.linuxfromscratch.org/lfs/view/stable-systemd/index.html)                                                           
Based on [the guide writen by James Kimball](http://lists.linuxfromscratch.org/pipermail/hints/2013-March/003304.html) in 2013                               
Based on [Pacman on LFS 8.1](https://github.com/benvd/lfs-pacman)" By Ben Van Daele                                                                           
Licensed under the [Creative Commons Attribution-ShareAlike 3.0 License](https://creativecommons.org/licenses/by-sa/3.0/)

### FAQ ### 

- **Q:** What is this? Another arch-based distro?
- **A:** Nope. This is more LFS than arch-based clone. But it's using pacman packages manager developed by arch linux community.
- **Q:** But LFS never used any package manager, it's not LFS philosophy.
- **A:** True, but this is not pure LFS. It's using most of LFS instruction to build software and put them in pacman's format packages.
- **Q:** Why then using package manager at all if you can compile and install all manualy?
- **A:** When you building LFS, you might found what you did mistake, for example installed software in wrong `prefix` or used `lib64` instead of `lib` or did another mistake. Fixing this could be problem, because you need find all installed files and remove them. Using packages, could help you track installed files and revert changes back if needed.
- **Q:** Dependensies, repo and etc for pacman packages?
- **A:** Nope. No dependensies. Packages what you will produce will only contating pure LFS compiled programs and you need resolve nesesary dependisies for
missing libraries and headers yourself. They will be stored localy on you PC in separate folder not on server.
- **Q:** But is it possible to made my own repo for my own packages in case if I need reinstall stuff fast?
- **A:** Yes, it's possible but not covered here. You need read additional info using arch wiki.
- **Q:** Why sytemd  and EFI?
- **A:** Systemd is standard for most linux distros now. If you want use old init scripts you must do it yourself. EFI used in 99% modern PC now, if you still have legacy BIOS you need install grub and again do it yourself.
- **Q:** Version 11.1? But LFS almost released 11.2 now.
- **A:** I'll update this guide later when stable LFS 11.2 will be released.

## Introduction

This guide is divided in few stages:

- Stage 1 - Almost same as LFS book. You will only need download additional archives required for building pacman, EFI support in systemd and initramfs.
- Stage 2 - Similar to LFS book, when you build cross toolchain and cross compile temporary tools for your system where you later chroot in to it. 
- Stage 3 - Chrooting to your system, building more temporary stuff and installing temporary pacman there. Almost same as LFS book with few small changes.
- Stage 4 - Installing part of basic system software, same as LFS book, but using PKGBUILD files to produce pacman's packages to track it.
- Stage 5 - Reinstalling pacman as basic software and putting it in pacman's package too.
- Stage 6 - Finishing building basic software. Installing support for EFI in systemd. Installing systemd package. Installing packages for building initramfs             Making systemd bootable with systemd-boot and finishing the book.

### Stage 1

Follow the book instructions until you reach chapter **3.1. Introduction**. When you finishing downloading LFS packages using:

````
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources
````

If desired do md5 checksums:

````
pushd $LFS/sources
  md5sum -c md5sums
popd
````

Get additional archives and place them to you `$LFS/osurces` folder too :

- libarchive: `wget https://github.com/libarchive/libarchive/releases/download/v3.6.0/libarchive-3.6.0.tar.xz --directory-prefix=$LFS/sources`
- fakeroot: `wget https://deb.debian.org/debian/pool/main/f/fakeroot/fakeroot_1.29.orig.tar.gz --directory-prefix=$LFS/sources`
- pacman: `wget https://sources.archlinux.org/other/pacman/pacman-6.0.1.tar.xz --directory-prefix=$LFS/sources`
- gnu-efi: `wget https://download.sourceforge.net/gnu-efi/gnu-efi-3.0.14.tar.bz2 --directory-prefix=$LFS/sources`
- cpio:  `wget https://ftp.gnu.org/gnu/cpio/cpio-2.13.tar.bz2 --directory-prefix=$LFS/sources`
- dracut: `wget -O dracut-057.tar.gz https://github.com/dracutdevs/dracut/archive/refs/tags/057.tar.gz --directory-prefix=$LFS/sources`

I'm not providing here md5 sums for them, if you want to check use other download sources and do it yourself. This shouldn't be big issue. 

**Note:** You may skip download patches for LFS files and additional archives because they all present in [packages](https://github.com/ShiroiBara/LFS-11.1-systemd-with-EFI-and-pacman/tree/main/packages) folder with `PKGBUILD` files.

Download `packages` folder from this repo, using `git clone` or as `zip` file and unpack it to `$LFS/sources` folder. No additional instruction here, please
google how to use `git` if needed. This is guide not covering any aspect of using linux, if you tried pure `LFS` early you probably have some linux knowledge. Nothing personal.

**Note:** EFI boot require separate partition formated to `FAT32` file system. Keep it in mind when reading chapters **2.4. Creating a New Partition** and **2.5. Creating a File System on the Partition**. Assuming what you are using GPT layout on your disk.

Proceed next now, using LFS book until you reach  **iii. General Compilation Instructions** 

### Stage 2

As it was said, this is just same as LFS chapters from **5. Compiling a Cross-Toolchain** to **6.18. GCC-11.2.0 - Pass 2**. Follow LFS book instructions.

### Stage 3

Follow LFS book chapters from **7. Entering Chroot and Building Additional Temporary Tools** to **7.10. Perl-5.34.0**. Now you need break, read some info and follow this guide.

#### pacman and makepkg dependencies

To build latest pacman and use makepkg for creating you own packages you will need to install those libraries and tools:

- zlib
- opensll
- python with zlib module for .pyz files
- ninja
- meson as .pyz file
- util-linux for getopt
- libcap
- su from shadow
- libarchive
- pkg-config
- fakeroot

Build these packages using the following commands. Just like the LFS book, these commands assume you have extracted the relevant sources and `cd`'d into the resulting directory.

#### zlib 1.2.11

```
./configure --prefix=/usr
make
make install
rm -fv /usr/lib/libz.a
```

#### openssl 3.0.1

```
./config --prefix=/usr         \
         --openssldir=/etc/ssl \
         --libdir=lib          \
         shared                \
         zlib-dynamic
make         
sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile
make MANSUFFIX=ssl install
mv -v /usr/share/doc/openssl /usr/share/doc/openssl-3.0.1
```

#### python 3.10.2

```
./configure --prefix=/usr   \
            --enable-shared \
            --without-ensurepip
make
make install
```

**Note:** This is temporary python builded with same commands as LFS book chapter **7.11. Python-3.10.2**. All messages about missing libararies and
headers could be safely ignored because we only need zlib module. Even openssl has already build, it's not require for this version of python, if desired
you could build openssl after python.

#### ninja 1.10.2

```
python3 configure.py --bootstrap
install -vm755 ninja /usr/bin/
install -vDm644 misc/bash-completion /usr/share/bash-completion/completions/ninja
install -vDm644 misc/zsh-completion  /usr/share/zsh/site-functions/_ninja
```

#### meson 0.61

**Note:** Since python was build without setuptools module we need put meson in single executable ziped file what we store in /source directory. 
This file will be used by python later to build pacman

```
./packaging/create_zipapp.py --outfile meson.pyz --interpreter '/usr/bin/python3' .
cp meson.pyz ..
```

#### util-linux 2.37.4

```
mkdir -pv /var/lib/hwclock
./configure ADJTIME_PATH=/var/lib/hwclock/adjtime    \
            --libdir=/usr/lib    \
            --docdir=/usr/share/doc/util-linux-2.37.4 \
            --disable-chfn-chsh  \
            --disable-login      \
            --disable-nologin    \
            --disable-su         \
            --disable-setpriv    \
            --disable-runuser    \
            --disable-pylibmount \
            --disable-static     \
            --without-python     \
            runstatedir=/run
make
make install
```

#### libcap 2.63

```
sed -i '/install -m.*STA/d' libcap/Makefile
make prefix=/usr lib=lib
make prefix=/usr lib=lib install
```

#### shadow 4.11.1

```
sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /'   {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /'   {} \;
./configure --sysconfdir=/etc \
            --disable-static
make
cp ./src/su /usr/bin/
```

#### libarchive 3.6.0

```
sed -i '436a if ((OSSL_PROVIDER_load(NULL, "legacy")) == NULL) \
      return (ARCHIVE_FAILED);' libarchive/archive_digest.c
./configure --prefix=/usr --disable-static
make
make install
```

#### pkg-config 0.29.2

```
./configure --prefix=/usr              \
            --with-internal-glib       \
            --disable-host-tool        \
            --docdir=/usr/share/doc/pkg-config-0.29.2
make
make install
```

#### fakeroot 1.29

```
./configure --prefix=/usr                   \
            --libdir=/usr/lib/libfakeroot   \
            --disable-static                \
            --with-ipc=sysv
make
make install
```

#### pacman 6.0.1

```
cp ../meson.pyz .
python3 meson.pyz --prefix=/usr                   \
                  --buildtype=plain               \
                  -Ddoc=disabled                  \
                  -Ddoxygen=enabled               \
                  -Dscriptlet-shell=/usr/bin/bash \
                  -Dldconfig=/usr/bin/ldconfig    \
build
python3 meson.pyz compile -C build
python3 meson.pyz install -C build
rm ../meson.pyz
```

This will have installed, amongst others, the `makepkg.conf` and `pacman.conf` config files in `/etc`; you may want to edit them. For `makepkg.conf`, be sure that `CARCH` and `CHOST` are appropriate, e.g.:

```
CARCH="x86_64"
CHOST="x86_64-pc-linux-gnu"
```

For deafult makepkg using options such striping debug symbols keeping static libraries empty directories and compress man files. Also it removes some info files like .pod. Some of those change not compatible with lfs and should be changed. Edit lines in make.conf file to achive it:

```
OPTIONS=(!strip !libtool !staticlibs !zipman purge)
PURGE_TARGETS=(usr/{,share}/info/dir .packlist)
```

If you desire you can change them for you own use. Refer to makepkg.conf [manual.](https://man.archlinux.org/man/makepkg.conf.5.en)

You can set your name and email address as the `PACKAGER` if you want.

#### Setting up to build packages with pacman using makepkg

Makepkg not allow build packages under root user. LFS book using user **tester** for test when building basic system software in chapter 8. This user could be used to envoke makepkg in chroot environment:

```
su tester -c "PATH=$PATH LANG=$LANG makepkg <options>"
```

Now you need finish building temporary tools with `text-info` package, refer to **7.12. Texinfo-6.8** of book.

### Stage 4

#### Clean up temporary system acording book, remove some files and move missplaced directories:

````
rm -rf /usr/share/{info,man,doc}/*
find /usr/{lib,libexec} -name \*.la -delete
rm -rf /tools
find /usr -name \*.packlist -delete
install -vdm755 /usr/share/gdb/auto-load/usr/lib
mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib
````

#### Change to directory for building packages

````
cd /source/packages
````
Before you start build packages, using makepkg command, you will need `PKGBUILD` files and some patches. They all present inside subfolders in `/source/packages` directory

Change to package directory. Our first package will be linux-api-headers. Even it was installed manually we need reinstall it with pacman for future easy removal or replacement in case of upgrading system.

````
cd linux-api-headers
cp ../../sources linux-5.16.9.tar.xz .
su tester -c "PATH=$PATH LANG=$LANG makepkg -c"
mv linux-api-headers-5.16.9-1-x86_64.pkg.tar.gz ..
cd ..
pacman -U linux-api-headers-5.16.9-1-x86_64.pkg.tar.gz --overwrite=*
````

For second and rest packages follow book order. Instructions for building will stay same. First you need to change to package directory and copy package tarball file, include patches if they needed. Next you envoke `makepkg` as example above, move builded pacman packages to `/package` directory and install it to final system.

Build next three packages:

- Man-pages-5.13
- Iana-Etc-20220207
- Glibc-2.35 

using instruction above.

**Note:** You need add time zone data, or some packages later will not build corectly. 

LFS book using a lot of commands to do that, but instead of it just use PKGBUILD file build and install `tzdata` package.

Set you time zone, for example:

````
ln -sfv /usr/share/zoneinfo/Canada/Eastern /etc/localtime

````

The dynamic loader will be configured automaticly according commands present in `glibc` `PKGBUILD` file.

**Note:** Setup `LANG` variable acording your system locale. It will be later used while building `GCC`. Incorect setup of it will produce building error.

For example for US issue this:

````
LANG=en_US.UTF-8
````

Now you can contnue build system until you finish chapter **8.34. Bash-5.1.16**

**Note:** When you build and install `bash` with pacman you will use `exec /usr/bin/bash --login` command to run compiled bash. This also reset `LANG` variable. You must set it back:

````
LANG=en_US.UTF-8
````

Continue building basic software until you finish chapter **8.58. Groff-1.22.4**

## Stage 5

It's time now to reinstall pacman as pacman's package. But before we need build and reinstall as pacman's packages two libraries:

- libarchive
- fakeroot

Build them using `PKGBUILD` files and install using `pacman -U <package name> --overwrite=*` command.
Backup yours `.config` files, because reinstalling pacman will overwrite them:

````
cp /etc/pacman.conf /etc/pacman.conf-back
cp /etc/makepkg.conf /etc/makepkg.conf-back
````

Finally you can build pacman as it's own package and intall it as basic software. After finishing, don't forget restore your config files:

````
su tester -c "PATH=$PATH LANG=$LANG makepkg -c"
pacman -U pacman-6.0.1-1-x86_64.pkg.tar.gz --overwrite=*
````

And restore `.config` files back:

````
mv /etc/pacman.conf-back /etc/pacman.conf
mv /etc/makepkg.conf-back /etc/makepkg.conf
````

## Stage 6

Skip building **chapter 8.59. GRUB-2.06**. Since system will be booted in EFI mode, instead of `grub` bootloader we will use `systemd-boot`. Continue build pakages from **chapter 8.60. Gzip-1.11**, using `PKBUILD` files and install them to your final system until you reach **chapter 8.70. Jinja2-3.0.3**.
Now you need build additional package `gnu-efi` which adds EFI support for using to systemd. Build and install it using correspondent `PKGBUILD` file.
When you finish building and installing `gnu-efi` package you can build `systemd` package. The `PKGBUILD` has some additional lines to satisfy EFI support
for `systemd` and use later `systemd-boot` command to install bootloader:

```
-Dgnu-efi=true          \
-Dsbat-distro='lfs'     \
-Dsbat-distro-summary="Linux From Scratch"  \
-Dsbat-distro-url="https://linuxfromscratch.org"
```

Build and install `systemd` package. Continue from chapter **8.72. D-Bus-1.12.20** to chapter **8.78. Stripping**. Skip chapter **8.79. Cleaning Up**
User **tester** still needed for package building, so keep him. You only need remove partial compiler:

````
find /usr -depth -name $(uname -m)-lfs-linux-gnu\* | xargs rm -rf
````

Build and install packages for creating `ititramfs`:

- cpio
- dracut

Mount you EFI partition if you have not did it early:

````
mount /dev/xxx /boot
````
 
Where `xxx` your EFI partition name. For example sda1 for primary hard disk. Proceed to chapter **9. System Configuration** until you reach chapter     **10.3. Linux-5.16.9**.

#### Linux firmware ####

Most of modern PC require additional firmware in order to boot kernel and use computer devices. The great article about could be found in BLFS [book](https://www.linuxfromscratch.org/blfs/view/stable-systemd/postlfs/firmware.html). I would recommend you inspect you hardware using `lsmod` command. When you deside which firmware you need, copy necessary folders and files in your `/usr/lib/firmware` folder. For exmaple if you using laptop with cheap AMD GPU picasso/raven family you need copy content of 'amdgpu' folder inside of `/usr/lib/firmware/amdgpu`. You don't need all files, you just need all `picasso*.bin`, `raven*.bin`. If you want support all devices and make sure to boot you system in different PCs just copy everything to `/usr/lib/firmware`

Compile and install Linux kernel and it's modules using `PKGFILE`

**Note:** `config` file for linux kernel contain minimal default parametrs for fast compilation and EFI support. If you desire change it you need unpack `linux-5.16.9.tar.xz` folder, copy `config` file as hidden `.config` run `make menuconfig` change or set kernel options, quit, save new `.config` file and replace `config` file in `\sources\packages\linux` folder:

````
cd /sources
tar xf linux-5.16.9.tar.xz
cd linux-5.16.9
cp /sources/packages/linux/config ./.config
make menuconfig
cp ./.config /sources/packages/linux/config
````

Install bootloader, using `systemd-boot` command:

````
systemd-boot install
````

Install `cpio` and `dracut` packages. Create initramfs using dracut:

````
dracut --kver=5.16.9
mv /boot/initramfs-5.16.9.img /boot/initramfs-5.16.9-lfs-11.1-systemd.img
````

Change necessary loader entires in `/bool/loader` folder:

````
cd /boot/loader
echo 'default lfs` > loader.conf
cd entires
cat > lfs.conf << EOF
title Linux From Scratch
linux /vmlinuz-5.16.9-lfs-systemd
initrd /initramfs-5.16.9-lfs-11.1-systemd.img
options root=/dev/sda3 rw
EOF
````

**Note:** replace `root=/dev/sda3` by yours root partiton using `vim`

Congratulations! You are done. If you kernel were set correctly and you have necessary firmware you could reboot now to yours new `LFS with pacman system`. Enjoy funny penguines while booting which represent amount of you PCs CPU cores.
