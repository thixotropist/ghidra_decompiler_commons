---
title: Advanced Debugging
description: Ghidra will throw low level errors if we violate unpublished integrity constraints.  What methods work in resolving such issues?
weight: 40
---

>Note: This page summarizes work to resolve [Issue #3](https://github.com/thixotropist/ghidra_decompiler_commons/issues/3)

The thixotropist plugin recognizes inlined RISC-V vector instruction patterns implementing `memcpy`, `memset`, `strlen`,
and `strcmp` library functions, replacing them with typed builtin function calls `vector_memcpy`, `vector_memset`, `vector_strlen`,
and `vector_strcmp` respectively.  For the reference executable `whisper.cpp`, with roughly 1500 non-thunk functions, it improves the
decompiler results in 98% of the functions - and crashes with a couple of low-level errors in the other 2%.  Since the failed functions
include critical functions like `main`, it's worthwhile resolving and fixing the bug or bugs.

First some context:

* The plugin recognizes certain vectorized loops, transforming those loops into function calls.  That transform involves Rule-based changes
  to the function's BlockGraph, removing one Block entirely and rewriting affected Edges and Varnodes.
* When running the plugin and decompiler from the Ghidra GUI, the errors manifest as one of two different low-level errors.  When running
  the plugin and decompiler as datatests, the errors manifest as std::vector bounds checks - trying to access a non-existent element of a std::vector.
* The error is *not* thrown within the plugin code, but in an Action that executes after the plugin completes its work.
* Visual inspection of the plugin logging messages provides no insight into the nature of the problem.

The decompiler has many implicit consistency rules that are easy to misunderstand during Pcode transforms.
Violating these rules will manifest as Low level errors and exceptions crashing the decompiler.
There are several debugging paths available to us:

1. Source code inspection to try and infer obscure decompiler design rules and constraints.
2. Inspection of decompiler plugin logging and tracing messages, especially `warn` or `error` log messages.
   This is often accompanied with additional trace-level log messages to provide more context.
3. Run the decompiler datatest under `valgrind` or `gdb` to inspect backtrace stacks and objects associated with an error.
4. Generate synthetic, minimal functions manifesting the observed error(s) to perform differential analysis on elements
   triggering the faults.
5. Add additional logging messages to the core decompiler - not a plugin - to display private object data
   associated with a fault.

This case study uses methods 1 through 4 to resolve a particularly confusing set of problems.
Think of these paths as dimensions to explore in parallel, where each step down one path helps to refine
the preferred options for progressing down the other paths.

### Getting started

>Warning: Some of the assumptions in the text below turn out to be incorrect.  Backtracing from faulty
>         assumptions to solid insight is a skill worth developing.

The largest number of decompiler low level errors are now reported as this - from `get_next_arg`:

      Low-level Error: Unable to force merge of op at 0x00037f10:182

This is now wrapped as the integration test `whisper_sample_15`.  Inspecting the log traces suggests that
this is *not* a consistency check related to edge editing, but something dealing with varnode merging into
symbols.  The decompiler will attempt to merge varnodes and tie them into a smaller number of symbols.
The plugin removes a lot of varnodes, and may be breaking links needed for that merge step.

The `get_next_arg` refers to multiple symbols, but only a few are loaded into registers:
* `vl` is the symbol tied to a Control and Status register field.  Ghidra tracks this `MULTIEQUAL` PcodeOp with an internal
  register named (in this case) `c0x0c20`.  This symbol is loaded into register `a4` during a `vector_strlen` stanza.
* `stderr` is a symbol bound to a RAM address and loaded into register `a2`.
* a string symbol is loaded into register `a1` in a code block ending in `exit`, so its heritage need not be tracked.

Restart Ghidra with the plugin enabled, and open `whisper_cpp_rva23` to inspect `get_next_arg`.  The decompiler fails
with `Unable to force merge of op at 0x00037f10:182`.  Can we patch this function to clear the fault, and get something
useful out of the decompiler?

Use the `Patch instruction` feature in the listing page to replace these lines with nop instructions:

```text
                             LAB_ram_00037e5c
    ram:00037e5c 17 b6 10 00     auipc  param_3,0x10b
    ram:00037e60 03 36 c6 9b     ld     param_3,-0x644(param_3=>->stderr)
...
    ram:00037eb2 73 27 00 c2     csrr   param_5,vl
```

After *all* three lines are patched to `nop` the decompiler window shows the low level error has cleared.  The decompiler
reruns after *each* of the patches is installed, so all three patches appear to be needed to clear the error.

Export the patched function to a debug file, which we can later use in a series of differential tests to see how each patch
alters the generated Pcode and especially MULTIEQUAL nodes.

We are likely to also need some sort of audit of the symbol table ties to varnodes.  Most of the symbol table work occurs
after the plugin runs, so we need to look for subtle context data that prevents symbol analysis.

### Differential testing methodology

We need a methodology that lets us make steady progress towards a solution.  Finding minimal failing and passing examples
and studying the plugin differential results is a good start.  The steps taken so far are:

1. Export a largish executable as C/C++ with the plugin enabled.
2. Collect instances of Low-level errors and decompiler failures.
3. Resolve any easy problems to reduce clutter.
4. Pick a remaining problem, in this case `Unable to force merge`
5. Survey the functions exhibiting this problem, picking one of the smaller such functions, `get_next_arg`.
6. Export this function as an integration test, showing that it exhibits the same error repeatably.
7. Use Ghidra with the plugin enabled to inspect `get_next_arg` within the executable, patching various
   instructions into `nop` or `c.nop` until the decompiler completes with good transforms.

Continue by generating a true differential test:

1. Export the assembly listing to a new file `get_next_arg.S`
2. Remove Ghidra annotations to give vanilla Gnu assembler syntax
  a. replace `param_{n}` with `a{n-1}`
  b. remove leading address fields
  c. remove `=>...` annotations
3. Compile to a sharable object file and load into Ghidra with
   `riscv64-linux-gnu-gcc -march=rv64gcvzcb_zba -c get_next_arg.S -o get_next_arg.so`
4. Selectively patch the binary to isolate immaterial code and instructions triggering the fault
   * This suggests an interaction between the `vector_memcpy` transform and one or more subroutine
     calls in an adjacent block.
5. Wrap the assembly code in a macro with two arguments: a name suffix and a flags variable
6. Invoke the macro twice, once with a `_fail` suffix and flags producing the observed fault
   and once with a `_pass` suffix and flags enabling correct decompilation.
7. Compile to a sharable object file and import into Ghidra with the plugin enabled.  You should see
   two functions with and without the observed fault.
8. Close Ghidra and restart *without* an enabled plugin, opening `get_next_arg.so`.
9. Export both pass and fail versions of the function as debug XML files, `get_next_arg_pass.xml`
   and `get_next_arg_fail.xml`
10. Convert these into two datatest save files `get_next_arg_pass_save.xml` and `get_next_arg_fail_save.xml`,
    removing state information and aligning core type definitions.
11. Generate two datatest `.ghidra` files `get_next_arg_pass.ghidra` and `get_next_arg_fail.ghidra`  These
    should be identical except for the `*save.xml` file they load.
12. Review the `.ghidra` files and add any function signatures needed to capture function signatures.  In this
    case, we need to add a signture for our `operator.new` replacement to show a single input parameter.
13. Run the `pass` and `fail` datatests and observe the output.

The results were unexpected - both `pass` and `fail` datatests passed without triggering the fault.  The
original `get_next_arg` datatest, `whisper_sample_15.ghidra`, still fails.  The  `get_next_arg_fail` function
also still fails within the Ghidra GUI.

This simply shifts the differential test components to be `whisper_sample_15.ghidra` and `get_next_arg_fail.ghidra`,
with the understanding that `get_next_arg_fail.ghidra` is actually a passing example.

The differential analysis proceeds:

1. Edit the plugin source code to optionally dump its final PCode result at the end of each transform, whether
   or not an exception is thrown later.  Call these `pass.log` and `fail.log`.
2. Edit the logs to remove the logging timestamp clutter, then run both logs through `meld` or a similar graphical
   `diff` tool.

Observations:

* The `fail` sample has one fewer blocks, due to an explicit block removal in the `pass` source code.  Thus
  `fail` Block 2 disappears entirely in the `pass` test.
* The `fail` sample has multiple PcodeOps that have no analog in the `pass` sample:
    ```text
    r0x00142818(0x00037e7c:157) = r0x00142818(i) [] i0x00037e7c:3a(free)
    r0x00142818(0x00037e92:159) = r0x00142818(0x00037e8c:158) [] i0x00037e92:49(free)
    r0x00142818(0x00037e96:168) = r0x00142818(i) ? r0x00142818(0x00037e92:159)
    r0x00142818(0x00037e96:15d) = r0x00142818(0x00037e96:168) [] i0x00037e96:4c(free)
    r0x00142818(0x00037e9c:15e) = r0x00142818(0x00037e96:15d) [] i0x00037e9c:51(free)
    r0x00142818(0x00037ef6:15a) = r0x00142818(0x00037e9c:15e) [] i0x00037ef6:af(free)
    r0x00142818(0x00037f06:164) = r0x00142818(0x00037ef6:15a) [] i0x00037f06:bb(free)
    r0x00142818(0x00037f0a:165) = r0x00142818(0x00037f06:164) [] i0x00037f0a:be(free)
    r0x00142818(0x00037f30:161) = r0x00142818(0x00037e9c:15e) [] i0x00037f30:92(free)
    r0x00142818(0x00037f34:162) = r0x00142818(0x00037f30:161) [] i0x00037f34:95(free)
    r0x00142818(0x00037f38:163) = r0x00142818(0x00037f34:162) [] i0x00037f38:98(free)
    r0x00142818(0x00037f44:15c) = r0x00142818(0x00037f38:163)
    r0x00142818(0x00037f10:198) = r0x00142818(0x00037f10:198) ? r0x00142818(0x00037e9c:15e) ? r0x00142818(0x00037f0a:165)
    r0x00142818(0x00037ed2:167) = r0x00142818(0x00037e9c:15e) ? r0x00142818(0x00037f10:198)
    r0x00142818(0x00037ed2:15f) = r0x00142818(0x00037ed2:167) [] i0x00037ed2:6d(free)
    r0x00142818(0x00037ed8:160) = r0x00142818(0x00037ed2:15f) [] i0x00037ed8:71(free)
    r0x00142818(0x00037ee4:15b) = r0x00142818(0x00037ed8:160)
    ```
    * 0x00142818 is the address of `stderr`
    * 0x00037e7c is the address of the instruction `jal ra, printf`
    * 0x00037e8c is the address of the instruction `jal ra,gpt_print_usage`
    * 0x00037e92 is the address of the instruction `jal ra,exit`
    * 0x00037e96 is the address of the first instruction of Block 3, the Block preceding the vector_strlen transform
    * 0x00037e9c is the address of a `sd param_1, 0x0(t1)` instruction in the middle of Block 3
    * 0x00037ed2 is the address of a `sd a5, 0x8(t1)` instruction
    * 0x00037ed8 is the address of a `sb zero, 0(param1)` instruction
    * 0x00037ef6 is the address of the instruction `jal operator.new`
    * 0x00037f06 is the address of a `sd param_1, 0x0(t1)` instruction in the middle of Block 6
    * 0x00037f0a is the address of a `sd a5, 0x10(t1)` instruction at the end of Block 6
    * 0x00037ee4 is the address of a `ret` instruction
    * 0x00037f44 is the address of a `ret` instruction

Pivot the differential testing to compare the output of the whisper_sample_15 datatest with and without a plugin
* The r0x00142818 PcodeOps appear in the final printRaw output of both, suggesting they are benign.
* The i0x000* PcodeOps appear in the final printRaw output of both, suggesting they are benign.
* The largest unexpected difference appears to be the number of u0x PcodeOps removed

### core dump inspection

We need a different perspective here, especially since we appear to be throwing two different low level errors.

```console
$ SLEIGHHOME=/opt/ghidra_12.1_DEV/ DECOMP_PLUGIN=/tmp/libriscv_vector.so valgrind /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/whisper_sample_15.ghidra
==579060==
==579060== Process terminating with default action of signal 6 (SIGABRT): dumping core
==579060==    at 0x4CAD3CC: __pthread_kill_implementation (pthread_kill.c:44)
==579060==    by 0x4C5315D: raise (raise.c:26)
==579060==    by 0x4C3A6CF: abort (abort.c:77)
==579060==    by 0x48AC083: std::__glibcxx_assert_fail(char const*, int, char const*, char const*) (assert_fail.cc:41)
==579060==    by 0xA4AE41: std::vector<ghidra::BlockEdge, std::allocator<ghidra::BlockEdge> >::operator[](unsigned long) const (stl_vector.h:1282)
==579060==    by 0xA49D5E: ghidra::FlowBlock::getIn(int) const (block.hh:306)
==579060==    by 0xAB502B: ghidra::Cover::addRefPoint(ghidra::PcodeOp const*, ghidra::Varnode const*) (cover.cc:607)
==579060==    by 0xBC010C: ghidra::Merge::eliminateIntersect(ghidra::Varnode*, std::vector<ghidra::BlockVarnode, std::allocator<ghidra::BlockVarnode> > const&) (merge.cc:505)
==579060==    by 0xBC0751: ghidra::Merge::unifyAddress(std::_Rb_tree_const_iterator<ghidra::Varnode*>, std::_Rb_tree_const_iterator<ghidra::Varnode*>) (merge.cc:600)
==579060==    by 0xBC0955: ghidra::Merge::mergeAddrTied() (merge.cc:632)
==579060==    by 0xA9EE15: ghidra::ActionMergeRequired::apply(ghidra::Funcdata&) (coreaction.hh:370)
==579060==    by 0xA1BBFA: ghidra::Action::perform(ghidra::Funcdata&) (action.cc:319)
==579060==
$ ls -lt vgcore*
-rw-------. 1 62705664 Apr 29 08:12 vgcore.579060
...
$ gdb /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest vgcore.579060
(gdb) bt
#0  __pthread_kill_implementation (threadid=<optimized out>, signo=signo@entry=6, no_tid=no_tid@entry=0) at pthread_kill.c:44
#1  0x0000000004cad493 in __pthread_kill_internal (threadid=<optimized out>, signo=6) at pthread_kill.c:89
#2  0x0000000004c5315e in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
#3  0x0000000004c3a6d0 in __GI_abort () at abort.c:77
#4  0x00000000048ac084 in std::__glibcxx_assert_fail (file=<optimized out>, line=<optimized out>, function=<optimized out>, condition=<optimized out>) at ../../../../../libstdc++-v3/src/c++11/assert_fail.cc:41
#5  0x0000000000a4ae42 in std::vector<ghidra::BlockEdge, std::allocator<ghidra::BlockEdge> >::operator[] (this=0x7155a88, __n=2) at /usr/lib/gcc/x86_64-redhat-linux/15/../../../../include/c++/15/bits/stl_vector.h:1282
#6  0x0000000000a49d5f in ghidra::FlowBlock::getIn (this=0x7155a50, i=2) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/block.hh:306
#7  0x0000000000ab502c in ghidra::Cover::addRefPoint (this=0x1ffefff330, ref=0x74da920, vn=0x721fa80) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/cover.cc:607
#8  0x0000000000bc010d in ghidra::Merge::eliminateIntersect (this=0x70eb118, vn=0x721fa80, blocksort=std::vector of length 19, capacity 19 = {...}) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/merge.cc:505
#9  0x0000000000bc0752 in ghidra::Merge::unifyAddress (this=0x70eb118, startiter=0x74a9460, enditer=0x717d630) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/merge.cc:600
#10 0x0000000000bc0956 in ghidra::Merge::mergeAddrTied (this=0x70eb118) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/merge.cc:632
#11 0x0000000000a9ee16 in ghidra::ActionMergeRequired::apply (this=0x70a22d0, data=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/coreaction.hh:370
#12 0x0000000000a1bbfb in ghidra::Action::perform (this=0x70a22d0, data=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/action.cc:319
#13 0x0000000000a1c786 in ghidra::ActionGroup::apply (this=0x7087f10, data=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/action.cc:514
#14 0x0000000000a1ca36 in ghidra::ActionRestartGroup::apply (this=0x7087f10, data=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/action.cc:560
#15 0x0000000000a1bbfb in ghidra::Action::perform (this=0x7087f10, data=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/action.cc:319
#16 0x000000000090c1dd in ghidra::IfcDecompile::execute (this=0x5283be0, s=...) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/ifacedecomp.cc:908
#17 0x0000000000936d1b in ghidra::IfaceStatus::runCommand (this=0x52815e0) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/interface.cc:369
#18 0x000000000091e082 in ghidra::execute (status=0x52815e0, dcp=0x5281830) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/ifacedecomp.cc:3620
#19 0x000000000091e4d2 in ghidra::mainloop (status=0x52815e0) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/ifacedecomp.cc:3661
#20 0x00000000008f3692 in main (argc=1, argv=0x1ffefffbf8) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/consolemain.cc:234
```

The parameters to function `ghidra::Cover::addRefPoint` show us the PcodeOp and Varnode causing the exception:

```text
(gdb) frame 7
#7  0x0000000000ab502c in ghidra::Cover::addRefPoint (this=0x1ffefff330, ref=0x74da920, vn=0x721fa80) at external/+_repo_rules+ghidra/Ghidra/Features/Decompiler/src/decompile/cpp/cover.cc:607
(gdb) print *ref
$21 = {opcode = 0x70e6690, flags = 131152, addlflags = 0, start = {pc = {base = 0x5c87430, offset = 229136}, uniq = 408, order = 65538}, parent = 0x7155a50, basiciter = 0x74da920, insertiter = 0x74da920, codeiter = non-dereferenceable iterator for std::list, output = 0x74da6d0,
  inrefs = std::vector of length 3, capacity 4 = {0x74da6d0, 0x757f790, 0x721fa80}}
(gdb) p/x ref->start.pc.offset
$23 = 0x37f10
(gdb) p/x ref->start.uniq
$24 = 0x198
(gdb) p/x ref->output->loc.offset
$28 = 0x142818

(gdb) print *vn
$22 = {flags = 19456048, size = 8, create_index = 1138, mergegroup = 0, addlflags = 0, loc = {base = 0x5c87430, offset = 1320984}, def = 0x721f8a0, high = 0x7578ce0, mapentry = 0x0, type = 0x74deae0, lociter = 0x721fa80, defiter = 0x721fa80, descend = std::__cxx11::list = {[0] = 0x74da920},
  cover = 0x7578c70, temp = {dataType = 0x74deae0, valueSet = 0x74deae0}, consumed = 18446744073709551615, nzm = 18446744073709551615}
(gdb) p/x vn->loc.offset
$29 = 0x142818
(gdb) p/x vn->def->start.pc.offset
$32 = 0x37f0a
(gdb) p/x vn->def->start.uniq
$33 = 0x165
```

The log file helps us identify these elements

* The PcodeOp* idenfied as `ref` is
    ```text
    Basic Block 10 0x00037f10-0x00037f22
    0x00037f10:198: r0x00142818(0x00037f10:198) = r0x00142818(0x00037f10:198) ? r0x00142818(0x00037e9c:15e) ? r0x00142818(0x00037f0a:165)
    ```
* The Varnode* identified as `vn` is defined at
    ```text
    0x00037f0a:165: r0x00142818(0x00037f0a:165) = r0x00142818(0x00037f06:164) [] i0x00037f0a:be(free)
    ```

That helps refine our differential testing:
* we definitely want to include the code referencing `stderr` at 0x00142818
* we want to understand indirect copy PcodeOps containing the `[]` operator
* we want to focus attention on varnodes like `i0x00037f0a:be(free)`

### iterate differential testing to two dimensions

Modify `get_next_arg.S` to generate three variants of our problem sample, minimizing the differences between those variants.
For example, make all three variants start at the same address and with matching instructions placed at matching addresses.
* The base code remains in `get_next_arg.S`, with the macro expansions moved to `get_next_arg_test_1.S`, `get_next_arg_test_2.S`,
  and `get_next_arg_test_3.S`.
* The base code is modified such that any instructions removed are replaced by an equivalent span of `c.nop` instructions.
* The external functions like `fprintf` are replaced with local dummy functions

1. `get_next_arg_test_1` - throws `Low-level Error: Could not find address-tied varnode`
    * Enables the error exit blocks including reference to `stderr` and function calls to `fprintf` and `exit`.
    * Enables the function call to `dummy_new`
2. `get_next_arg_test_2` - throws `Low-level Error: Unable to force merge of op at 0x001000e0:159`
    * Disables the error exit blocks referencing `stderr`.
    * Enables the function call to `dummy_new`
3. `get_next_arg_test_3` - decompiles correctly with no visible errors in the Ghidra GUI
    * Disables the error exit blocks referencing `stderr`.
    * Disables the function call to `dummy_new`

Run all three through the datatest sequence, generating three instances of `ghidraRiscvLogger_{pid}.log` and inspect the differences using `meld`.

The logs are very similar, with no obvious differences that help refine the problem.  Perhaps the logs are missing something critical?

### expand the diagnostic commands

Browse through the source code to locate additional datatest commands we can use, especially ones that might show missing detail on varnodes.

For example, decompile `get_next_arg_test_2` without a plugin, running the baseline commands and then trying some new ones:

```
decomp]> restore test_data/get_next_arg_test_2_save.xml
test_data/get_next_arg_test_2_save.xml successfully loaded: RISC-V 64 little default
[decomp]> map function 0x100000 get_next_arg_test_2
[decomp]> parse line extern ulong* get_next_arg_test_2(int *param_1,int param_2,char** param_3,void *param_4,void *param_5);
[decomp]> map function 0x100118 dummy_new
[decomp]> parse line extern ulong* dummy_new(ulong size);
[decomp]> load function get_next_arg_test_2
Function get_next_arg_test_2: 0x00100000
[decomp]> decompile get_next_arg_test_2
Decompiling get_next_arg_test_2
Decompilation complete
...
Basic Block 10 0x001000e0-0x001000f2
0x001000e0:175:	r0x0020a9e8(0x001000e0:175) = r0x0020a9e8(0x001000e0:175) ? r0x0020a9e8(0x0010006c:148) ? r0x0020a9e8(0x001000da:14f)
0x001000e0:124:	a6(0x001000e0:124) = a6(0x001000ea:81) ? a6(0x00100066:127) ? a6(0x00100066:127)
0x001000e0:10d:	a3(0x001000e0:10d) = u0x1000007d(0x001000f2:17f) ? u0x100000b5(0x00100096:186) ? u0x100000b5(0x00100096:186)
0x001000e0:fc:	a0(0x001000e0:fc) = a0(0x001000f0:19d) ? a0(0x00100068:18e) ? a0(0x001000b6:191)
0x001000e0:7d:	a4(0x001000e0:7d) = vsetvli_e8m1tama(a3(0x001000e0:10d))
0x001000e4:7f:	v1(0x001000e4:7f) = vle8_v(a6(0x001000e0:124))
...
[decomp]>  print inputs
Function: get_next_arg_test_2
r0x0020a9e8(i)    puRam000000000020a9e8     nontriv
a0(i)    param_1     nontriv
a1:4(i)    param_2     nontriv
a2(i)    param_3     nontriv
a3(i)    param_4     nontriv
a4(i)    param_5     nontriv
a5(i)    in_a5     nontriv
c0x0c20(i)    vl     nontriv
[decomp]>  print varnode r0x0020a9e8(0x001000e0:175)
Variable: puRam000000000020a9e8
Type: undefined8 *

0: undefined8 * = r0x0020a9e8(i) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399047c0) (create=0x4a9)
0: undefined8 * = r0x0020a9e8(0x0010004c:142) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399b6400) (create=0x408)
0: undefined8 * = r0x0020a9e8(0x0010005c:143) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399b7490) (create=0x40b)
0: undefined8 * = r0x0020a9e8(0x00100062:144) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bcb30) (create=0x40e)
0: undefined8 * = r0x0020a9e8(0x00100066:147) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399bce70) (create=0x415)
0: undefined8 * = r0x0020a9e8(0x00100066:152) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399bbec0) (create=0x435)
0: undefined8 * = r0x0020a9e8(0x0010006c:148) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399bdf30) (create=0x418)
0: undefined8 * = r0x0020a9e8(0x001000a2:149) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399bd390) (create=0x41b)
0: undefined8 * = r0x0020a9e8(0x001000a2:151) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399b14a0) (create=0x432)
0: undefined8 * = r0x0020a9e8(0x001000a8:14a) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x398aba10) (create=0x41e)
0: undefined8 * = r0x0020a9e8(0x001000b4:145) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bc0b0) (create=0x410)
0: undefined8 * = r0x0020a9e8(0x001000d6:14e) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399aa370) (create=0x42a)
0: undefined8 * = r0x0020a9e8(0x001000da:14f) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399b50e0) (create=0x42d)
0: undefined8 * = r0x0020a9e8(0x001000e0:175) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x39900f00) (create=0x4b6)
0: undefined8 * = r0x0020a9e8(0x00100100:14b) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399bf420) (create=0x421)
0: undefined8 * = r0x0020a9e8(0x00100104:14c) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399c00b0) (create=0x424)
0: undefined8 * = r0x0020a9e8(0x00100108:14d) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399ab430) (create=0x427)
0: undefined8 * = r0x0020a9e8(0x00100114:146) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399ba100) (create=0x412)
[decomp]>  print varnode c0x0c20(0x001000b4:156)
Variable: vl
Type: long

0: long = c0x0c20(i) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399a5e30) (create=0x4a8)
0: long = c0x0c20(0x0010004c:153) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399b7e10) (create=0x439)
0: long = c0x0c20(0x0010005c:154) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bc400) (create=0x43c)
0: long = c0x0c20(0x00100062:155) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x39904c30) (create=0x43f)
0: long = c0x0c20(0x00100066:158) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x39905360) (create=0x445)
0: long = c0x0c20(0x001000b4:156) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bd5d0) (create=0x441)
0: long = c0x0c20(0x00100114:157) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bc940) (create=0x443)

[decomp]> print high vl
Variable: vl
Type: long

0: long = c0x0c20(i) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x399a5e30) (create=0x4a8)
0: long = c0x0c20(0x0010004c:153) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399b7e10) (create=0x439)
0: long = c0x0c20(0x0010005c:154) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bc400) (create=0x43c)
0: long = c0x0c20(0x00100062:155) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x39904c30) (create=0x43f)
0: long = c0x0c20(0x00100066:158) tied mapped persistent (consumed=0xffffffffffffffff) (internal=0x39905360) (create=0x445)
0: long = c0x0c20(0x001000b4:156) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bd5d0) (create=0x441)
0: long = c0x0c20(0x00100114:157) tied mapped persistent addrforce (consumed=0xffffffffffffffff) (internal=0x399bc940) (create=0x443)

[decomp]> print cover high vl
0: begin-end
1: begin-end
2: begin-end
3: begin-end
4: begin-end
5: begin-end
6: begin-end
7: begin-end
8: begin-0x00100114:157
9: begin-end
10: begin-end
11: begin-end
12: begin-0x001000b4:156
```

Note:
* HighVariables are only available if FuncData::isHighOn returns true, that is if `FuncData::flags&highlevel_on` is non-zero.
  That is probably not true during our plugin execution.
* We should add `vn->printInfo(ss)` and `vn->printCover(ss)` in places we use `vn->printRaw(ss)`.  `printInfo` helps somewhat, but `printCover`
  fails since it is generated in an Action that follows our plugin's execution.

### back to code inspection with gdb

Run test_2 under gdb and localize the error thrown.  The code suggests that `Cover::addRefPoint` expects that MULTIEQUAL Pcodes
at the beginning of a block will have one input Varnode for each of the inward edges.

```c
void Cover::addRefPoint(const PcodeOp *ref,const Varnode *vn)
{
    ...
    bl = ref->getParent();
    ...
    if (ref->code() == CPUI_MULTIEQUAL) {
    for(j=0;j<ref->numInput();++j)
      if (ref->getIn(j)==vn)
	    addRefRecurse(bl->getIn(j));
}

```

Gdb shows that the `ref` op is:

    0x001000e0:175: r0x0020a9e8(0x001000e0:175) = r0x0020a9e8(0x001000e0:175) ? r0x0020a9e8(0x0010006c:148) ? r0x0020a9e8(0x001000da:14f)

Perhaps the self-reference in slot 0 should be removed? Yes - that correction reduces the number of errors significantly.

### retest all whisper.cpp functions

All current test cases pass, while the whole-binary export of whisper-cpp shows 8 remaining errors:

| function | size (bytes) |  error |
| -------- | ------- | ----- |
| main | 10844 | Low-level Error: Unable to force merge of op at 0x000230c8:188cc |
| output_wts | 7056 | Low-level Error: Unable to force merge of op at 0x00064a9a:7bfb |
| whisper_bench_ggml_mul_mat_str | 3922 | Low-level Error: Unable to force merge of op at 0x000bb242:37fc |
| whisper_bench_memcpy_str | 3636 | Low-level Error: Unable to force merge of op at 0x000bd93c:25d1 |
| whisper_process_logits | 5954 | Low-level Error: Unable to force merge of op at 0x000c366a:5dfd |
| __do_str_codecvt... | 1052 | Low-level Error: Unable to force merge of op at 0x000cf652:921 |
| ggml_backend_cpu_device_context | 586 | Low-level Error: Free varnode has multiple descendants |
| drwav__metadata_process_chunk | 6586 |  Exception while decompiling ram:00032aee: Decompiler process died |

>Note: we don't yet know the *correct* number of transforms in whisper.cpp.

| transform | reported instances |
| --------- | ------------------ |
| vector_memset | 403 |
| vector_memcpy | 1032 |
| vector_strlen | 69 |
| vector_strcmp | 15 |

We've got the failure rate down to 0.5%, but the main routine is still failing when part of the entire function,
so we can't claim the code is good enough.  We need at least one more round of debugging.

Observations:
* The six 'force merge' errors look very similar to earlier force-merge errors, occuring at the start of a
  `vector_memcpy` stanza nested within other loops and blocks
* The 'free varnode' error occurs in a smallish function, so we can likely resolve it fairly quickly.
* The 'Exception while decompiling' error likely merits a dedicated datatest

Add some ugly code to Inspector so that the audit performs an 'autocorrect' if blocks contain MULTIEQUAL PcodeOps
with too many input Varnodes.

| function | size (bytes) |  error |
| -------- | ------- | ----- |
| whisper_bench_memcpy_str | 3636 | Low-level Error: Could not find address-tied varnode |
| ggml_backend_cpu_device_context | 586 | Low-level Error: Free varnode has multiple descendants |
| drwav__metadata_process_chunk | 6586 |  Exception while decompiling ram:00032aee: Decompiler process died |

| transform | reported instances |
| --------- | ------------------ |
| vector_memset | 403 ⇒ 413|
| vector_memcpy | 1032 ⇒ 1094|
| vector_strlen | 69 ⇒ 83|
| vector_strcmp | 15 ⇒15 |
