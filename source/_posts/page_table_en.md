---
title: Linux ARM64 `__create_pgd_mapping` Analysis
date: 2023-02-13 18:21:15
summary:
tags:
  - Linux
  - page tables
categories: technical
---

> Linux version: v6.0
>
> Architecture: ARMv8

## Introduction

Memory mapping is probably the toughest part to understand in the boot flow of the Linux kernel, this is partly because the code setting up the memory mappings is also running under a certain mapping. Therefore other than understanding what the code is doing, knowing what the current mapping is is also important. The code may also alter the memory mapping in several ways as well. The variables complicates matters even more, contains sometimes physical addresses, sometimes virtual addresses.

`__create_pgd_mapping` is introduced in this article, it serves as a helper function to create page table mappings between virtual and physical address ranges. Instead of how the function is used, its implementation is discussed here.

## `__create_pgd_mapping` Execution Context

Let’s talk a little about the execution context of `__create_pgd_mapping` before diving into the implementation. This function is mostly used early during boot, MMU has been enabled, the pgd is (mostly) `init_pg_dir` , linear mapping may not be created yet so it is not used in the function, the fixmap is created other than the actual level 3 entries, the kernel is running at high address.

## `__create_pgd_mapping` Function Signature

`__create_pgd_mapping` ‘s function signature is as follows:

```c
static void __create_pgd_mapping(pgd_t *pgdir, phys_addr_t phys,
                                 unsigned long virt, phys_addr_t size,
                                 pgprot_t prot,
                                 phys_addr_t (*pgtable_alloc)(int),
                                 int flags);
```

* `pgdir` : virtual address of the root of the page table to create the mapping

* `phys` : physical address of the start of the range to map

* `virt` : virtual address of the start of the range to map

* `size` : size of the range to map

* `prot` : attributes of the mapping

* `pgtable_alloc` : the memory allocation function to be used during the creation of the mapping

* `flags` : some flags to dictate the behavior when creating the mapping

## `__create_pgd_mapping` Implementation

```c
static void __create_pgd_mapping(pgd_t *pgdir, phys_addr_t phys,
                                 unsigned long virt, phys_addr_t size,
                                 pgprot_t prot,
                                 phys_addr_t (*pgtable_alloc)(int),
                                 int flags)
{
        mutex_lock(&fixmap_lock);
        __create_pgd_mapping_locked(pgdir, phys, virt, size, prot,
                                    pgtable_alloc, flags);
        mutex_unlock(&fixmap_lock);
}
```

This is trivial, take the fixmap lock and call `__create_pgd_mapping_locked`

### `__create_pgd_mapping_locked`

```c
static void __create_pgd_mapping_locked(pgd_t *pgdir, phys_addr_t phys,
                                        unsigned long virt, phys_addr_t size,
                                        pgprot_t prot,
                                        phys_addr_t (*pgtable_alloc)(int),
                                        int flags)
{
        unsigned long addr, end, next;
        // calculate the address of the pgd entry to translate virt
        pgd_t *pgdp = pgd_offset_pgd(pgdir, virt);

        /*
         * If the virtual and physical address don't have the same offset
         * within a page, we cannot map the region as the caller expects.
         */
        if (WARN_ON((phys ^ virt) & ~PAGE_MASK))
                return;
        // round down phys to size of a page
        phys &= PAGE_MASK;
        // addr is a local variable recording the virtual address to be mapped now,
        // it is initialized to virt rounded down to size of a page
        addr = virt & PAGE_MASK;
        // end is the end of the virtual address range to be mapped
        end = PAGE_ALIGN(virt + size);
        // this loop creates mapping for a pgd entry in one iteration,
        // e.g. a 512GB range for the 48bits VA, 4K case
        do {
                // pgd_addr_end returns (addr + range for one pgd entry), or end, 
                // depending on which is smaller
                next = pgd_addr_end(addr, end);
                // send current pgdp,
                // addr(start of the VA of this iteration)
                // next(end of the VA of this iteration)
                // phys(start of the PA of this iteration)
                // prot, pgtable_alloc, flag to the function handling the pud level
                alloc_init_pud(pgdp, addr, next, phys, prot, pgtable_alloc,
                               flags);
                // alloc_init_pud creates mapping for addr~next, so step forward
                // phys an amount of next - addr after return
                phys += next - addr;
        // process the next pgd entry, also step forward addr, end if addr == end
        } while (pgdp++, addr = next, addr != end);
}
```

