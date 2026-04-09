---
title: Common Test and Development Resources for the Ghidra Decompiler
---

{{% blocks/section color='white' %}}

## Introduction

Ghidra's decompiler uses a system of rules, actions, and transforms to process binary code and data into
something similar to C for display in the Ghidra User Interface.  This is a powerful and complex core component
of the Ghidra tool suite - and one that needs to evolve as quickly as the compilers and processor cores that generate
those binaries.

This site collects documents, resource links, and design experiments that may help the broader Ghidra community
to help that evolution along.

## Outline

>Warning: everything here is subject to reorganization and reprioritization.  Updates and corrections are strongly encouraged.

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
* Community Testbeds and Initiatives
   * mumbel testbed and initiatives
       * pending
   * LukeSerne testbed and initiatives
       * pending
   * thixotropist testbed and initiatives
       * Description of research goals and scope
       * Add support for SPDLOG internal logging
       * Add binary RISC-V exemplars for Decompiler survey and testing
       * Add support for user-specific decompiler plugins
       * Add support for RISC-V vector instruction transforms into more readable C-like displays

{{% /blocks/section %}}