---
title: Challenges
description: Vectorizing compilers can produce hard-to-understand code.  Can we help Ghidra help us to understand it?
weight: 20
---

You might think that this function would be easy to recognize after compiled with a vectorizing compiler.

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
It simply copies one array into another, in reverse order.  Compile it with O3 optimization and vectorization enabled, load it into
Ghidra, and edit the function signature to match the source.

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

* if there are fewer than 14 8-byte elements to move, use the simple scalar loop at the end.
* otherwise, use a vector loop to process whole vector registers followed by a different simple scalar loop
  to finish any tail elements.
    * `vlenb` is a control and status register holding the length of vector registers, measured in bytes.
      If the vector length is 128 bits, then `vlenb` is 16.  If it is 1024 bits, then `vlenb` is 128.  It is entirely possible
      for a single processor to have cores with different values of `vlenb`
    * the `vid_v` and `vrsub_vx` help generate a vector of permutations
    * the `vl1re64_v` and `vs1r_v` load and store entire vector registers.  The `vrgather_vv` instruction applies the permutation
      vector to the loaded results

The compiler would have generated different code if it knew what `vlenb` and `size` were at compile time.

An algorithmic Ghidra plugin probably couldn't translate the decompiled results back into the simple scalar source loop.
An AI plugin could *possibly* recognize the overall block structure and specific vector operations to converge on source code
and toolchain parameters that compiles into a match.