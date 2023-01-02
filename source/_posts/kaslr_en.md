---
title: "Linux ARM64 KASLR Implementation(1): Kernel Image Randomization"
date: 2023-01-02 22:03:15
summary:
tags:
  - Linux
  - KASLR
  - ARMv8
categories: technical
---

> Linux version: v6.0
>
> Architecture: ARMv8

## Introduction

I have heard of KASLR (Kernel Address Space Layout Randomization) for a long time now, and had always find it interesting. However because KASLR isn’t really a core operating system concept, I never got the chance to research it other than reading some random articles online that only discusses the topic conceptually. The deep knowledge of how it actually works had always been missing.

I recently got some time to deeply investigate this cool feature of Linux, I will split what I found in two blogs, first explaining the implementation of the kernel image randomization, the second explaining the randomization of the linear mapping.

> It is advised to learn about the boot flow of the Linux ARM64 kernel before proceeding.

## KASLR Implementation

KASLR is closely related to Linux ARM64’s boot sequence, the execution context will be breifly presented, then the code explanation will start from the early function `__primary_switch` .

### Execution Context

The boot (primary) CPU runs to the `__primary_switch` with the MMU not yet enabled, with CPU system registers initialized, identity map page tables initialized. Also with `x22` storing the physical address of the FDT (Flat Device Tree).

### `__primary_switch`

`__primary_switch` means “primary CPU switch on MMU”

```c
SYM_FUNC_START_LOCAL(__primary_switch)
        adrp    x1, reserved_pg_dir
        adrp    x2, init_idmap_pg_dir
        // turn on the MMU, use the identity map in init_idmap_pg_dir
        bl      __enable_mmu
#ifdef CONFIG_RELOCATABLE
        // x23 = physical address of the kernel image
        adrp    x23, KERNEL_START
        // x23 = x23 % 2MB, most likely == 0 since the kernel is mandated to
        // be loaded with 2MB alignment
        and     x23, x23, MIN_KIMG_ALIGN - 1
#ifdef CONFIG_RANDOMIZE_BASE
        // x0 = FDT address, this is the argument for __pi_kaslr_early_init
        mov     x0, x22
        // use init_pg_end as the temporary stack
        adrp    x1, init_pg_end
        mov     sp, x1
        // not sure, probably only just for the sake of clearing x29
        mov     x29, xzr
        // see explanation below, x0 = KASLR random offset upon return
        bl      __pi_kaslr_early_init
        // x24 = x0 % 2MB
        and     x24, x0, #SZ_2M - 1             // capture memstart offset seed
        // x0 = x0 round down to 2MB
        bic     x0, x0, #SZ_2M - 1
        // x23 = x23 + x0 = 0 + random offset
        orr     x23, x23, x0                    // record kernel offset
#endif
#endif
        // not important to KASLR
        bl      clear_page_tables
        // this creates the high address mapping for the kernel
        // the important part is it uses KIMAGE_VADDR + x23 to map it,
        // therefore the mapping it creates is randomized using the random offset
        bl      create_kernel_mapping

        adrp    x1, init_pg_dir
        // use init_pg_dir for the high address
        load_ttbr1 x1, x1, x2
#ifdef CONFIG_RELOCATABLE
        // see explanation below
        bl      __relocate_kernel
#endif
        // see explanation below (end of __primary_switch)
        ldr     x8, =__primary_switched
        adrp    x0, KERNEL_START                // __pa(KERNEL_START)
        br      x8
SYM_FUNC_END(__primary_switch)
```

### `kaslr_early_init`

The Makefile which `arch/arm64/kernel/pi/kaslr_early.c` uses, which `kaslr_early_init` resides in, prepends all symbols with `__pi_` , therefore the `bl __pi_kaslr_early_init` call gets directed here.

```c
asmlinkage u64 kaslr_early_init(void *fdt)
{
        u64 seed;
        // return if KASLR is disabled in the command line
        if (is_kaslr_disabled_cmdline(fdt))
                return 0;
        // get KASLR seed from FDT
        seed = get_kaslr_seed(fdt);
        // if we don't get the seed from FDT
        if (!seed) {
                // return if the CPU does not have the ability to generate
                // a random seed or the operation failed
                if (!__early_cpu_has_rndr() ||
                    !__arm64_rndr((unsigned long *)&seed))
                        return 0;
        }

        /*
         * OK, so we are proceeding with KASLR enabled. Calculate a suitable
         * kernel image offset from the seed. Let's place the kernel in the
         * middle half of the VMALLOC area (VA_BITS_MIN - 2), and stay clear of
         * the lower and upper quarters to avoid colliding with other
         * allocations.
         */
        // use the random seed to generate a suitable KASLR offset and return
        return BIT(VA_BITS_MIN - 3) + (seed & GENMASK(VA_BITS_MIN - 3, 0));
}
```

### `__relocate_kernel`

In the code explanation below, I removed the part in the `CONFIG_RELR` option, because I decided to skip that part and also `defconfig` does not use it.

