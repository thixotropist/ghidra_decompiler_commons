---
title: Testbed Overview
description: Thixotropist uses a testbed designed for rapid Decompiler experiments
weight: 10
---

>Note: The testbed described here is *arbitrarily* tuned for Ghidra analysis of larger RISC-V executables potentially found in
>      networked AI devices.  This kind of testbed should be easily redirected to other toolchains and platforms.

Recommended Testbed components:
* A current Ghidra source package, compiled and deployed.  We arbitrarily deploy to `/opt/ghidra_12.1_DEV`.
    * Running `doxygen` on the decompiler sources is highly recommended, with persistent browser tabs set to the `Funcdata`,
      `PcodeOp`, and `Varnode` class documentation.
* A compile-and-link crosscompiler toolchain tuned for the types of binaries to be analyzed.  This currently includes:
    * gcc 15.2 with C and C++ support, binutils, libc, etc.  The gcc configuration is `--prefix=/opt/riscv/sysroot --with-sysroot=/opt/riscv/sysroot --enable-languages=c,c++ --disable-multilib --target=riscv64-linux-gnu`
    * a system root directory representative of what the target binary's kernel might have generated.  We use a lightweight Ubuntu
      sysroot.  A real developer would bootstrap themselves up from a kernel build.
    * a build system that supports fast incremental builds and treats platforms and toolchains as first class managed objects.
      We use Bazel.  Cmake, meson, and others are alternates.
* A matching virtual machine capable of emulating the target processor.  We use `qemu-system-riscv64` over `u-boot`, with\
  configurable RISC-V extensions:
    ```console
    QEMU_CPU=rv64,v=true,zba=true,zbb=true,zbc=true,zbkb=true,zbkc=true,zbkx=true,zvbb=true,zvbc=true,vlen=256,vext_spec=v1.0
    ```
  The virtual machine is sometimes needed to exercise a function or code fragment of the binary under test, to see if it really
  does what the analyst *thinks* it does.
* debugging tools like `valgrind` and` gdb`.
* a C++ logging subsystem like [spdlog](https://github.com/gabime/spdlog).
* A suite of decompiler `datatests`, run within a python `integrationTest.py`.  Each data test consists of a code fragment wrapped
  as an importable Ghidra XML file and a matching set of Ghidra decompiler standalone test commands.  If you are using something
  like DPDK as an exemplar training binary, then you will have files like `dpdk_sample_1_save.xml` and `dpdk_sample_1.ghidra`in your
  `test` directory.  A single test execution will often look like:
    ```console
    SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test/dpdk_sample_1.ghidra.ghidra
    ```

There are a lot of ways to manage your source-level experiments with the Ghidra decompiler.  Our current choices are:
* Don't fork the Ghidra source repo
* Do closely track developer commits to the Ghidra source repo
* Don't expect a decompiler PR to be reviewed or accepted anytime soon.
* Do patch a released tag of the Ghidra source repo to support a dynamic plugin capability and to add relatively simple components
  like `spdlog` logging support and possibly new methods to core decompiler classes.
* Do structure the bulk of your enhancements to the decompiler as new plugin Rules and Actions,
  added to the existing Rules and Actions and triggered at an appropriate phase.
* Do use a lot of logging messages in your code.  It is very easy to trigger an internal consistency check and segfault,
  so flag PcodeOp changes with `info` messaging and non-modifying progress steps with `trace` messages.
  Check if your code leaves any `free` Varnodes
  in the wrong places and emit an `error` message followed by a `flush`, as the decompiler will surely segfault sometime after your new code completes.

>EDIT: add links to existing thixotropist testbeds to show some examples
