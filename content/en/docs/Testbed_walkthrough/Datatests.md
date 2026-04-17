---
title: A first data test
description: Data tests feed chunks of binary data into the decompiler along with function context, producing C and raw Pcode to review.
weight: 20
---

>Note: Data tests can be chosen to drive development or check for regressions.  They are chosen to challenge some
>      current Ghidra behavior, such as lack of clarity in the decompiler C display.

The Ghidra decompiler normally acts on commands, data, and context received from the Ghidra main Java code via a pipe or
Unix socket. In this mode it uses the `decompile` executable built in the
[previous page]({{% relref "Baseline.md" %}}) as agent.  Ghidra also provides `decompile_datatest` to
drive the decompiler code from interactive or scripted input.
See [Data Tests]({{% relref "/docs/ghidra_decompiler_internals/Data_tests.md" %}}) for more details.


The decompiler datatest is launched via a command:

```console
SLEIGHHOME=/opt/ghidra_12.1_DEV/ /opt/ghidra_12.1_DEV/Ghidra/Features/Decompiler/os/linux_x86_64/decompile_datatest < test_data/memcpy_exemplars.ghidra
decomp]> restore test_data/memcpy_exemplars_save.xml
test_data/memcpy_exemplars_save.xml successfully loaded: RISC-V 64 little default
[decomp]> map function 0x00000 memcpy_i2
[decomp]> map function 0x0000e memcpy_i4
[decomp]> map function 0x00022 memcpy_i8
[decomp]> map function 0x00036 memcpy_i15
[decomp]> map function 0x00048 memcpy_v1
...
[decomp]>
```

* The *deployed* Ghidra root must be identified in the environment using the `SLEIGHHOME` variable.  This allows
  the `decompile_datatest` executable to find Processor SLEIGH files.  It also lets us test variants that rely on changes
  to the `Ghidra/Processors` directory tree under the Ghidra root.
* The `decompile_datatest` executable is identified, possibly preceded by `valgrind` or maybe `gdb`
* The decompiler test script is piped into the decompiler.

Our first test script is:

```text
restore test_data/memcpy_exemplars_save.xml
map function 0x00000 memcpy_i2
map function 0x0000e memcpy_i4
map function 0x00022 memcpy_i8
map function 0x00036 memcpy_i15
map function 0x00048 memcpy_v1
parse line extern void memcpy_i2(void* to, void* from);
parse line extern void memcpy_i4(void* to, void* from);
parse line extern void memcpy_i8(void* to, void* from);
parse line extern void memcpy_i15(void* to, void* from);
parse line extern void memcpy_v1(void* to, void* from, long size);
load function memcpy_i2
decompile memcpy_i2
print C
print raw
load function memcpy_i4
decompile memcpy_i4
print C
print raw
load function memcpy_i8
decompile memcpy_i8
print C
print raw
load function memcpy_i15
decompile memcpy_i15
print C
print raw
load function memcpy_v1
decompile memcpy_v1
print C
print raw
```

This script:
* identifies the Ghidra save file as `test_data/memcpy_exemplars_save.xml`
* gives the entry address of the five functions present in this save file.
* gives function signatures for the five functions, using types defined in the save file
* provides a sequence of `load`, `decompile`, `print C`, and `print raw` commands for each of the defined
  functions.

The file `test_data/memcpy_exemplars_save.xml` looks like this:

