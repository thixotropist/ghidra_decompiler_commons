---
title: Decompiler Overview
description: The Ghidra decompiler is a c++ program responsible for turning binary instructions into C-like displays
weight: 10
---

The decompiler converts a binary executable into something resembling C. It uses the SLEIGH processor language definitions
to convert machine instructions into processor independent Pcode, then follows branch, call, and return instructions to group
that Pcode into blocks, accepts context and constraints from the Ghidra GUI, then recasts the blocks of Pcode into something
very much like C.

The decompiler does a great job, but it doesn't evolve very quickly.

Key decompiler external features include:

* the source code is mostly within `.../Ghidra/Features/Decompiler/src/decompile/cpp`.
  It amounts to 134K lines of very nicely documented code.
* the executable binary - on a linux host - will be found at `.../Ghidra/Features/Decompiler/os/linux_x86_64/decompile`.  A standalone
  `decompile_datatest` exists in the same directory.  This is incredibly useful to anyone seeking to extend the decompiler without
  trying a full Ghidra rebuild.
* The decompiler requires a processor language definition and works operates on one function at a time.  The Ghidra GUI will usually
  start a decompiler process by giving it the binary image, an Architecture definition (e.g., x86_64), a compiled SLEIGH language definition, and whatever is currently known about that function.  This include type assignments, variable names, function signatures,
  and structure definitions.
* The decompiler processes run as single-threaded agents.  In most cases the Ghidra GUI launches a single agent as the users
  examine a single function at a time.  If an entire program is exported to C/C++ source code, then multiple decompiler process
  agents will be launched with no lateral communication between the agents.

Internally, the decompiler relies on per-processor SLEIGH definitions to turn machine instructions into Pcode.  An integer addition
instruction will turn the instruction at 0x00000050 emitted by `c.sub a2,a3` and assembed into `0x15 0x8e` into Pcode like:

```text
0x00000050:3:	a2(0x00000050:3) = a2(0x00000048:e) - a3(0x00000048:0)
```

This `PcodeOp` has these components:
* an address of `0x00000050` and a sequence number of `3`
* The opcode `CPUI_INT_SUB`, for signed or unsigned integer subtraction
* two Varnodes as input `a2(0x00000048:e)` and `a3(0x00000048:0)`
* one Varnode as output `a2(0x00000050:3)`.

The decompiler uses [Static Single Assignment](https://en.wikipedia.org/wiki/Static_single-assignment_form) internally,
so register Varnodes are identified by their register names (like `a2` or `a3`) the address at which they are singly assigned
(like `0x00000048` or `0x00000050`), and sequence number (or `time` field) showing the exact step at which it was assigned.

The decompiler can now translate this Pcode into a line in the Decompiler window like `lVar1 = lVar1 - lVar2`.

If machine instructions can be represented in the SLEIGH language definition files, this works very well.
That's not always the case, opening up opportunities to extend the decompiler with new capabilities.

For instance, The Gnu Compiler Collection (`gcc`) will often recognize byte copy sequences and convert them into `__builtin_memcpy` calls.
If the compilation targets a RISC-V processor supporting the vector extensions, then this can generate assembly code like this:

```as
memcpy_v1:
00100048 d7 76 06 0c     vsetvli  a3,size,e8,m1,ta,ma  
0010004c 87 80 05 02     vle8.v   v1,(src)
00100050 15 8e           c.sub    size,a3
00100052 36 95           c.add    dest,a3
00100054 a7 00 05 02     vse8.v   v1,(dest)
00100058 b6 95           c.add    src,a3
0010005a 7d f6           c.bnez   size,memcpy_v1
```

The decompiler window has useful SLEIGH semantics for four out of seven instructions, and displays this in the decompiler window:

```c
  do {
    lVar1 = vsetvli_e8m1tama(size);
    auVar2 = vle8_v(src);
    size = size - lVar1;
    dest = (void *)((long)dest + lVar1);
    vse8_v(auVar2,dest);
    src = (void *)((long)src + lVar1);
  } while (size != 0);
```

Maybe we would rather see this instead:

```c
__builtin_memcpy(dest, src, size);
```

>Note: `__builtin_memcpy` is the visible version of gcc's generic `memcpy` pattern.  Internal to the gcc compiler the RTL name is
>      `cpymem`.  See the gcc source file `gcc/doc/md.texi` for more examples.
