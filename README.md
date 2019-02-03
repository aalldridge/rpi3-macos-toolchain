# Cross-compiling toolchain for macOS Mojave 10.14 host and Raspberry Pi Model 3 B+ Raspbian Stretch target

## Description and acknowledgements

This is a basic GNU toolchain for Mac to RPi Model 3 B+ cross compilation, with  

- hard-float ABI, and 
- optimisation for Cortex-A53 CPU and Neon-Cortex-v8 FPU.

The toolchain build was based on the procedure described in detail in the following two sources:

- [Guide no. 1 by Jared Wolff](https://www.jaredwolff.com/cross-compiling-on-mac-osx-for-raspberry-pi/#show1)
- [Guide no. 2 by Yuzhou Cheng](https://medium.com/coinmonks/setup-gcc-8-1-cross-compiler-toolchain-for-raspberry-pi-3-on-macos-high-sierra-cb3fc8b6443e)

The updated version of Jared Wolff's guide (referred to in the following as *Guide no. 1*) is geared towards a host running OS X 10.11.6 and a Pi 2 or Pi 3 B target. The comments on his page contain a wealth of helpful information for later configurations. Yuzhou Cheng's guide (referred to as *Guide no. 2* in the following) updates Jared Wolff's procedure to a host running OS X 10.13.6 and has a number of helpful troubleshooting hints. Moreover, his configuration allows for much more recent versions of the tools involved, e.g. gcc 8.1.0.

This document describes how I needed to update the procedure to work on a host running macOS Mojave 10.14.2. The procedure also installs the versions of all tool available of Raspbian Stretch at the time of writing. One exception is GCC, which is installed in version 6.3.0 on my RPi 3 target. There are no working Crosstool-NG patches available for this version, so I have chosen to go with GCC 6.5.0.

**Before proceeding, make yourself familiar with Crosstool-NG.** I highly recommend the overview in *Guide no. 2*. I learnt everything I needed to know there. 

## Basic set-up of tools

As described in *Guide no. 2* (and to some extent in the original *Guide no. 1*), a number of tools need to be installed using *Homebrew*. After installing Homebrew, the basic packages needed are described on the [Crosstool-NG web site](https://crosstool-ng.github.io/docs/os-setup/).

In particular, we need to do the following: 
```shell
brew install crosstool-ng --with-grep
chmod +x /usr/local/Cellar/crosstool-ng/1.23.0_1/lib/crosstool-ng-1.23.0/scripts/crosstool-NG.sh
brew install help2man bison make guile
```
Here, the path to Crosstool-NG has to be modified according to the version installed. Note, moreover, that we insist on installing GNU `make`. This is because some of the steps require `make` 4.2.1, whereas macOS only provides an older version of GNU `make`. (On my system, version 3.81.)

## Forcing Mojave with a wrecking bar

An annoying aspect of macOS Mojave is that the filesystem protection mechanism has been escalated to the extent that previous solutions to forcing macOS to use the Homebrew versions of `bison` and `make` do *not* work. In particular, adding the corresponding directories to the beginning of `$PATH`, using `ct-ng link --force`, together with `chown -R ${whoami} /usr/local` or something similar do *not* work.

The only way I have found to get the `libc_start_files` step of the toolchain creation to work is the following:

- Temporarily disable filesystem protection. To do that: 
	- Boot into recovery.
	- Invoke
	```shell 
	csrutil enable --without fs
	``` 
	in Terminal.
	- Restart into macOS.
- At the command line, do 
```shell
cd /usr/bin
sudo mv bison bison-old
sudo mv gnumake gnumake-old
ln -s /path/to/brew/bison bison
ln -s /path/to/brew/make gnumake
```
In my case, `/path/to/brew/bison` was `/usr/local/opt/bison/bin/bison` and `/path/to/brew/make` was `/usr/local/opt/make/bin/make`.
- Reenable filesystem protection as follows:
	- Boot into recovery. 
	- In Terminal, invoke 
	```shell
	csrutil enable
	```
	- Reboot into macOS. 

## Crosstool-NG configuration and patches

From this point on, I am assuming that two case-sensitive disk images are in place and mounted, as explained both in *Guide no. 2*  and *Guide no. 1*. In my case, the respective directories are reflected in the Crosstool-NG configuration as follows:
```shell
CT_LOCAL_TARBALLS_DIR="/Volumes/xtools-scratch/src"
CT_WORK_DIR="/Volumes/xtools-scratch/.build"
CT_PREFIX_DIR="/Volumes/rpi-xtools/${CT_TARGET}"
```

It is necessary to configure Crosstool-NG in `.config` to install the latest versions of the tools. Since the `menuconfig` option will not reflect the latest versions, it is best to do this manually using some text editor of your persuasion. In my case, the requisite settings are: 
```shell
CT_KERNEL_VERSION="4.14.79"
CT_BINUTILS_VERSION="2.28"
CT_LIBC_VERSION="2.24"
CT_CC_GCC_VERSION="6.5.0"
CT_GDB_VERSION="7.12.1"
CT_LTRACE_VERSION="0.7.3"
CT_STRACE_VERSION="4.15"
CT_ZLIB_VERSION="1.2.8"
CT_LIBICONV_VERSION="1.15"
CT_GETTEXT_VERSION="0.19.8.1"
CT_GMP_VERSION="6.1.2"
CT_MPFR_VERSION="3.1.5"
CT_ISL_VERSION="0.19.8.1"
CT_MPC_VERSION="1.0.3"
CT_LIBELF_VERSION="0.8.13"
CT_EXPAT_VERSION="2.2.0"
CT_NCURSES_VERSION="6.0"
CT_AUTOCONF_VERSION="2.69"
CT_AUTOMAKE_VERSION="1.15"
CT_LIBTOOL_VERSION="2.4.6"
CT_M4_VERSION="1.4.18"
CT_MAKE_VERSION="4.1"
```

Crosstool-NG will only build successfully with these versions set if the corresponding patches are in place. To that end, clone the Crosstools-NG repository, viz. 
```shell
git clone https://github.com/crosstool-ng/crosstool-ng.git
cp -R crosstool-ng/packages /Volumes/xtools-scratch
```
This assumes that you have `git` installed. See [here](https://git-scm.com/download/mac). It makes sense to do this at a resonable location, preferably case-sensitive (just in case). I did it in `/Volumes/xtools-scratch`, the root directory of my larger case-sensitive image.

Now, `.config` needs to be adapted to reflect the location of the patches correctly; also, it should be configured to apply the local patches. In my case: 
```shell
CT_PATCH_LOCAL=y 
CT_PATCH_ORDER="local"
CT_PATCH_SINGLE=y
CT_PATCH_USE_LOCAL=y
CT_LOCAL_PATCH_DIR="/Volumes/xtools-scratch/packages"
```

For some tools, there are no corresponding patches available. This may be a problem, in particular for the target libraries, if the versions installed on your target are different. For this reason, I have chosen not to build zlib for the toolchain. For GCC, I need to opt for a version different from that on my target, namely 6.5.0 instead of 6.3.0, since I could not find working patches. In case of other inconsistencies, you may have to manually upgrade the versions of libraries on your target.

This summarises the most important changes to `.config`. The file is also part of this repository. 

## Crosstool-NG fine tuning for hard-float and optimization for Cortex-A53 CPUs and their NEON-Cortex-v8 FPUs

The Raspberry Model 3 B and 3 B+ are based on a Broadcom BCM2837 SoC which has four Cortex-A53 cores. Each of these cores has a Neon-Cortex-v8 FPU. 

Since there are FPUs present, it makes sense to base the toolchain on the hard-float ABI. Otherwise, we are sacrificing performance, which may severely limit the applicability of software compiled with this toolchain for the RPi 3 B+ platform. 

Moreover, NEON allows the parallel execution of up to four FP operations, so it is to be expected that optimisation of FP operations that takes this into account will be much more efficient than without these optimisations. 

In order to activate these optimisations (and some others) in the toolchain, the following configuration is passed to Crosstool-NG:
```shell
CT_ARCH_SUPPORTS_WITH_ARCH=y
CT_ARCH_SUPPORTS_WITH_CPU=y
CT_ARCH_SUPPORTS_WITH_TUNE=y
CT_ARCH_SUPPORTS_WITH_FLOAT=y
CT_ARCH_SUPPORTS_WITH_FPU=y
CT_ARCH_SUPPORTS_SOFTFP=y
CT_ARCH_EXCLUSIVE_WITH_CPU=y
CT_ARCH_CPU="cortex-a53"
CT_ARCH_FPU="neon-fp-armv8"
CT_ARCH_FLOAT_HW=y
CT_TARGET_CFLAGS=""
CT_TARGET_LDFLAGS=""
CT_ARCH_FLOAT="hard"
CT_ARCH_ARCH_CFLAG="armv8-a+crc"
CT_ARCH_ABI_FLAG="aapcs-linux"
CT_ARCH_CPU_CFLAG="cortex-a53"
CT_ARCH_FLOAT_CFLAG="hard-float"
CT_ARCH_TUNE_CFLAG="cortex-a53"
CT_ARCH_ENDIAN_CFLAG="little-endian"
CT_ARCH_TARGET_CLFAGS="-funsafe-math-optimizations -mtls-dialect=gnu2 -munaligned-access -mvectorize-with-neon-quad"
```

## Crosstool-NG: First and second run

Invoke
```shell
ct-ng +companion_tools_for_build
```
This will download the tarballs for the selected tools, extract and patch them. Subsequently, it will attempt to build the tools such as `autoconf` and `make` for the remaining build. 

This will fail. First, it might happen that some of the tarball downloads fail. This is due to the fact that the SourceForge locations for their latter versions are not entirely reliable. Just download the missing tarballs manually into `/Volumes/xtools-scratch/.build/tarballs` (adapting this path to your configuration) and keep reinvoking `ct-ng +companion_tools_for_build` until all the patches are applied successfully. 

Secondly, you will most probably experience trouble when building `make`. Just run `autoreconf` in `/Volumes/xtools-scratch/.build/src/make-whatever-the-version`.

At this point, it is important that you previously installed `guile` using Homebrew. If not, you may have to install it, add some paths to your `~/.bash_profile`, load them via `source ~/.bash_profile`, and rerun `autoreconf`. 

Once this has completed successfully, reinvoke 
```shell
ct-ng +companion_tools_for_build
```
at the command line. This should now successfully build and install the tools for the further steps. 

## Manual patches to binutils and gcc configuration 

At this point, all your sources are in place and patched, and you have build the tools for the build. 

We have apply some manual patches gleaned from the two guides: 

- Patch `/Volumes/xtools-scratch/.build/src/binutils-whatever-the-version/gold/gold-threads.cc` according to *Guide no. 1* under *Change source file*.
- Patch `/Volumes/xtools-scratch/.build/src/gcc-whatever-the-version/libatomic/configure` as described in *Guide no. 2*. 

## Completing the build

We are now ready to continue. Before proceeding, say 
```shell
ulimit -n 1024 
```
to increase the maximum number of open file handles to 1024. The system default set by macOS will not be sufficient. 

Then we can continue with the next step in Crosstool-NG, which is
```shell
ct-ng companion_libs_for_build+
```
How long this will take depends strongly on your setup and on the allowed parallel threads set in `.config`. The setting I used (which is also in the file included with this repository) is 
```shell
CT_PARALLEL_JOBS=4
```
You may want to modify this depending on system resources. Expect at least 40 minutes for the build to complete. 

## A note of caution when working with disk images 

Especially when working on a laptop, you may have restrictive power saving settings in place. I have experienced corruption of disk images when building on a laptop that went to sleep during the build.

For this reason, **I recommend setting *Turn display off after* in `System Settings/Energy Saver` to *never* until the build is finished and the images are successfully unmounted.**

