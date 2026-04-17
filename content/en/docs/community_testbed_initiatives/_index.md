---
title: Community Testbed Initiatives
linkTitle: initiatives
menu: {main: {weight:10}}
description: Proposed new Decompiler features are tested here.
weight: 10
---

>TODO: a diagram of dependencies would be good here

Show that the decompiler can ...

* [X] be rebuilt outside of the Ghidra Gradle build system and replace the native decompiler.
* [X] be built under the C++-20 standard rather than the C++-11 standard.
* [X] be built with modular source code imported.  For instance, the [spdlog](https://github.com/gabime/spdlog) logging system.
* [X] be built with source code patches to support new features, such as user-contributed plugins, BlockGraph editing,
      inspection, and code survey.

Show that user-contributed plugins can ...

* [X] recognize sequences of vector instructions and transform them into higher semantic forms, such as `memcpy` or `strlen`
      variants
* [X] remove and replace Edges within a function
* [X] replace Blocks within a function, for instance absorbing loops into function calls.
* [X] survey complex binaries for instruction sequences that *may* be candidates for further transforms.
* [ ] do all of the above with over 95% success rates
