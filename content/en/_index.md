---
title: Ghidra Decompiler Extensions Testbed
---

{{% blocks/section color='white' %}}

## Introduction

Ghidra's decompiler uses a system of rules, actions, and transforms to process binary code and data into
something similar to C for display in the Ghidra User Interface.  This is a powerful and complex core component
of the Ghidra tool suite - and one that needs to evolve as quickly as the compilers and processor cores that generate
those binaries.

This site collects:
* integration tests to help show that improvements made to the decompiler do not cause regression failures.
* demonstration of decompiler feature extensions, showing what improvements are possible and at what cost in
  the future maintenance burden.
* documents, resource links, and design experiments that may help the broader Ghidra community to help that
  evolution along;

Not all features tested here will make it into decompiler pull requests.  Some 'scaffolding' features, like extensive
logging and tracing, are used to accelerate exploratory development.

{{% /blocks/section %}}