This is the critical function for KASLR, it relocates the kernel at runtime. The kernel build process produces a relocation entry for each global symbol access, by including the entries in the kernel image, it allows the kernel to read the entries and relocate itself at runtime.

> [This reference](https://github.com/ARM-software/abi-aa/blob/main/aaelf64/aaelf64.rst) contains information about ARM relocation entry types, only one type is used here.

The format of ELF relocation entry looks like:

```c
typedef struct {
        // compile time address of the place to be relocated
        Elf64_Addr      r_offset;
        // symbol table index and relocation type
        Elf64_Xword     r_info;
        // the addend used in the relocation, meaning depends on the type
        Elf64_Sxword    r_addend;
} Elf64_Rela;
```

KASLR only processes one type of relocation that is `R_AARCH64_RELATIVE` , it requires writing “`r_addend + random offset (address difference between run time and compile time)` “ to the address corresponding to `r_offset` at run time. For example assume there’s an instruction trying to load symbol `sym` ‘s address into register `x0` , the compiler emits `ldr x0, #a0` , `#a0` here is just for demoing purpose. Assume `sym` ‘s compile time address is `0x500` , the instruction’s compile time address is `0x1000` , the relocation generated would be: `r_offset = 0x10a0` , `r_addend = 0x500` . Further assume the instruction’s run time address is `0x3000` , (so `sym` ‘s run time address is `0x2500` ), the random offset is `0x2000` . What we need to do to relocate is read `r_addend (0x500)` , add it with `0x2000` (the offset), then save the result into `r_offset` \+ (the offset), that is writing `0x2500` into `0x10a0 + 0x2000 = 0x30a0` . Now when the instruciton is run `x0` would be assigned `0x2500` , which is the correct run time address of `sym` .

```c
SYM_FUNC_START_LOCAL(__relocate_kernel)
        /*
         * Iterate over each entry in the relocation table, and apply the
         * relocations in place.
         */
        // x9 = physical address of the first relocation entry
        adr_l   x9, __rela_start
        // x10 = address just past the last entry
        adr_l   x10, __rela_end
        // x11 = default kernel virtual address
        mov_q   x11, KIMAGE_VADDR               // default virtual offset
        // x11 += random offset, creates the random virtual address
        add     x11, x11, x23                   // actual virtual offset
        // return if x9 >= x10
0:      cmp     x9, x10
        b.hs    1f
        // x12 = r_offset, x13 = r_info and move x9 to the next entry
        ldp     x12, x13, [x9], #24
        // x14 = r_addend
        ldr     x14, [x9, #-8]
        // is r_info R_AARCH64_RELATIVE ?
        cmp     w13, #R_AARCH64_RELATIVE
        // process the next entry if not R_AARCH64_RELATIVE
        b.ne    0b
        // x14 = r_addend + random offset
        add     x14, x14, x23                   // relocate
        // write (r_addend + random offset) into (r_offset + random offset)
        // to relocate
        str     x14, [x12, x23]
        b       0b
1:
        ret
SYM_FUNC_END(__relocate_kernel)
```

### Ending of `__primary_switch`

A global address load happens **right** after `__relocation kernel` returns:

```c
// this load instruction generates a relocation entry which is relocated
// in __relocate_kernel, x8 gets the run time address of __primary_switched,
// which has had the random offset added at this point,
// create_kernel_mapping created the mapping with the random offset added,
// and the page table was installed at load_ttbr1 x1, x1, x2,
// everything is setup at this point, the br instruction jumps to high
// address for the kernel to continue to run
ldr     x8, =__primary_switched
adrp    x0, KERNEL_START                // __pa(KERNEL_START)
br      x8
```

## Linker Options

The ARM64 Makefile contains linker options for KASLR:

```c
ifeq ($(CONFIG_RELOCATABLE), y)
# Pass --no-apply-dynamic-relocs to restore pre-binutils-2.27 behaviour
# for relative relocs, since this leads to better Image compression
# with the relocation offsets always being zero.
LDFLAGS_vmlinux         += -shared -Bsymbolic -z notext \
                        $(call ld-option, --no-apply-dynamic-relocs)
endif
```

Four options are used:

* `—no-apply-dynamic-relcos` seems to be related to the skipped `CONFIG_RELR` , but I’m not sure

* `-shared` means to create a shared library

* `-Bsymbolic` means to bind references to global symbols to the definition within the shared library, if any. I think this prevents `ld` from emitting relocation entries that is used for external symbols.

* `-z notext` : not sure about this one

## Some Questions

The code analysis above is pretty thorough, but a lot of further questions are raised as well, hope I will be able to answer them as I learn more.

* What are the actual differences when we add the four linker options? What if we don’t add them?

* Originally I thought all of the relocation entry would disappear after turning off KASLR, but they still persisted, this doesn’t affect the kernel from running properly but doesn’t this waste some space?

* How does the build process limit the relocation entry to only emit the `R_AARCH64_RELATIVE` type? What circumstances generate different relocation types?
