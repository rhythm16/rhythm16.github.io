---
title: "KVM ARM: The New Page Table Walker"
date: 2022-11-08 07:20:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
  - page tables
categories: technical
---

> Linux version: v6.0
>
> Architecture: ARMv8

## Foreword

During the 5.10 release cycle, KVM ARM had many code improvements in preparation for the [google pkvm](https://www.youtube.com/watch?v\=wY-u6n75iXc) project. We are going to talk about one part of it here: the new page table walker.

Previously the logic for walking page tables were implemented wherever it was needed, e.g. functions `create_hyp_{p4d, pud, pmd, pte}_mappings`. This approach causes the code for walking page tables to be duplicated. To improve this, a new modularized page table walker was introduced in 5.10. Other page table accesses benefit from using this same facility.

## Important Structures

Some information need to be provided when using the new page table walker:

1. The page table that is going to be accessed (`struct kvm_pgtable`)
2. Operations to be done to the page table, and when to do them (`struct kvm_pgtable_walker`)
3. The virtual address range to be accessed (`struct kvm_pgtable_walk_data`)

### `kvm_pgtable`

This struct stores the metadata of a page table. The comments are self-explanatory.

```c
/**
 * struct kvm_pgtable - KVM page-table.
 * @ia_bits:		Maximum input address size, in bits.
 * @start_level:	Level at which the page-table walk starts.
 * @pgd:		Pointer to the first top-level entry of the page-table.
 * @mm_ops:		Memory management callbacks.
 * @mmu:		Stage-2 KVM MMU struct. Unused for stage-1 page-tables.
 * @flags:		Stage-2 page-table flags.
 * @force_pte_cb:	Function that returns true if page level mappings must
 *			be used instead of block mappings.
 */
struct kvm_pgtable {  
        u32                                     ia_bits;
        u32                                     start_level;
        kvm_pte_t                               *pgd;
        struct kvm_pgtable_mm_ops               *mm_ops;
                                  
        /* Stage-2 only */  
        struct kvm_s2_mmu                       *mmu;
        enum kvm_pgtable_stage2_flags           flags;
        kvm_pgtable_force_pte_cb_t              force_pte_cb;
};  
```

### `kvm_pgtable_walker`

The user sets up this struct with the callback used during the walk, also when to invoke it.

```c
/**
 * struct kvm_pgtable_walker - Hook into a page-table walk.
 * @cb:		Callback function to invoke during the walk.
 * @arg:	Argument passed to the callback function.
 * @flags:	Bitwise-OR of flags to identify the entry types on which to
 *		invoke the callback function.
 */
struct kvm_pgtable_walker {
        const kvm_pgtable_visitor_fn_t          cb;       
        void * const                            arg; 
        const enum kvm_pgtable_walk_flags       flags;
// there are three non-exclusive options for flags:
// 1. KVM_PGTABLE_WALK_LEAF: call callback when visiting a leaf node
// 2. KVM_PGTABLE_WALK_TABLE_PRE: call callback before visiting children
// 3. KVM_PGTABLE_WALK_TABLE_POST: call callback before visiting children
};
```

### `kvm_pgtable_walk_data`

What range to walk, it also points to instances of the two previous structs.

```c
struct kvm_pgtable_walk_data {
        struct kvm_pgtable              *pgt; // points to the page table metadata
        struct kvm_pgtable_walker       *walker; // points to the walker

        u64                             addr; // starting address for the walk
        u64                             end; // ending address for the walk
};
```

## Implementation of the New Walker

Call `kvm_pgtable_walk` to use the new page table walker:

### `kvm_pgtable_walk`

```c
int kvm_pgtable_walk(struct kvm_pgtable *pgt, u64 addr, u64 size,
                     struct kvm_pgtable_walker *walker)
{
        struct kvm_pgtable_walk_data walk_data = {
                .pgt    = pgt,
                .addr   = ALIGN_DOWN(addr, PAGE_SIZE),
                .end    = PAGE_ALIGN(walk_data.addr + size),
                .walker = walker,
        };

        return _kvm_pgtable_walk(&walk_data);
}
```

This function expects the caller to provide `pgt` and `walker` initialized, then it creates a `kvm_pgtable_walk_data` together with `addr` and `size`. Then calls `_kvm_pgtable_walk`.

### `_kvm_pgtable_walk`

This function is responsible for the root pages of the page table, see the comments added for more info.

```c
static int _kvm_pgtable_walk(struct kvm_pgtable_walk_data *data)
{
        u32 idx;
        int ret = 0;
        struct kvm_pgtable *pgt = data->pgt;
        u64 limit = BIT(pgt->ia_bits);

        if (data->addr > limit || data->end > limit) // range check
                return -ERANGE;

        if (!pgt->pgd) // does root page table exist?
                return -EINVAL;
        // there may be multiple concatenated stage 2 page table root pages
        // (less than 16 pages) under certain system register settings
        // in ARMv8, this loop iterates over the multiple root pages
        // if the range to walk is large enough to requires that
        for (idx = kvm_pgd_page_idx(data); data->addr < data->end; ++idx) {
                // ptep points to the root page of the current iteration
                kvm_pte_t *ptep = &pgt->pgd[idx * PTRS_PER_PTE];
                // call the lower level implementation
                ret = __kvm_pgtable_walk(data, ptep, pgt->start_level);
                if (ret)
                        break;
        }
 
        return ret;
}
```

### `__kvm_pgtable_walk`

This function iterates over the entries in a page.

```c
static int __kvm_pgtable_walk(struct kvm_pgtable_walk_data *data,
                >             kvm_pte_t *pgtable, u32 level)
{
        u32 idx;
        int ret = 0;
        // return if this page table level is larger than max level (4)
        if (WARN_ON_ONCE(level >= KVM_PGTABLE_MAX_LEVELS))
                return -EINVAL;
        // use kvm_pgtable_idx to calculate the index to start with
        for (idx = kvm_pgtable_idx(data, level); idx < PTRS_PER_PTE; ++idx) {
                // calculate the current entry's address
                kvm_pte_t *ptep = &pgtable[idx];
                // break if completed walking the range
                if (data->addr >= data->end)
                        break;
                // input data, the current entry's address, and current level
                // to the function that does the visiting
                ret = __kvm_pgtable_visit(data, ptep, level);
                if (ret)
                        break;
        }

        return ret;
}
```

### `__kvm_pgtable_visit`

```c
static inline int __kvm_pgtable_visit(struct kvm_pgtable_walk_data *data,
                                      kvm_pte_t *ptep, u32 level)
{
        int ret = 0;
        u64 addr = data->addr;
        // pte: the value of the current entry
        kvm_pte_t *childp, pte = *ptep;
        // does this entry point to a table?
        bool table = kvm_pte_table(pte, level);
        enum kvm_pgtable_walk_flags flags = data->walker->flags;
        // if the entry points to a table and the caller wishes to visit
        // the entry before its children
        if (table && (flags & KVM_PGTABLE_WALK_TABLE_PRE)) {
                // kvm_pgtable_visitor_cb is just a simple wrapper for
                // passing arguments and calling walker->cb
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_TABLE_PRE);
        }
        // if entry points to a physical page (not the next level table)
        // and the caller wishes to visit the entry before its children
        if (!table && (flags & KVM_PGTABLE_WALK_LEAF)) {
                // call walker->cb
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_LEAF);
                // the memory content where ptep points to may change after
                // calling cb, for example cb might allocate the next level
                // table and inserted its address in the entry, so pte is
                // updated here
                pte = *ptep;
                // also update whether this pte points to a table or not
                table = kvm_pte_table(pte, level);
        }

        if (ret)
                goto out;
        // it the entry does not point to a table (it is a leaf node of the
        // page table)
        if (!table) {
                // align data->addr to the size of this page table level 
                // translation
                data->addr = ALIGN_DOWN(data->addr, kvm_granule_size(level));
                // step over this level's translation size
                data->addr += kvm_granule_size(level);
                // return after processing this leaf node
                goto out;
        }
        // the entry points to a table if this gets executed, use kvm_pte_follow
        // to get the next level page table's address. Note that entry holds the
        // physcal address of the next level table, so kvm_pte_follow uses
        // phys_to_virt to transform the address held in entry to EL1 kernel VA
        // used by software here.
        childp = kvm_pte_follow(pte, data->pgt->mm_ops);
        // call __kvm_pgtable_walk to walk the next level page table
        // see explanation after this code block 
        ret = __kvm_pgtable_walk(data, childp, level + 1);
        if (ret)
                goto out;
        // if the caller wishes to call cb after visiting current entry and all
        // of its children
        if (flags & KVM_PGTABLE_WALK_TABLE_POST) {
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_TABLE_POST);
        }

out:
        return ret;
}
```

Page tables are a n-ary tree data structures (512-ary for 4K pages), the page table walker uses recursion to access the tree with options for pre- and post-order traversal. `level` stands for which level and `__kvm_pgtable_visit` is responsible for:

1. calling the callback (`cb`) at the specified moments

2. steping `data→addr` to indicate progress

3. calling `__kvm_pgtable_walk` when the entry points to the next level table

And `__kvm_pgtable_walk` iterates over the entries in a single page.

## Usage: `create_hyp_mappings`

Linux runs in EL1 when initializing KVM, it uses `create_hyp_mappings` to setup the EL2 page tables before entering EL2.

`create_hyp_mappings` is called in `init_hyp_mode`, which is a crucial function in KVM initialization. It creates mappings for these areas for EL2.

* EL2 code (`__hyp_text_start` \~ `__hyp_text_end`)
* EL2 read only data (`__hyp_rodata_start` \~ `__hyp_rodata_end`)
* EL1 read only data (`__start_rodata` \~ `__end_rodata`)
* EL2 BSS (`__hyp_bss_start` \~ `__hyp_bss_end`)
* EL1 BSS (`__hyp_bss_end` \~ `__bss_stop`)
* EL2 stack
* EL2 percpu area

> This function only prepares the page tables, does not enter EL2 and start the address translation mechanism.

```c
/**
 * create_hyp_mappings - duplicate a kernel virtual address range in Hyp mode
 * @from:       The virtual kernel start address of the range
 * @to:         The virtual kernel end address of the range (exclusive)
 * @prot:       The protection to be applied to this range
 *
 * The same virtual address as the kernel virtual address is also used
 * in Hyp-mode mapping (modulo HYP_PAGE_OFFSET) to the same underlying
 * physical pages.
 */
int create_hyp_mappings(void *from, void *to, enum kvm_pgtable_prot prot)
{
        phys_addr_t phys_addr;
        unsigned long virt_addr;
        // use kern_hyp_va to transform the input EL1 VA into EL2 VA
        unsigned long start = kern_hyp_va((unsigned long)from);
        unsigned long end = kern_hyp_va((unsigned long)to);
        // CPU is EL2 mode? (VHE)
        if (is_kernel_in_hyp_mode())
                return 0;
        // related to pkvm, skip
        if (!kvm_host_owns_hyp_mappings())
                return -EPERM;
        // align the addresses with the page size
        start = start & PAGE_MASK;
        end = PAGE_ALIGN(end);
        // the loop runs through the input address range (steps one page every iteration),
        // uses kvm_kaddr_to_phys to calculate the corresponding physical address
        // and call __create_hyp_mappings
        for (virt_addr = start; virt_addr < end; virt_addr += PAGE_SIZE) {
                int err;

                phys_addr = kvm_kaddr_to_phys(from + virt_addr - start);
                err = __create_hyp_mappings(virt_addr, PAGE_SIZE, phys_addr,
                                            prot);
                if (err)
                        return err;
        }

        return 0;
}
```

The comments above the function are very helpful. `from` and `to` are the EL1 virtual address region to map to EL2, and `prot` specifies the protection. The interesting part is that EL2 virtual address range is not needed, KVM itself has a mechanism for translating EL1 VAs to EL2 VAs, specifically it uses `kern_hyp_va` for the translation.

### `__create_hyp_mappings`

takes the lock `kvm_hyp_pgd_mutex` then call `kvm_pgtable_hyp_map`. Note that `hyp_pgtable` is the `kvm_pgtable` needed for the walk.

```c
int __create_hyp_mappings(unsigned long start, unsigned long size,
                          unsigned long phys, enum kvm_pgtable_prot prot)
{
        int err;

        if (WARN_ON(!kvm_host_owns_hyp_mappings()))
                return -EINVAL;
 
        mutex_lock(&kvm_hyp_pgd_mutex);
        err = kvm_pgtable_hyp_map(hyp_pgtable, start, size, phys, prot);
        mutex_unlock(&kvm_hyp_pgd_mutex);
 
        return err;
}
```

### `kvm_pgtable_hyp_map`

Now this function calls the new page table walker `kvm_pgtable_walk`. It creates and passes the argument `kvm_pgtable_walker`(1), alongside `hyp_pgtable`(named `pgt` here) to `kvm_pgtable_walk`(2).

```c
int kvm_pgtable_hyp_map(struct kvm_pgtable *pgt, u64 addr, u64 size, u64 phys,
                        enum kvm_pgtable_prot prot)
{
        int ret;
        struct hyp_map_data map_data = {
                .phys   = ALIGN_DOWN(phys, PAGE_SIZE),
                .mm_ops = pgt->mm_ops,
        };
        struct kvm_pgtable_walker walker = { // (1)
                .cb     = hyp_map_walker,       
                .flags  = KVM_PGTABLE_WALK_LEAF,
                .arg    = &map_data,
        };

        ret = hyp_set_prot_attr(prot, &map_data.attr);
        if (ret)
                return ret;

        ret = kvm_pgtable_walk(pgt, addr, size, &walker); // (2)
        dsb(ishst);
        isb();
        return ret;
}
```

As the `cb` and set up to only run when visiting leaf nodes (`flags: KVM_PGTABLE_WALK_LEAF`), `hyp_map_walker` allocates pages for each level of the page table and installs the addresses in the corresponding entries.
