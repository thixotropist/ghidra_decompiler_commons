---
title: Decompiler Internals
linkTitle: decompiler internals
description: What we've learned about the Ghidra decompiler's internals.
menu: {main: {weight:30}}
weight: 30
---

>Note: a rough sketch of things we can put here

* Overview of the Ghidra decompiler
    * The decompiler executes as one or more processes, each single threaded,
      responding to piped commands from the Ghidra GUI.
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
