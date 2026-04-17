---
title: Exploratory developments
description: Exploratory development features modify the baseline decompiler.
weight: 30
---

>Note: We don't know how this will be structured when there are multiple contributors.  Assume for now that each contributor
>      maintains their own git branch, reviewing all branches for common code possibilities.

This walkthrough exercises an early version of the
[RISC-V vector transformation plugin](https://github.com/thixotropist/ghidra_decompiler_plugins).
That plugin has several layered dependencies, all of which we need to test in a series of  integration tests.
These tests are collected in the top level file `integrationTest.py`.

For this walkthrough we will run those tests in stages, annotated with descriptive text.

## Verify locally built decompile_datatest and our test inputs

The `T000BuildBaselineDecompiler` tests show that we can:

* rebuild `decompile` and `decompile_datatest` from the tarball source files, without using `gradle buildGhidra`, changing the C++ standard
  from `c++11` to `c++20` , or patching any new code into the decompiler
* run 21 datatest files through the `decompile_datatest` baseline with no low level errors
    * roughly 14 of the simpler datatest files run under `valgrind` to show no memory leaks

```console
$  python integrationTest.py T000BuildBaselineDecompiler
INFO:root:Cleaning the executable directory /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest
INFO:root:Running bazel build -c opt @ghidra_baseline//:decompile
INFO:root:Running bazel build -c dbg @ghidra_baseline//:decompile_datatest
.INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/memcpy_exemplars.ghidra with output to /tmp/memcpy_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/strcmp_exemplars.ghidra with output to /tmp/strcmp_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisperInit.ghidra with output to /tmp/whisperInit.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_1a.ghidra with output to /tmp/whisper_sample_1a.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_1b.ghidra with output to /tmp/whisper_sample_1b.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_2.ghidra with output to /tmp/whisper_sample_2.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_3.ghidra with output to /tmp/whisper_sample_3.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_6.ghidra with output to /tmp/whisper_sample_6.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_7.ghidra with output to /tmp/whisper_sample_7.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_8.ghidra with output to /tmp/whisper_sample_8.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_10.ghidra with output to /tmp/whisper_sample_10.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_11.ghidra with output to /tmp/whisper_sample_11.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/dpdk_sample_1.ghidra with output to /tmp/dpdk_sample_1.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/dpdk_sample_2.ghidra with output to /tmp/dpdk_sample_2.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/dpdk_sample_3.ghidra with output to /tmp/dpdk_sample_3.testlog
.INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_4.ghidra with output to /tmp/whisper_sample_4.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_5.ghidra with output to /tmp/whisper_sample_5.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_8.ghidra with output to /tmp/whisper_sample_8.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_9.ghidra with output to /tmp/whisper_sample_9.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_sample_12.ghidra with output to /tmp/whisper_sample_12.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_baseline_datatest < test_data/whisper_main.ghidra with output to /tmp/whisper_main.testlog
.
----------------------------------------------------------------------
Ran 3 tests in 156.082s
OK
```

## Verify build of patched decompile_datatest with c++-20 language standard

The next set of tests rebuilds `decompile` and `decompile_datatest` after patching the tarball sources to support plugins and
some basic Pcode edit methods.

```console
$  python integrationTest.py T001BuildPluginEnabledDecompiler
INFO:root:Cleaning the executable directory /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest
INFO:root:Running bazel build -c opt @ghidra_patched//:decompile
INFO:root:Running bazel build -c dbg @ghidra_patched//:decompile_datatest
.INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/memcpy_exemplars.ghidra with output to /tmp/memcpy_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/strcmp_exemplars.ghidra with output to /tmp/strcmp_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisperInit.ghidra with output to /tmp/whisperInit.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_1a.ghidra with output to /tmp/whisper_sample_1a.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_1b.ghidra with output to /tmp/whisper_sample_1b.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_2.ghidra with output to /tmp/whisper_sample_2.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_3.ghidra with output to /tmp/whisper_sample_3.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_6.ghidra with output to /tmp/whisper_sample_6.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_7.ghidra with output to /tmp/whisper_sample_7.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_8.ghidra with output to /tmp/whisper_sample_8.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_10.ghidra with output to /tmp/whisper_sample_10.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_11.ghidra with output to /tmp/whisper_sample_11.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/dpdk_sample_1.ghidra with output to /tmp/dpdk_sample_1.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/dpdk_sample_2.ghidra with output to /tmp/dpdk_sample_2.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/dpdk_sample_3.ghidra with output to /tmp/dpdk_sample_3.testlog
.INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_4.ghidra with output to /tmp/whisper_sample_4.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_5.ghidra with output to /tmp/whisper_sample_5.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_8.ghidra with output to /tmp/whisper_sample_8.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_9.ghidra with output to /tmp/whisper_sample_9.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_12.ghidra with output to /tmp/whisper_sample_12.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_main.ghidra with output to /tmp/whisper_main.testlog
.
----------------------------------------------------------------------
Ran 3 tests in 153.020s

OK

```

Verify that this `decompile_datatest` executable was plugin-enabled and that it loaded the spdlog logging module by reviewing
`/tmp/ghidraPluginManager.log`.  It should show 21 runs in which no plugin was found.

## Verify build and test of thixotropist plugin experiments

Now we can exercise thixotropist plugin code for RISC-V vector transforms.

```console
$  python integrationTest.py T100Thixotropist
INFO:root:Cleaning the executable directory /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile
INFO:root:Running rm -f /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest
INFO:root:Running bazel build -c opt @ghidra_patched//:decompile
INFO:root:Running bazel build -c dbg @ghidra_patched//:decompile_datatest
.INFO:root:Removing any previous plugin
INFO:root:Running rm -f /tmp/libriscv_vector.so
INFO:root:Building the riscv_vector plugin
INFO:root:Running bazel build -c dbg @//plugins:riscv_vector
INFO:root:copying libriscv_vector.so into the plugin search directory /tmp
.INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/memcpy_exemplars.ghidra with output to /tmp/memcpy_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/strcmp_exemplars.ghidra with output to /tmp/strcmp_exemplars.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisperInit.ghidra with output to /tmp/whisperInit.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_1a.ghidra with output to /tmp/whisper_sample_1a.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_1b.ghidra with output to /tmp/whisper_sample_1b.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_2.ghidra with output to /tmp/whisper_sample_2.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_3.ghidra with output to /tmp/whisper_sample_3.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_6.ghidra with output to /tmp/whisper_sample_6.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_7.ghidra with output to /tmp/whisper_sample_7.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_8.ghidra with output to /tmp/whisper_sample_8.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_10.ghidra with output to /tmp/whisper_sample_10.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_11.ghidra with output to /tmp/whisper_sample_11.testlog
INFO:root:Running SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/dpdk_sample_1.ghidra with output to /tmp/dpdk_sample_1.testlog
F
======================================================================
FAIL: test_03_shorter_exemplars (__main__.T100Thixotropist.test_03_shorter_exemplars)
Verify correct behavior under valgrind with smaller binaries, generally save files of less than 10 KB
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/thixotropist/projects/github/ghidra_decompiler_commons/integrationTest.py", line 254, in test_03_shorter_exemplars
    run_datatest(self, i, plugin=True, datatest_path=f"valgrind {DATATEST_PATH}", plugin_path=self.PLUGIN_PATH)
    ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/thixotropist/projects/github/ghidra_decompiler_commons/integrationTest.py", line 76, in run_datatest
    test_case.assertNotIn("Low-level ERROR", result.stdout,
    ~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                        "Decompiler completes without a low level error")
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AssertionError: 'Low-level ERROR' unexpectedly found in '[decomp]> restore test_data/dpdk_sample_1_save.xml\ntest_data/dpdk_sample_1_save.xml successfully loaded: RISC-V 64 little default\n[decomp]> map function 0x2ab4c2 rte_member_create\n[decomp]> parse line extern void rte_member_create(void* this);\n[decomp]> load function rte_member_create\nFunction rte_member_create: 0x002ab4c2\n[decomp]> decompile rte_member_create\nDecompiling rte_member_create\nLow-level ERROR: Free varnode has multiple descendants\nUnable to proceed with function: rte_member_create\n[decomp]> print C\nExecution error: No function selected\n[decomp]> print raw\nExecution error: No function selected\n[decomp]> \n' : Decompiler completes without a low level error

----------------------------------------------------------------------
Ran 3 tests in 117.148s

FAILED (failures=1)
```

This test set ran 12 tests and failed on the 13th test when the decompile_datatest executable threw an error:

    Low-level ERROR: Free varnode has multiple descendants

That low level error aborted the small binary test run, so we have no test data on the large binary tests.

Let's review the results for the `memcpy_exemplars` test.

The `/tmp/memcpy_exemplars.testlog` file shows that the vector instruction series and loop have been transformed:

```c
void memcpy_i2(void *to,void *from)
{
  vsetivli_e8mf8tama(2);
  vector_memcpy((void *)to,(void *)from,2);
  return;
}
...
void memcpy_v1(void *to,void *from,long size)
{
  vector_memcpy((void *)to,(void *)from,size);
  return;
}
```

This plugin has `trace` level logging enabled, so we can find details of the transformation logic in `/tmp/ghidraRiscvLogger_1252696.log`.
Note that the process ID is inserted into the logfile name, since Ghidra can launch multiple decompiler agents in parallel.

This plugin also provides survey reports on vector instruction sequences.  This is useful in feature extraction to support
transformation code.  For the `memcpy_v1` instance the report (in file `/tmp/riscv_summaries_1252696.txt`) is:

```
Vector Loop (simple):
        control structure is simple
        Loop start address: 0x48
        Loop length: 0x12
        setvli mode: element size=1, multiplier=1
        vector loads: 1
        vector stores: 1
        integer arithmetic ops: 4
        scalar comparisons: 1
        vector logical ops: 0
        vector integer ops: 0
        vector comparisons: 0
        vector source operands: 1
        vector destination operands: 1
        edges in: 1
        Vector instructions (handled | unhandled | epilog): vsetvli_e8m1tama, vle8_v, vse8_v, | | ?,
        Loop control variable: a2(0x00000050:3) = a2(0x00000048:e) + u0x10000000(0x00000050:11)
        Loop Local-scope Varnodes: a3(0x00000048:0), a1(0x00000048:d), a0(0x00000052:4), a2(0x00000050:3),
```

That survey report is enough to select the `vector_memcpy` transform as the transform to attempt.

Now look at the test failure, generated when running the `dpdk_sample_1` test case.  The testlog there gives us:

```console
[decomp]> restore test_data/dpdk_sample_1_save.xml
test_data/dpdk_sample_1_save.xml successfully loaded: RISC-V 64 little default
[decomp]> map function 0x2ab4c2 rte_member_create
[decomp]> parse line extern void rte_member_create(void* this);
[decomp]> load function rte_member_create
Function rte_member_create: 0x002ab4c2
[decomp]> decompile rte_member_create
Decompiling rte_member_create
Low-level ERROR: Free varnode has multiple descendants
Unable to proceed with function: rte_member_create
[decomp]> print C
Execution error: No function selected
[decomp]> print raw
Execution error: No function selected
[decomp]>
```

That `Low-level ERROR: Free varnode has multiple descendants` says that we incorrectly deleted a PcodeOp whose output Varnode
was used in multiple other locations.  Related tests show that four datatests fail, giving thixotropist some work to do.
These integration tests need refinement, since they only fail if the decompiler fails, not if the decompiler plugin generates
something other than what was expected.
