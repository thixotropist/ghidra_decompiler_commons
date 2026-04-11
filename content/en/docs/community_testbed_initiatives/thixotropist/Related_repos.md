---
title: Related repositories
description: thixotropist has several public repositories on Github that may be relevant
weight: 30
---

These are all hosted under [github](https://github.com/thixotropist).

* a fork of the [primary Ghidra repo](https://github.com/thixotropist/ghidra), where RISC-V Instruction set
  extensions are supported in SLEIGH language files.  Note that this isa_ext branch rebases vector support
  from the unratified version 0.7 to the ratified version 1.0.
* Semi-portable [crosscompiler tool chains](https://github.com/thixotropist/bazel_compiler_modules) 
  building executables for RISC-V 64 bit platforms. These are packaged as Bazel modules for more reliable
  experimentation with multiple toolchain variants.  Contents include gcc 15.2 compiler, current binutils and libc, and a minimal Ubuntu-like sysroot for generic kernel integration.
* [Ghidra Decompiler Plugins](https://github.com/thixotropist/ghidra_decompiler_plugins) providing basic
  support for user-provided and processor-specific decompiler plugins.  This repo includes:
  * patching and building a decompiler to load new Rules from a sharable object plugin and to allow fast C++
    logging.
  * a few new methods to key Ghidra classes to suport Pcode and Block transforms in the Cleanup action
    group.
  * new Ghidra classes `Inspector` and `FunctionEditor`.  The `Inspector` class allows the user to explore
    Pcode and Block structures before and after modifications.  The `FunctionEditor` class supports edits
    to the decompiler's `Funcdata` objects, including BlockGraph edits, Varnode and Edge replacement, and
    descendent checks.
  * A sample RISV-V vector plugin to exercise the new capability, surveying a binary for RISC-V vector
    sequences and transforming relatively simple vector loops into higher level objects like `vector_memcpy`
    or `vector_strlen`.

