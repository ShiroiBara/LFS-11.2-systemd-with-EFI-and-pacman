# LFS 11.1 systemd with EFI and pacman

### Installing LFS systemd edition with EFI support and maintain it with pacman package manager ###

Based on [Linux From Scratch](https://www.linuxfromscratch.org/lfs/view/stable-systemd/index.html)                                                           
Based on [the guide writen by James Kimball](http://lists.linuxfromscratch.org/pipermail/hints/2013-March/003304.html) in 2013                               
Based on [Pacman on LFS 8.1](https://github.com/benvd/lfs-pacman)" By Ben Van Daele                                                                           
Licensed under the [Creative Commons Attribution-ShareAlike 3.0 License](https://creativecommons.org/licenses/by-sa/3.0/)

## Introduction

This guide is divided in five stages, the first one of which starts just after chrooting to your final system.

- Stage 1 - Installing pacman to you root with /usr prefix as temporary tool. All libraries, headers and binaries will be later replaced by their
            recompiled versions during **Chapter 8. Installing Basic System Software.** 
- Stage 2 - Installing part of basic system software from chapter 8 using pacman and PKGBUILD files following order from the book until
            you satisfy all prerequsites to build final pacman as package with PKGBUILD file.
- Stage 3 - Installing pacman with temporary pacman as final basic software.
- Stage 4 - Installing rest of basic software from chapter 8 following order from the book include aditional packages for EFI support. 
- Stage 5 - Finishing the book.

### Stage 1 - Installing temporary pacman to your root

This stage begins right after finishing subsection of chapter **7.10. Perl-5.34.0** of Linux From Scratch Version 11.1-systemd.

#### pacman and makepkg dependencies

To build latest pacman and use makepkg for creating you own packages you need to install those libraries and tools:

- zlib
- opensll
- python with zlib module for .pyz files
- ninja
- meson as .pyz file
- util-linux for getopt
- libcap
- shadow for su
- libarchive
- pkg-config
- fakeroot

Most of these are not part of the LFS or BLFS books, so download their sources manually:

- libarchive: <https://github.com/libarchive/libarchive/releases/download/v3.6.0/libarchive-3.6.0.tar.xz>
- fakeroot: <https://deb.debian.org/debian/pool/main/f/fakeroot/fakeroot_1.29.orig.tar.gz>
- pacman: <https://sources.archlinux.org/other/pacman/pacman-6.0.1.tar.xz>

Additional packages to support EFI boot:

- gnu-efi https://download.sourceforge.net/gnu-efi/gnu-efi-3.0.14.tar.bz2

Build these packages using the following commands. Just like the LFS book, these commands assume you've extracted the relevant sources and `cd`'d into the resulting directory.

**Note:** At this point, I would recommend having TWO terminals running, one CHROOTed into $LFS and one logged in as root on the host. Some files require editing with a text editor (such as vi/vim or nano) and at this point, there is not one present in the chrooted environment. Download/save the files in the chroot environment and if necessary, edit them from the host.

**Note:** If you working in non graphical environment, but have something like boot-cd with minimal tools, you may build some simple editors like nano or wim and use them for edit PKGBUILD and other text files.

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

### Stage 2 - Installing part of basic system software

#### Clean up temporary system acording book

````
rm -rf /usr/share/{info,man,doc}/*
find /usr/{lib,libexec} -name \*.la -delete
rm -rf /tools
````

#### Create directory for building and store packages and change to it

````
mdkir /packages
cd /packages
````
Before you start build packages, using makepkg command, you will need `PKGBUILD` files. You can get them from [here.](https://github.com/ShiroiBara/lfs-pacman/tree/master/packages) You can download them in zip format using github option. Unpack folder packages inside `/packages` directory 

Change to package directory. Our first package will be linux-api-headers. Even it was installed manually we need reinstall it with pacman for future easy removal or replacement in case of upgrading system.

````
cd linux-api-headers
cp ../../sources linux-5.16.9.tar.xz .
su tester -c "PATH=$PATH LANG=$LANG makepkg -c"
mv linux-api-headers-5.16.9-1-x86_64.pkg.tar.gz ..
cd ..
pacman -U linux-api-headers-5.16.9-1-x86_64.pkg.tar.gz --overwrite=*
````

For second and rest packages follow book order. Instructions for building will stay same. First you need to change to package directory and copy package tarball file, include patches if they needed. Next you envoke `makepkg` as example above, copy builded pacman packages to `/package` directory and install it to final system.

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

And configure the dynamic loader exactly as it used in **chapter 8.5.2.3**

**Note:** Setup `LANG` variable acording your system locale. It will be later used while building GCC. Incorect setup of it will produce building error.

For example for US issue this:

````
LANG=en_US.UTF-8
````

Now you can contnue build system until you finish chapter **8.58. Groff-1.22.4**

## Stage 3 - Installing pacman with temporary pacman as final basic software

It's time now to reinstall pacman as pacman's package. But before we need build and reinstall as pacman's packages two libraries:

- libarchive
- fakeroot

Build them using `PKGBUILD` files and install using `pacman -U <package name> --overwrite=*` command.
Backup yours `.config` files, because reinstalling pacman will overwrite them:

````
cp /etc/pacman.conf /etc/pacman.conf-back
cp /etc/makepkg.conf /etc/makepkg.conf-back
````

Finally we can build pacman as it's own package and intall it as final software. After finishing, don't forget restore your config files:

````
su tester -c "PATH=$PATH LANG=$LANG makepkg -c"
pacman -U pacman-6.0.1-1-x86_64.pkg.tar.gz --overwrite=*
````

And restore `.config` files back:

````
mv /etc/pacman.conf-back /etc/pacman.conf
mv /etc/makepkg.conf-back /etc/makepkg.conf
````

## Stage 4 - Installing rest of basic software from chapter 8 following order from the book include aditional packages for EFI support.

Skip building **chapter 8.59. GRUB-2.06**. Since system will be booted in EFI mode, instead of `grub` bootloader we will use `systemd-boot`. Continue build pakages from **chapter 8.60. Gzip-1.11**, using `PKBUILD` files and install them to your final system until you reach **chapter 8.70. Jinja2-3.0.3**.
Now we need build additional package `gnu-efi` to add EFI support for using it with systemd. Build and install it using correspondent `PKGBUILD` file.
When you finishing building and installing `gnu-efi` package you can build `systemd` package. The `PKGBUILD` has some additional lines to satisfy EFI support
for `systemd` and use later `systemd-boot` command to install bootloader:

```
-Dgnuefi=true
-Dsbat=
```
