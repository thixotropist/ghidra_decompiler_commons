---
title: Setting the Baseline
description: Building and testing the base Ghidra system before extensions.
weight: 10
---

Our testbed starts with a baseline Ghidra system, built from source.  We build it and deploy it to the host
file system, then run baseline tests to check for problems in the test harness.

## Pick a Ghidra source baseline

First identify the specific baseline source.  We initially might have two choices:
* [ghidra 12.0.4 source](https://github.com/NationalSecurityAgency/ghidra/archive/refs/tags/Ghidra_12.0.4_build.tar.gz)
* [a fork of 12.0 with customized SLEIGH files](file:////tmp/ghidra_isa.tar.xz)

Update `MODULE.bazel` so that the `http_archive`s named `ghidra*` identify the baseline you choose.  For this walkthrough,
that's the fork.

```python
# From MODULE.bazel
LOCAL_URL = "file:////tmp/ghidra_isa.tar.xz"
LOCAL_SHA256 = "02a9c20126b87a7e04bc5ff1a5c230e42fe1b885c3b74127de41a1a105f681b9"
```
## Build and deploy

Unpack the tarball into a ghidra source directory, then build and deploy

```console
$ gradle -I gradle/support/fetchDependencies.gradle
...
$ gradle buildGhidra
...
$ cd build/dist
$ ls
ghidra_12.1_DEV_20260415_linux_x86_64.zip
$ pwd
/home/thixotropist/projects/github/ghidra/build/dist
$ cd /opt
$ sudo unzip /home/thixotropist/projects/github/ghidra/build/dist/ghidra_12.1_DEV_20260415_linux_x86_64.zip
$ ls ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile
total 4488
-rwxr-xr-x. decompile
-rwxr-xr-x. sleigh
$ cd ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64
$ sudo cp decompile decompile_baseline
$ sudo chown -R thixotropist:thixotropist .
$ ls -lh
total 7.8M
-rwxr-xr-x.  3.4M decompile
-rwxr-xr-x.  3.4M decompile_original
-rwxr-xr-x. 1009K sleigh
```

Note the last few steps:
* We change the ownership of the directory holding the original decompiler, as we will want to replace it with
  a decompiler variant for testing
* Since we will change the baseline decompiler, we want to save the original version as `decompile_baseline`
* The base distribution builds only the `decompile` executable, not any part of the Decompiler test infrastructure.
  The test harness may add that later as `decompile_datatest`

## Test the baseline deployment

Test this with a component of the integration test:

```console
$ python integrationTest.py T0BuildBaselineDecompiler
INFO:root:Cleaning the executable directory /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/
INFO:root:Running bazel build -c opt @ghidra_baseline//:decompile
INFO:root:Running bazel build -c dbg @ghidra_baseline//:decompile_datatest
.
----------------------------------------------------------------------
Ran 1 test in 18.727s

OK
```

That integration test case replaced the baseline decompile:

```console
$ ls -lh /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/
total 139M
-r-xr-xr-x.   4.7M decompile
-r-xr-xr-x.    65M decompile_baseline_datatest
-r-xr-xr-x.    65M decompile_datatest
-rwxr-xr-x.   3.4M decompile_original
-rwxr-xr-x.  1009K sleigh
```

This is a good time to verify that the full Ghidra system is working, by launching `/opt/ghidra_12.1_DEV/ghidraRun`.

## Summary

We now have a local copy of Ghidra built from a defined source archive, with multiple changes to the directory
holding the Ghidra decompiler:

* The original `decompile` executable, as built by Ghidra's gradle build system, is now named `decompile_original`.
* A rebuild of `decompile` using the local Bazel build system rather than gradle replaces the original
    * ⇒this allows us integration tests to see if using the local build system - with no changes to the decompiler
      C++ sources - causes problems.
* A new build of `decompile_datatest` as `decompile_baseline_datatest` using the local Bazel build system -
  also with no changes to the decompiler C++ sources.  We use this to run through our regression tests to show what
  an unmodified decompiler gives us.
* Each decompiler variant will replace the `decompile` and `decompile_datatest` files.  The `decompile` is used by
  the Ghidra GUI while `decompile_datatest` exercises just the decompiler with no Java dependency.