```text
<xml_savefile name="main" target="default" adjustvma="0">
    <!-- A binary image defines the architecture and one or more chunks of bytes -->
    <binaryimage arch="RISCV:LE:64:default">
        <bytechunk space="ram" offset="0x0" readonly="true">
            577051cc87800502a700050282805770
            62cc01000100878005020100a7000502
            8280577074cc01000100878005020100
            a7000502828057f007cc878005020100
            0100a70005028280d776060c87800502
            158e3695a7000502b6957df682805a00
        </bytechunk>
    </binaryimage>
    <!-- core type definitions come from function debug xml files.  These appear to be the basic primitive types -->
    <coretypes>
      <type name="code" size="1" metatype="code" id="0xe000000000000001"/>
      <type name="bool" size="1" metatype="bool" id="0xc000000000000079"/>
      <type name="byte" size="1" metatype="uint" id="0xc00000000000007a"/>
      <type name="char" size="1" metatype="int" char="true" id="0xc00000000000007b"/>
      <type name="double" size="8" metatype="float" id="0xc000000000000082"/>
      <type name="float10" size="10" metatype="float" id="0xc000000000000084"/>
      <type name="float16" size="16" metatype="float" id="0xc000000000000085"/>
      <type name="float2" size="2" metatype="float" id="0xc000000000000086"/>
      <type name="float" size="4" metatype="float" id="0xc00000000000008a"/>
      <type name="int16" size="16" metatype="int" id="0xc000000000000090"/>
      <type name="int3" size="3" metatype="int" id="0xc000000000000091"/>
      <type name="int5" size="5" metatype="int" id="0xc000000000000092"/>
      <type name="int6" size="6" metatype="int" id="0xc000000000000093"/>
      <type name="int7" size="7" metatype="int" id="0xc000000000000094"/>
      <type name="int" size="4" metatype="int" id="0xc000000000000095"/>
      <type name="long" size="8" metatype="int" id="0xc000000000000097"/>
      <type name="short" size="2" metatype="int" id="0xc0000000000000a8"/>
      <type name="sbyte" size="1" metatype="int" id="0xc0000000000000a9"/>
      <type name="undefined1" size="1" metatype="unknown" id="0xc0000000000000b4"/>
      <type name="undefined2" size="2" metatype="unknown" id="0xc0000000000000b5"/>
      <type name="undefined3" size="3" metatype="unknown" id="0xc0000000000000b6"/>
      <type name="undefined4" size="4" metatype="unknown" id="0xc0000000000000b7"/>
      <type name="undefined5" size="5" metatype="unknown" id="0xc0000000000000b8"/>
      <type name="undefined6" size="6" metatype="unknown" id="0xc0000000000000b9"/>
      <type name="undefined7" size="7" metatype="unknown" id="0xc0000000000000ba"/>
      <type name="undefined8" size="8" metatype="unknown" id="0xc0000000000000bb"/>
      <type name="uint16" size="16" metatype="uint" id="0xc0000000000000bf"/>
      <type name="uint3" size="3" metatype="uint" id="0xc0000000000000c0"/>
      <type name="uint5" size="5" metatype="uint" id="0xc0000000000000c1"/>
      <type name="uint6" size="6" metatype="uint" id="0xc0000000000000c2"/>
      <type name="uint7" size="7" metatype="uint" id="0xc0000000000000c3"/>
      <type name="uint" size="4" metatype="uint" id="0xc0000000000000c4"/>
      <type name="ulong" size="8" metatype="uint" id="0xc0000000000000c6"/>
      <type name="ushort" size="2" metatype="uint" id="0xc0000000000000c8"/>
      <type name="void" size="0" metatype="void" id="0xc0000000000000c9"/>
      <type name="wchar32" size="4" metatype="int" utf="true" id="0xc0000000000000cc"/>
      <type name="wchar_t" size="2" metatype="int" utf="true" id="0xc0000000000000cd"/>
    </coretypes>
</xml_savefile>
```

The `<xml_savefile>` schema is very similar to what the `Debug function decompilation` button gives you in the
decompiler window.  For this example we've combined the binaries of all five functions into a single `bytechunk` element.

>Note: This savefile started with the RISC-V assembly file `test_data/memcpy_exemplars.S`, which was assembled and linked
>      into `test_data/memcpy_exemplars.so` and exported to binary to fill out the `<bytechunk`> XML element.

The source for the first function `memcpy_i2` is:

```as
.section .text

# copy fixed 2 bytes
.extern memcpy_i2
memcpy_i2:
    vsetivli zero,0x2,e8,mf8,ta,ma
    vle8.v   v1,(a1)
    vse8.v   v1,(a0)
    ret
```

The decompile_datatest output for this function is:

```text
...
[decomp]> load function memcpy_i2
Function memcpy_i2: 0x00000000
[decomp]> decompile memcpy_i2
Decompiling memcpy_i2
Decompilation complete
[decomp]> print C

void memcpy_i2(void *to,void *from)

{
  undefined1 auVar1 [32];

  vsetivli_e8mf8tama(2);
  auVar1 = vle8_v(from);
  vse8_v(auVar1,to);
  return;
}
[decomp]> print raw
0
Basic Block 0 0x00000000-0x0000000c
0x00000000:1:	vsetivli_e8mf8tama(#0x2:5)
0x00000004:3:	v1(0x00000004:3) = vle8_v(a1(i))
0x00000008:5:	vse8_v(v1(0x00000004:3),a0(i))
0x0000000c:6:	return(#0x0)
...
```

The `print C` command generates exactly what you would find in the Ghidra `Decompiler` window.  The `print raw` command
shows the intermediate [Pcode]({{% relref "/docs/ghidra_decompiler_internals/Raw_Pcode.md" %}}) and `Varnode` representation.

We will likely run many more datatests to verify our test framework.  If they pass, we can claim that a rebuilt
`decompile_datatest` executable functions like the native `decompile` executable, but a lot faster.  This execution
took 90 ms of real time to complete.

When we wrap this test inside `integrationTest.py` we will want to add lots of checks to make sure there were no memory
leaks and no decompiler error messages present.

Later on we will want to rerun this datatest with an experimental riscv_vector plugin, and expect to see the `print C`
command returning instead:

```c
void memcpy_i2(void *to,void *from)

{
  vector_memcpy(to, from, 2);
  return;
}
```