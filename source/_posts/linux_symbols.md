---
title: Looking for Kernel Symbol Addresses in the Linux Kernel Image
date: 2022-01-27 21:20:15
summary:
tags:
  - Linux
  - kallsyms
categories: technical
---

I have always wondered how the linux kernel translates addresses to symbol names. This mechanism (called `kallsyms`) is used in multiple places in the kernel, for example in the `panic()` call the kernel prints out the call trace with function names and offsets with no problem at all, also the `ftrace` system is able to show the user all function names that the kernel has run, similarly, using `kallsyms`.

[This article](https://blog.csdn.net/jasonchen_gbd/article/details/44025681?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&utm_relevant_index=2) written in Chinese explains the translation mechanism pretty well, and is worth a read if one is interested in knowing how it is done. However when discussing this topic with a professor, he asked me how I can find the raw bytes of the symbols used by `kallsyms` which are explained in the linked article above e.g. `kallsyms_addresses` and `kallsyms_num_syms`, etc. So, I set to find out where they are.

> It is unlikely for me and you to use the exact same kernel image, so the offsets and symbols won't be the same. Moreover, I'm using an ARMv8 image of linux version 5.4.

From the article linked above we know that the `kallsyms` symbols are in the `.rodata` section, lets see where it is:

```
$ aarch64-linux-gnu-readelf -l vmlinux
Elf file type is EXEC (Executable file)
Entry point 0xffff800010080000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000010000 0xffff800010080000 0xffff800010080000
                 0x00000000008f2390 0x00000000008f2390  R E    0x10000
  LOAD           0x0000000000910000 0xffff800010980000 0xffff800010980000
                 0x00000000002de264 0x00000000002de264  RW     0x10000
  LOAD           0x0000000000bf0000 0xffff800010c70000 0xffff800010c70000
                 0x00000000001bf200 0x00000000001f51b8  RWE    0x10000
  NOTE           0x0000000000bee228 0xffff800010c5e228 0xffff800010c5e228
                 0x000000000000003c 0x000000000000003c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10

 Section to Segment mapping:
  Segment Sections...
   00     .head.text .text 
   01     .rodata .modinfo .init.eh_frame .pci_fixup __ksymtab __ksymtab_gpl __ksymtab_strings __param __modver __ex_table .notes 
   02     .init.text .exit.text .altinstructions .altinstr_replacement .init.data .data..percpu .data __bug_table .mmuoff.data.write .mmuoff.data.read .pecoff_edata_padding .bss 
   03     .notes 
   04     
```

Nice, `.rodata` is located at the very beginning of the second segment, very convenient. The space between the first segment and the second is obtained by subtracting the offsets of the two segments, `0x910000 - 0x10000 = 0x900000`. The first segment is placed at the very beginning of the kernel image, therefore starting from file offset `0x900000` of the kernel image should be the place where `.rodata` belongs. Now we just have to see where all those symbols are relative to the start of the `.rodata` section.

```
$ aarch64-linux-gnu-readelf -s vmlinux | grep kallsyms
...
 80586: ffff800010a73318     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_relative_base
 81462: ffff800010b5fd68     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_token_table
 81931: ffff800010b600d8     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_token_index
 83269: ffff800010a73320     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_num_syms
 85611: ffff800010a73328     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_names
 88091: ffff80001018aee0   204 FUNC    GLOBAL DEFAULT    2 kallsyms_lookup_size_offs
 88746: ffff80001018ad60   208 FUNC    GLOBAL DEFAULT    2 kallsyms_on_each_symbol
 91655: ffff80001018b7b0    96 FUNC    GLOBAL DEFAULT    2 kallsyms_show_value
 91983: ffff80001018ae30   172 FUNC    GLOBAL DEFAULT    2 kallsyms_lookup_name
 93340: ffff80001018afb0   348 FUNC    GLOBAL DEFAULT    2 kallsyms_lookup
 93641: ffff800010d6a07c     4 OBJECT  GLOBAL DEFAULT   20 bpf_jit_kallsyms
 93956: ffff80001018a640   260 FUNC    GLOBAL DEFAULT    2 module_kallsyms_lookup_na
 94125: ffff8000101e5290   204 FUNC    GLOBAL DEFAULT    2 bpf_prog_kallsyms_del
 94245: ffff800010b5f8c8     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_markers
 94636: ffff800010a29558     0 NOTYPE  GLOBAL DEFAULT    3 kallsyms_offsets
...
```

Note that one of, if not the most important symbol `kallsyms_addresses` is not present. This threw me off quite a bit, but eventually I found that in the file which is responsible for generating the symbol information `/scripts/kallsyms.c` it is explained that there are two ways of storing the addresses of kernel symbols, the first way uses `kallsyms_addresses` and it simply stores all the symbol addresses in the `kallsyms_addresses` "array". The second way is to save offsets relative to a base address, which is the address of the symbol with the lowest address. The two ways in summary:
1. `kallsyms_addresses` stores all the address
2. `kallsyms_relative_base` stores the base address and `kallsyms_offsets` store all the offsets

Let's first look for `kallsyms_relative_base`, its address is `0xffff800010a73318`, and the address of `.rodata` is `0xffff800010980000`, this gives us the offset `0xf3318`, adding back the offset of `.rodata` in the image (`0x900000`) we get the final file offset of the symbol `0x9f3318`. 

Check this in hexdump:
```
$ hexdump Image
...
09f3300 51b0 00de 51b8 00de 6000 00de 9000 00de
09f3310 9000 00de 0000 0000 0000 1008 8000 ffff
09f3320 276f 0001 0000 0000 fe04 6568 05be 5f54
...
```
Confirm that `0xffff800010080000` is the first symbol address.
We can calculate the file offset of the symbol offsets `kallsyms_offsets` the same way, the result is `0x9a9558`
```
09a9550 656c 0000 0000 0000 0000 0000 0000 0000
09a9560 0040 0000 0044 0000 0058 0000 0070 0000
09a9570 00f8 0000 1000 0000 1000 0000 1000 0000
09a9580 1000 0000 1070 0000 11a0 0000 1230 0000
09a9590 12e0 0000 1378 0000 1458 0000 1558 0000
```
It is an array with increasing elements of four bytes each, starting from `0x00000000`, my guess for the reason why there are two symbols of offset `0x00000000` is that there are two symbols both refering to the very start of the kernel image.

Lastly let's see how many kernel symbols are there (`kallsyms_num_syms`), `0xffff800010a73320 - 0xffff800010980000 + 0x900000 = 0x9f3320`. It's just the next 8 bytes of `kallsyms_relative_base`.
```
09f3320 276f 0001 0000 0000 fe04 6568 05be 5f54
```
so the value is `0x1276f`, 75631 in decimal.
