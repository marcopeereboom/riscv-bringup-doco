## Bringup of RISCV toolchain and virtual machine

This document walks the reader through the bringup of a RISCV virtual machine.
Currently the RISCV documentation is scattered all over the place and often
outdated. This document tries to simplify the steps involved in getting a viable
RISCV machine running an operating system built mostly from source.

All documentation was created using the `riscv64` platform only. The `riscv32`
tools have not all been upstreamed and should be avoided by casual users at
this time.

The steps outlined were done on a void Linux (https://voidlinux.org/) machine
but they should be roughly equivalent on other platforms such as macOS.

## Prerequisites

The following packages were installed on void Linux in order to build all other
components.

Update void Linux:
```
sudo xbps-install -Su
```

Install packages:
```
sudo xbps-install -S vim tmux git gcc make awk tar gzip gmp-devel mpfr-devel libmpc-devel flex bison automake autoconf autogen expect gperf tcl dejagnu patch guile diffutils wget texinfo subversion python pkg-config glib-devel pixman-devel zlib-devel bc xz
```

Note that `vim` and `tmux` are not required.

Add the following exports to .profile
```
export RISCV=~/riscv
export PATH=$PATH:$RISCV/bin
export QEMU=~/qemu
```

Create target directories:
```
mkdir -p $RISCV $QEMU
```

Make sure you get rid of `dash` since it breaks configure scripts.
```
xbps-alternatives -s bash
```

## RISCV Linux binutils and cross-compiler

Latest gcc branch is 8.3.0.

First we build gcc C cross compiler. We don't build the upstream compiler
because that requires one to jump through too many hoops. The RISCV provided
cross compiler is adequate. This builds glibc and binutils as well. 

```
git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
mkdir build && cd build
../configure --prefix=$RISCV
make -j8 linux
```

## riscv-pk

We build riscv-pk in order to verify that the cross compiler works.
```
git clone https://github.com/riscv/riscv-pk.git
cd riscv-pk
mkdir build && cd build
../configure --prefix=$RISCV --host=riscv64-unknown-elf
make -j8
```

This cross compiles several bits that we don't need to install but it is a
smoke test for the cross compiler.

## QEMU

Latest QEMU branch is v4.0.0-rc3
```
git clone https://github.com/qemu/qemu.git
cd qemu
git checkout v4.0.0-rc3
mkdir build && cd build
../configure --prefix=$RISCV
make install
```

This document that the QEMU disk images, bootloaders, kernel etc are stored in
the directory pointed at by environment variable `$QEMU`.

## Linux RISCV bootloader

There are two viable options to boot a RISCV Linux kernel in QEMU.
* OpenSBI (https://github.com/riscv/opensbi)
* bbl (https://github.com/riscv/riscv-pk)

The recommended method is `OpenSBI`. `bbl` is still supported but it was meant
as bringup tool and not a permanent solution. 

`OpenSBI` method

Clone repo, build and copy OpenSBI firmware.
```
git clone https://github.com/riscv/opensbi
cd opensbi
CROSS_COMPILE=riscv64-unknown-elf- make PLATFORM=qemu/virt
cp build/platform/qemu/virt/firmware/fw_jump.elf $QEMU
```

`bbl` method

## Linux kernel

Latest stable kernel is Linux 5.1-rc4.
```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/palmer/riscv-linux.git/
cd riscv-linux
```

Apply the following diff:
```
diff --git a/arch/riscv/kernel/vdso/Makefile b/arch/riscv/kernel/vdso/Makefile
index eed1c137f618..69caa42b7218 100644
--- a/arch/riscv/kernel/vdso/Makefile
+++ b/arch/riscv/kernel/vdso/Makefile
@@ -33,7 +33,7 @@ $(obj)/vdso.so.dbg: $(src)/vdso.lds $(obj-vdso) FORCE
 # table and layout of the linked DSO.  With ld -R we can then refer to
 # these symbols in the kernel code rather than hand-coded addresses.

-SYSCFLAGS_vdso.so.dbg = -shared -s -Wl,-soname=linux-vdso.so.1 \
+SYSCFLAGS_vdso.so.dbg = -s -Wl,-soname=linux-vdso.so.1 \
                             $(call cc-ldoption, -Wl$(comma)--hash-style=both)
 $(obj)/vdso-dummy.o: $(src)/vdso.lds $(obj)/rt_sigreturn.o FORCE
 	$(call if_changed,vdsold)

```
We need to apply this diff because our cross compiler does not support shared
objects at this time. That is ok for a kernel build.

Then create config and build the kernel:
```
make CROSS_COMPILE=$RISCV/bin/riscv64-unknown-elf- ARCH=riscv mrproper defconfig
make CROSS_COMPILE=$RISCV/bin/riscv64-unknown-elf- ARCH=riscv defconfig
make -j8 CROSS_COMPILE=$RISCV/bin/riscv64-unknown-elf- ARCH=riscv all
```

Depending on the boot method we need either `vmlinux` or `arch/riscv/boot/Image`

OpenSBI
```
cp arch/riscv/boot/Image $QEMU
```

bbl is a bit more involved because we need to link bbl and vmlinux together.
First go to the `riscv-pk` build directory and then link them. This section
asumes that all git repos were checked out in `~/git`.
```
cd ~/git/riscv-pk/build/
../configure --prefix=$RISCV --host=riscv64-unknown-elf --with-payload=~/git/riscv-linux/vmlinux
make -j8
cp bbl $QEMU
```

## Linux userspace

For now we are going to reuse the fedora disk image.
```
cd $QEMU
wget https://fedorapeople.org/groups/risc-v/disk-images/stage4-disk.img.xz
xz --decompress --keep stage4-disk.img.xz
```

This section will be expanded once this has been reduce to somewhat simpler steps.

## Booting QEMU

Create a script that contains the following lines:
```
qemu-system-riscv64 \
        -nographic \
        -machine virt \
        -smp 8 \
        -m 8G \
        -object rng-random,filename=/dev/urandom,id=rng0 \
        -device virtio-rng-device,rng=rng0 \
        -append "console=ttyS0 ro root=/dev/vda" \
        -device virtio-blk-device,drive=hd0 \
        -drive file=stage4-disk.img,format=raw,id=hd0 \
        -device virtio-net-device,netdev=usernet \
        -netdev user,id=usernet,hostfwd=tcp::10000-:22
```
This tells QEMU to launch a 64 bit RISCV machine with 8 CPUs and 8G of ram.
Additionally it disables the graphical console and switches the kernel to
serial console. It uses the downloaded Fedora userspace image and it redirects
the RISCV machine ssh port to 10000.

What is missing is what kernel to use. Pick on of the following methods and pay
close attention to add proper `\` to ensure line continuation in the script.

OpenSBI
```
        -kernel fw_jump.elf \
        -device loader,file=Image,addr=0x80400000 \
```

bbl
```
	-kernel bbl \
```

If everything was done correctly the machine will now boot a fresh RISCV
machine. You can either login on the console or ssh to it. Note that the root
password is `riscv`.
```
ssh -p 10000 root@localhost
```

At this point one can add a second disk to the QEMU virtual machine to add more
space and start natively building applications etc.

Note that Fedora has RISCV up to a point that it can boot vmlinux with an
initrd however it is still a little clunky. As RISCV progresses this document
will be updated to reflect new realities.

## FreeBSD

FreeBSD boots to single user and is not quite ready for prime time.

## NetBSD

NetBSD RISCV support is just words on a website. There is no working code.

## llvm-clang cross-compiler

The llvm-clang compiler works but reqires cross compiled binutils to link.
There is currently no hardfloat support upstream but patches are making it into
the downstream repo (https://github.com/lowRISC/riscv-llvm-integration). In
addition llvm/clang uses GNU headers and not their own yet.  Building
llvm/clang requires many GB of hard drive space.

```
sudo xbps-install -S cmake
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
mkdir build && cd build
cmake -DLLVM_ENABLE_PROJECTS=clang -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="RISCV" ../llvm
make -j8
make install
```

While the compiler compiles and can be made to work with tricks it is just not
ready for primtime use and requires the GNU toolchain to have been built with
multilib.