### `alloc_init_pud`

```c
static void alloc_init_pud(pgd_t *pgdp, unsigned long addr, unsigned long end,
                           phys_addr_t phys, pgprot_t prot,
                           phys_addr_t (*pgtable_alloc)(int),
                           int flags)
{
        unsigned long next;
        pud_t *pudp;
        // p4d is the extra level used in five level paging, if fewer than five levels
        // p4dp = pgdp, assume four level paging for now
        p4d_t *p4dp = p4d_offset(pgdp, addr);
        // assume p4d = pgd here
        p4d_t p4d = READ_ONCE(*p4dp);
        // if the entry pointed to by pgdp is none
        if (p4d_none(p4d)) {
                // prepare an entry pointing to a table (pud), and not executable in
                // EL0
                p4dval_t p4dval = P4D_TYPE_TABLE | P4D_TABLE_UXN;
                phys_addr_t pud_phys;
                // if the flags contains NO_EXEC_MAPPING
                if (flags & NO_EXEC_MAPPINGS)
                        // then not executable in EL1 as well
                        p4dval |= P4D_TABLE_PXN;
                BUG_ON(!pgtable_alloc);
                // allocate a pud page using the function passed in
                pud_phys = pgtable_alloc(PUD_SHIFT);
                // populate the pgd entry with the physical address of the pud page
                // and the attributes in p4dval
                __p4d_populate(p4dp, pud_phys, p4dval);
                // read the entry out to check later
                p4d = READ_ONCE(*p4dp);
        }
        // here the p4d has to be a valid pgd entry
        BUG_ON(p4d_bad(p4d));
        // this line of code is tricky, it maps the pud page to a fixed virtual address
        // this is necessary because we read a pgd entry from pgdp, which contains the
        // pud page's physical address, so we must map the pud page to a virtual
        // address before we can actually access the pud page, here the fixmap mechanism
        // is used to map the pud page
        // pudp = the virtual address of the pud entry used to translate addr
        pudp = pud_set_fixmap_offset(p4dp, addr);
        do {
                // read the pud entry
                pud_t old_pud = READ_ONCE(*pudp);
                // next = ((addr + range of one pud entry), or end, whichever is smaller)
                next = pud_addr_end(addr, end);

                /*
                 * For 4K granule only, attempt to put down a 1GB block
                 */
                // if it is possible doing a section mapping (1GB)
                if (pud_sect_supported() &&
                   // the current range to map is precisely 1GB and is aligned
                   ((addr | next | phys) & ~PUD_MASK) == 0 &&
                    // also the flag does not forbid a section(block) mapping
                    (flags & NO_BLOCK_MAPPINGS) == 0) {
                        // map the 1GB page here
                        pud_set_huge(pudp, phys, prot);

                        /*
                         * After the PUD entry has been populated once, we
                         * only allow updates to the permission attributes.
                         */
                        BUG_ON(!pgattr_change_is_safe(pud_val(old_pud),
                                                      READ_ONCE(pud_val(*pudp))));
                } else {
                        // pudp: the virtual address of the current pud entry
                        // addr: the starting virtual address of the range to map
                        // next: the ending virtual address of the range to map
                        // phys: the physical address of the range to map
                        // alloc_init_cont_pmd is responsible for mapping addr~next
                        alloc_init_cont_pmd(pudp, addr, next, phys, prot,
                                            pgtable_alloc, flags);
                        BUG_ON(pud_val(old_pud) != 0 &&
                               pud_val(old_pud) != READ_ONCE(pud_val(*pudp)));
                }
                // here the mapping of addr~next is completed, so step forward phys
                // by next - addr
                phys += next - addr;
        // go on to the next pud entry, and step forward addr, finish if addr == end
        } while (pudp++, addr = next, addr != end);
        // clear the pud page map created earlier
        pud_clear_fixmap();
}
```

