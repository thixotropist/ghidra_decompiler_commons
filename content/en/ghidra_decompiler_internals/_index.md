---
title: Ghidra Decompiler Internals
linkTitle: Ghidra Decompiler Internals
menu: {main: {weight:10}}
weight: 10
---

{{% blocks/section color='white' %}}

* Overview of the Ghidra decompiler
    * multiple processes, each single threaded, responding to commands from the Ghidra GUI
    * source and executable code locations
    * SLEIGH dependencies
    * build system
* Ghidra decompiler native build and test infrastructure
    * datatests
    * standalone command interface
* Ghidra Decompiler internals
    * Actions and Rules
    * PcodeOps and Varnodes
    * Blocks and Edges
    * Low level errors and common segfaults

{{% /blocks/section %}}
