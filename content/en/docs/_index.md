---
title: Decompiler Commons
---

>Note: Everything here is subject to reorganization and reprioritization.
>      Updates and corrections are strongly encouraged.

* Demonstration of a Ghidra Decompiler common testbed to include:
    * Enhanced logging, support for c++20, build with imported source modules, enable user plugins.
    * Plugin features for inspection, function and block editing, code surveys, rules transforming
      instruction sequences into function calls.
    * Decompiler design constraints when importing architectures where the SLEIGH semantics are not fully known until
      runtime
* Catalog of proposed Decompiler features and their test/demo/complexity results
    * Change C++ standard from 2011 to 2020
    * Separate decompiler build from the `gradle buildGhidra` build
    * Include open source software modules
    * Add classes and methods for `BlockGraph` editing
    * Demonstrate new Rules for transforming instruction sequences into well-understood function calls like `memcpy`.
    * Add Survey capability to search for common instruction sequences.
* Ghidra decompiler reference material for community debugging
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