Here requires some explanation, the call chain looks like this:

```bash
__create_pgd_mapping
--> __create_pgd_mapping_locked
    --> alloc_init_pud
        // the following functions are explained below
        --> alloc_init_cont_pmd
            --> init_pmd
                --> alloc_init_cont_pte
                    --> init_pte 
```

The analysis up to now explained that:

`__create_pgd_mapping_locked` is responsible for filling in the pgd page

`alloc_init_pud` is responsible for filling in the pud page

Then why do pmd and pte each requires two functions to handle them? The answer is that in ARMv8 the page table entries contain a contiguous bit, simply put, if serveral continuous page table entries (continuous virtual address range) is mapped to a contiguous physical address range, software can set this bit in the page table entries to achieve better TLB performance. `alloc_init_cont_pmd` and `alloc_init_cont_pte` tries to use the contiguous bit base on the address ranges passed in, and `init_pmd` and `init_pte` is responsible for the actual filling of the pmd and pte pages.

### `alloc_init_cont_pmd`

```c
static void alloc_init_cont_pmd(pud_t *pudp, unsigned long addr,
                                unsigned long end, phys_addr_t phys,
                                pgprot_t prot,
                                phys_addr_t (*pgtable_alloc)(int), int flags)
{
        unsigned long next;
        // read the pud entry
        pud_t pud = READ_ONCE(*pudp);

        /*
         * Check for initial section mappings in the pgd/pud.
         */
        // we shouldn't be called it the pud is a section mapping, so BUG
        BUG_ON(pud_sect(pud));
        // if this entry is none
        if (pud_none(pud)) {
                // prepare an entry pointing to a table (pmd), and not executable in
                // EL0
                pudval_t pudval = PUD_TYPE_TABLE | PUD_TABLE_UXN;
                phys_addr_t pmd_phys;
                // if the flags contains NO_EXEC_MAPPING
                if (flags & NO_EXEC_MAPPINGS)
                        // then not executable in EL1 as well
                        pudval |= PUD_TABLE_PXN;
                BUG_ON(!pgtable_alloc);
                // allocate a pud page using the function passed in
                pmd_phys = pgtable_alloc(PMD_SHIFT);
                // populate the pud entry with the physical address of the pud page
                // and the attributes in pudval
                __pud_populate(pudp, pmd_phys, pudval);
                // read the entry out to check later
                pud = READ_ONCE(*pudp);
        }
        // here the pud has to be a valid pud entry
        BUG_ON(pud_bad(pud));

        do {
                pgprot_t __prot = prot;
                // next = ((addr + range of one pmd entry), or end, whichever is smaller)
                next = pmd_cont_addr_end(addr, end);

                /* use a contiguous mapping if the range is suitably aligned */
                // check if the range addr~next is large enough and aligned for 
                // contiguous mapping
                if ((((addr | next | phys) & ~CONT_PMD_MASK) == 0) &&
                    // also the flag does not forbid a contiguous mapping
                    (flags & NO_CONT_MAPPINGS) == 0)
                        // or the attributes with the contiguous bit
                        __prot = __pgprot(pgprot_val(prot) | PTE_CONT);
                // call init_pmd to process the pmd page
                init_pmd(pudp, addr, next, phys, __prot, pgtable_alloc, flags);
                // here the mapping of addr~next is completed, so step forward phys
                // by next - addr
                phys += next - addr;
        // step forward addr, break if addr == end
        } while (addr = next, addr != end);
}
```

### `init_pmd`

