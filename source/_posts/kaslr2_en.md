---
title: "Linux ARM64 KASLR Implementation(2): Linear Mapping Randomization"
date: 2023-04-23 20:23:15
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

This post is the continuation of [Linux ARM64 KASLR Implementation(1): Kernel Image Randomization](../kaslr_en/).

## Implementation mechanism of linear address randomization

In the previous article, it was mentioned that in `__primary_switch`, the program called `__pi_kaslr_early_init` to obtain the random offset used by KASLR and stored the low 20 bits in register `x24`. Then in `__primary_switched`, the value of `x24` was stored in `memstart_offset_seed`.

```c
SYM_FUNC_START_LOCAL(__primary_switched)

        [...]

#ifdef CONFIG_RANDOMIZE_BASE
        adrp    x5, memstart_offset_seed        // Save KASLR linear map seed    
        strh    w24, [x5, :lo12:memstart_offset_seed]
#endif

        [...]

SYM_FUNC_END(__primary_switched)
```

`memstart_offset_seed` is defined in `arch/arm64/kernel/kaslr.c` :

```c
u16 __initdata memstart_offset_seed;
```

As you can see, it is a `u16`, so `strh` is used above to leave the remaining 16 bits of its random value.

### `arm64_memblock_init`

The initialization process will then proceed to the `arm64_memblock_init` function.

```c
start_kernel
 --> setup_arch
      --> arm64_memblock_init
      --> paging_init
```

Extracting the part related to KASLR:

```c
void __init arm64_memblock_init(void)
{

        [...]

        /*
         * Select a suitable value for the base of physical memory.
         */
        // Set memstart_addr to the address where DRAM starts (and then perform an alignment).
        memstart_addr = round_down(memblock_start_of_DRAM(),
                                   ARM64_MEMSTART_ALIGN);

        [...]

        // if KASLR is enabled
        if (IS_ENABLED(CONFIG_RANDOMIZE_BASE)) {
                // get the 16 bit random value
                extern u16 memstart_offset_seed;
                // Read the size of the physical address range supported by the CPU.
                u64 mmfr0 = read_cpuid(ID_AA64MMFR0_EL1);
                int parange = cpuid_feature_extract_unsigned_field(
                                        mmfr0, ID_AA64MMFR0_PARANGE_SHIFT);
                // linear_region_size is the size of the linear mapping range calculated by the kernel
                // subtract the physical address range size from linear_region_size to obtain range
                // so range represents how much space there is for movement of physical addresses within the linear mapping range
                s64 range = linear_region_size -
                            BIT(id_aa64mmfr0_parange_to_phys_shift(parange));
    
                /*
                 * If the size of the linear region exceeds, by a sufficient
                 * margin, the size of the region that the physical memory can
                 * span, randomize the linear region as well.
                 */
                // if the random value is not 0, and the movement space is greater than
                // ARM64_MEMSTART_ALIGN (there is enough space for randomization), then
                // the method of changing memstart_addr is used to randomize the linear space.
                if (memstart_offset_seed > 0 && range >= (s64)ARM64_MEMSTART_ALIGN) {
                        range /= ARM64_MEMSTART_ALIGN;
                // The value used to subtract in the following line is not very
                // easy to understand. It can be rearranged mathematically to imagine
                // it as (range * ARM64_MEMSTART_ALIGN) * (memstart_offset_seed >> 16),
                // which means (range original value, because range was divided by ARM64_MEMSTART_ALIGN earlier)
                // * (memstart_offset_seed / (1 << 16), which is a random value between 0 and 1)
                // Therefore, the result is a random value between 0 and the original range.
                        memstart_addr -= ARM64_MEMSTART_ALIGN *
                                         ((range * memstart_offset_seed) >> 16);
                }
        }
 
        [...]

}
```

Why does changing `memstart_addr` change the address of the linear space?

## Linear Address Randomization

Let's take a look at the page table operations that actually randomize the linear space. The page table for the linear mapping is created in `paging_init` → `map_mem` :

```c
static void __init map_mem(pgd_t *pgdp)
{

        [...]

        /* map all the memory banks */
        // this loop calls __map_memblock to create a linear mapping
        // for all physical memory regions in the system.
        for_each_mem_range(i, &start, &end) {
                if (start >= end)
                        break;
                /*
                 * The linear map must allow allocation tags reading/writing
                 * if MTE is present. Otherwise, it has the same attributes as
                 * PAGE_KERNEL.
                 */
                // start: start of the physical region
                // end: end of the physical region
                // pgdp is `swapper_pg_dir, the root of the page table that the
                // kernel is going to use later
                __map_memblock(pgdp, start, end, pgprot_tagged(PAGE_KERNEL),
                               flags);
        }

        [...]

}
```

### `__map_memblock`

```c
static void __init __map_memblock(pgd_t *pgdp, phys_addr_t start,
                                  phys_addr_t end, pgprot_t prot, int flags)
{
        __create_pgd_mapping(pgdp, start, __phys_to_virt(start), end - start,
                             prot, early_pgtable_alloc, flags);
}
```

`__create_pgd_mapping` is analyzed in detail in [Linux ARM64 \__create_pgd_mapping analysis](../page_table_en/). The key point here is that a page table mapping from the physical address range `[start - end)` to the virtual address range `[__phys_to_virt(start) - __phys_to_virt(end))` is created here. This means that `__phys_to_virt` determines the virtual address of the linear mapping.

### `__phys_to_virt`

This macro is defined in `arch/arm64/include/asm/memory.h` :

```c
#define __phys_to_virt(x)       ((unsigned long)((x) - PHYS_OFFSET) | PAGE_OFFSET)
```

In the same file, `PHYS_OFFSET` is defined as `memstart_addr` (pretty much)

```c
extern s64                      memstart_addr;
/* PHYS_OFFSET - the physical address of the start of memory. */
#define PHYS_OFFSET             ({ VM_BUG_ON(memstart_addr & 1); memstart_addr; })
```

So we are back to `memstart_addr`. In `arm64_memblock_init`, `memstart_addr` is subtracted by a random value, which leads to the randomization of `PHYS_OFFSET` and `__phys_to_virt`. Therefore, the virtual address of the linear mapping created by `paging_init` → `map_mem` → `___map_memblock` → `__create_pgd_mapping` is also randomized.
