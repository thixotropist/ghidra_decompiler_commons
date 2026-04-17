---
title: Challenges
description: Vectorizing compilers can produce hard-to-understand code.  Can we help Ghidra help us to understand it?
weight: 20
---

## A simple/not-simple loop

You might think that this function would be easy to recognize and analyze in Ghidra after it
is compiled with a vectorizing compiler.

```c
void test_1_ref(unsigned long long *in, unsigned long long *out, unsigned int size)
{
    int i;
    int upper_index = size - 1;
    for (i=0; i < size; i++) {
        out[i] = in[upper_index - i];
    }
}
```

It simply copies one array into another, in reverse order.
Compile it with O3 optimization and vectorization enabled, load it into Ghidra, and edit the function
signature to match the source.  The simple loop is no longer simple.

```c
void test_1_ref(ulonglong *in,ulonglong *out,ulong size)
{
  int iVar1;
  ulonglong *puVar2;
  ulonglong *puVar3;
  ulonglong uVar4;
  ulong uVar5;
  long lVar6;
  ulonglong *puVar7;
  ulong uVar8;
  undefined1 auVar9 [32];
  undefined1 auVar10 [32];

  if (size != 0) {
    uVar5 = (ulong)((int)size + -1);
    if (0xc < uVar5) {
      if (((ulong)(long)((int)(vlenb >> 3) + -1) <= uVar5) &&
         ((out + (size & 0xffffffff) <= in + ((uVar5 + 1) - (size & 0xffffffff)) ||
          (in + uVar5 + 1 <= out)))) {
        vsetvli_e64m1tama(0);
        auVar10 = vid_v();
        auVar10 = vrsub_vx(auVar10,(vlenb >> 3) - 1);
        lVar6 = ((uVar5 * 8 + 8) - vlenb) + (long)in;
        iVar1 = (int)(vlenb >> 3);
        uVar8 = 0;
        puVar3 = out;
        do {
          auVar9 = vl1re64_v(lVar6);
          uVar8 = (ulong)((int)uVar8 + iVar1);
          lVar6 = lVar6 - vlenb;
          auVar9 = vrgather_vv(auVar9,auVar10);
          vs1r_v(auVar9,puVar3);
          puVar3 = (ulonglong *)((long)puVar3 + vlenb);
        } while (uVar8 <= (ulong)(long)((int)size - iVar1));
        if (size == uVar8) {
          return;
        }
        puVar7 = in + (uVar5 - uVar8);
        puVar3 = out + uVar8;
        do {
          uVar4 = *puVar7;
          uVar8 = (ulong)((int)uVar8 + 1);
          puVar7 = puVar7 + -1;
          *puVar3 = uVar4;
          puVar3 = puVar3 + 1;
        } while (uVar8 < size);
        return;
      }
    }
    puVar7 = in + uVar5;
    puVar3 = out;
    do {
      uVar4 = *puVar7;
      puVar2 = puVar3 + 1;
      puVar7 = puVar7 + -1;
      *puVar3 = uVar4;
      puVar3 = puVar2;
    } while (puVar2 != out + (size & 0xffffffff));
  }
  return;
}
```

The compiler has chosen to vectorize this loop in a surprising way.

* If there are fewer than 14 8-byte elements to move, use the simple scalar loop at the end and avoid
  the vectorized code altogether.
* Otherwise, use a vector loop to process *whole* vector registers followed by a different simple
  scalar loop to finish any tail elements not filling an entire vector register.
    * `vlenb` is a control and status register holding the length of the current hardware thread's
      vector registers, measured in bytes. If the vector length is 128 bits, then `vlenb` is 16.
      If it is 1024 bits, then `vlenb` is 128.  It is entirely possible for a single processor to have cores with different values of `vlenb`.
    * The `vid_v` and `vrsub_vx` help generate a vector of permutations.
    * The `vl1re64_v` and `vs1r_v` load and store entire vector registers.
      The `vrgather_vv` instruction applies the permutation vector to the loaded results

The compiler would have generated different code if it knew what `vlenb` and `size` were at compile time.
These values can either be discovered at runtime, as in the code above, or hard-coded into the toolchain and
C source.

An algorithmic Ghidra plugin probably couldn't translate the decompiled results back into the simple scalar source loop.
A survey or AI plugin *could* recognize the overall block structure and specific vector operations to
converge on source code and toolchain parameters that compiles into a match.  The Survey plugin would
recognize the special nature of `vl1re64_v`, `vs1r_v`, and `vlenb` as indicating multiple code paths, then
add decompiler comments to suggest the user examine just the much simpler `0x0c < (size - 1)` path.

## How do we represent transformed code?

Once the decompiler has analyzed a function's Pcode and Block structure, it has to display it in the GUI's
Decompiler window.  What form should that display take?

### We could transform non-loop sequences of vector loads and stores into ...

The compilers will often combine sequences of similar scalar load and store instructions into
a smaller number of vector load and store instructions.  These loads and stores likely span several
variables and fields.

We might want to break these vector sequences back into their scalar equivalents, with vanilla Pcode Ghidra
can process normally.  If we do that, would we need to recognize and transform those sequences *before* the
Heritage actions trigger?

### We could transform vector loop sequences into ...

* function calls similar to those in the C standard library
* function calls used internally by compilers like `__builtin_memcpy` or the gcc RTL `cpymem`.
* libstdc++ vector function calls
* function calls similar to those in the C++ `algorithm` collection, such as `std::transform`, `std::map`,
  and `std::for_each`.  This very likely implies support for C++ lambda expression to capture the scalar
  kernel of a vectorized operation.

## New intrinsic data types

AI and Inference Engine apps often use fractional floating point data types.  RISC-V instruction
extensions exist for *two* different 16 bit floating point types.

Should the decompiler be able to recognize these and treat them properly within the Ghidra Type systems?