```c
static void init_pmd(pud_t *pudp, unsigned long addr, unsigned long end,
                     phys_addr_t phys, pgprot_t prot,
                     phys_addr_t (*pgtable_alloc)(int), int flags)
{
        unsigned long next;
        pmd_t *pmdp;
        // just like pud_set_fixmap_offset previously, read the physical address
        // of the pmd page from pudp, and map it to a fixed virtual address, then
        // return the virtual address in the pmd page that points to the entry
        // used when translating addr
        pmdp = pmd_set_fixmap_offset(pudp, addr);
        do {
                // read the pmd entry
                pmd_t old_pmd = READ_ONCE(*pmdp);
                // next = ((addr + range of one pmd entry), or end, whichever is smaller)
                next = pmd_addr_end(addr, end);

                /* try section mapping first */
                // if the size of addr~next is a multiple of a pmd section (usually 2MB)
                // and is properly aligned
                if (((addr | next | phys) & ~PMD_MASK) == 0 &&
                    // also the flags does not forbid section mapping
                    (flags & NO_BLOCK_MAPPINGS) == 0) {
                        // create the huge page mapping
                        pmd_set_huge(pmdp, phys, prot);

                        /*
                         * After the PMD entry has been populated once, we
                         * only allow updates to the permission attributes.
                         */
                        BUG_ON(!pgattr_change_is_safe(pmd_val(old_pmd),
                                                      READ_ONCE(pmd_val(*pmdp))));
                } else {
                        // for contiguous pte
                        alloc_init_cont_pte(pmdp, addr, next, phys, prot,
                                            pgtable_alloc, flags);

                        BUG_ON(pmd_val(old_pmd) != 0 &&
                               pmd_val(old_pmd) != READ_ONCE(pmd_val(*pmdp)));
                }
                // alloc_init_pud creates mapping for addr~next, so step forward
                // phys an amount of next - addr after return
                phys += next - addr;
        // go on to the next pmd entry, and step forward addr, finish if addr == end
        } while (pmdp++, addr = next, addr != end);
        // clear the pmd page map created earlier
        pmd_clear_fixmap();
}
```

The following `alloc_init_cont_pte` and `init_pte` is pretty much the same as the pmd version, try to trace it yourself!

### `alloc_init_cont_pte`

```c
static void alloc_init_cont_pte(pmd_t *pmdp, unsigned long addr,
                                unsigned long end, phys_addr_t phys,
                                pgprot_t prot,
                                phys_addr_t (*pgtable_alloc)(int),
                                int flags)
{
        unsigned long next;
        pmd_t pmd = READ_ONCE(*pmdp);

        BUG_ON(pmd_sect(pmd));
        if (pmd_none(pmd)) {
                pmdval_t pmdval = PMD_TYPE_TABLE | PMD_TABLE_UXN;                                                                                                       
                phys_addr_t pte_phys;

                if (flags & NO_EXEC_MAPPINGS)
                        pmdval |= PMD_TABLE_PXN;
                BUG_ON(!pgtable_alloc);
                pte_phys = pgtable_alloc(PAGE_SHIFT);
                __pmd_populate(pmdp, pte_phys, pmdval);
                pmd = READ_ONCE(*pmdp);
        }
        BUG_ON(pmd_bad(pmd));

        do {
                pgprot_t __prot = prot;

                next = pte_cont_addr_end(addr, end);

                /* use a contiguous mapping if the range is suitably aligned */
                if ((((addr | next | phys) & ~CONT_PTE_MASK) == 0) &&
                    (flags & NO_CONT_MAPPINGS) == 0)
                        __prot = __pgprot(pgprot_val(prot) | PTE_CONT);

                init_pte(pmdp, addr, next, phys, __prot);

                phys += next - addr;
        } while (addr = next, addr != end);
}
```

### `init_pte`

```c
static void init_pte(pmd_t *pmdp, unsigned long addr, unsigned long end,
                     phys_addr_t phys, pgprot_t prot)
{
        pte_t *ptep;

        ptep = pte_set_fixmap_offset(pmdp, addr);
        do {
                pte_t old_pte = READ_ONCE(*ptep);
                // this line actually does the work of writing the pte entry,
                // which is consisted of phys and prot, to the location pointed
                // to by ptep
                set_pte(ptep, pfn_pte(__phys_to_pfn(phys), prot));

                /*
                 * After the PTE entry has been populated once, we
                 * only allow updates to the permission attributes.
                 */
                BUG_ON(!pgattr_change_is_safe(pte_val(old_pte),
                                              READ_ONCE(pte_val(*ptep))));

                phys += PAGE_SIZE;
        } while (ptep++, addr += PAGE_SIZE, addr != end);

        pte_clear_fixmap();
}
```
