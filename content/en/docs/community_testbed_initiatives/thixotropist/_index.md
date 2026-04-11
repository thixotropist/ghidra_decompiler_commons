---
title: Thixotropist Testbed Initiatives
linkTitle: Thixotropist testbed and initiatives
description: Thixotropist explores enhancements to the decompiler supporting newer processors and toolchains
weight: 30
---

{{% blocks/section color='white' %}}

This group of initiatives and testbed features emphasizes exploratory development of Ghidra Decompiler
features to better track areas in which the decompiler needs to evolve faster to keep up with modern
processor and toolchain evolution.  It seeks to answer questions like "We need Ghidra analysis of binaries
that will begin to show up in 36 months - will it take us 18 or 72 months to get support for that into the Ghidra Decompiler?"

Initiative features:

* Use RISC-V instruction set extensions - especially vector, crypto, and bit manipulation extensions -
  as the evolutionary driver.  This processor family is seeing rapid evolution and can serve as a good
  excellent evolutionary driver for Ghidra.
    * Narrow the scope to 64 bit RISC-V cores implementing the RVA-23 extension profile and serving medium
      to high end servers and embedded systems.
* Use current Gnu Compiler Suite toolchains supporting those extensions
    * Currently GCC 15.2 crosscompiler toolchain with a lightweight linux sysroot
* Assume for concreteness that we are interested in analyzing forthcoming autonomous vehicle firmware,
  but have no current examples of such firmware.  How do we explore those futures?
* Emphasize rapid build-and-test development cycles with good debugging instrumentation.
    * Trial and error works well, if the errors come quickly and you learn from and record each one.
* Rebase frequently with Ghidra releases, while avoiding relying on project PRs being accepted.
    * Therefore patch a plugin manager into the Decompiler, then load user-specific plugins to explore
      new features.
* Development testing should use the native Ghidra Decompiler `datatest` framework and not require a
  full rebuild of the Ghidra Java GUI or the Decompiler core code.  Just recompile the plugin.
* Users can launch `ghidraRun` easily either with or without a single plugin module.
* Project success is when 90% of binaries analyzed give clearer results in the Decompiler window,
  and 5% or fewer function decompilations result in a Decompiler segfault or Low level Error.
* No changes to the Ghidra GUI code or the interprocess API between the decompiler processes and the
 Ghidra GUI.

Non-features:
* no immediate testing on Windows or other non-linux host systems
* no attention to maintaining emulation capability - use qemu for that.
* no strict adherence to current Decompiler build tools and configuration.  
* no ability to switch plugins from within the Ghidra GUI - closing Ghidra and relaunching
  it is good enough for now.

{{% /blocks/section %}}