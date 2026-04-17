---
title: Testbed walkthrough
linkTitle: testbed walkthrough
menu: {main: {weight:20}}
description: The commons includes a testbed framework for exploratory development
weight: 20
---

>Note: Support for testing more than one Decompiler variant at a time will have to wait until we get more than one
>      to test. At the moment, just assume we have one long-running git branch per variant.

The test harness sequence is:
1. load a baseline software package such as
  [ghidra 12.0.4 source](https://github.com/NationalSecurityAgency/ghidra/archive/refs/tags/Ghidra_12.0.4_build.tar.gz)
2. build on the host system and deploy it locally, say to `/opt/ghidra_12.1_DEV`.
3. run all available data tests on this baseline, verifying success in each case.
4. build the test version of the Decompiler, for instance by patching the baseline source and rebuilding
5. run the test version of the Decompiler over the same test cases, likely with different definitions of success
