---
title: Linux ARM64 `__create_pgd_mapping` 分析
date: 2023-01-17 21:21:15
summary:
tags:
  - Linux
  - page tables
categories: technical
---

> Linux版本：v6.0
>
> 處理器架構：ARMv8

## 前言

在理解 Linux kernel 開機流程時，最難理解的應該就是記憶體映射的部份了，我認為困難的點在於處理記憶體映射的程式本身也在記憶體的映射之中，所以除了看懂程式邏輯之外，還必須理解該部份程式是在怎樣的映射環境中執行，以及他的操作會對記憶體影射造成什麼樣的影響等等。再加上變數中存的位址有些是虛擬位址有些是實體位址，更把事情給複雜化。

這篇介紹`__create_pgd_mapping` ，負責在開機階段建立各級頁表，主要著重在實作的分析，使用場景則先略過。

## `__create_pgd_mapping` 執行環境

在說明函式原型及實作之前，先說明一下執行環境，`__create_pgd_mapping` 是開機流程所使用的工具函式，記憶體的映射狀況為MMU已開啟，使用的pgd(大部分)為`init_pg_dir` ，linear mapping有可能尚未建立所以無法使用，fixmap除了level 3以外的頁表已經建立好，kernel跑在高虛擬位址。

## `__create_pgd_mapping` 函式原型

以下是`__create_pgd_mapping` 的函式原型：

```c
static void __create_pgd_mapping(pgd_t *pgdir, phys_addr_t phys,
                                 unsigned long virt, phys_addr_t size,
                                 pgprot_t prot,
                                 phys_addr_t (*pgtable_alloc)(int),
                                 int flags);
```

* `pgdir` : 想要建立映射的頁表的根(虛擬位址)

* `phys` : 建立映射的物理位址起始位址

* `virt` : 建立映射的虛擬位址起始位址

* `size` : 映射的大小

* `prot` : 映射的屬性

* `pgtable_alloc` : 建立頁表過程中，使用的記憶體分配函式

* `flags` : 一些決定建立頁表過程行為的選項

## `__create_pgd_mapping` 實作

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

沒什麼好講的，取fixmap lock然後呼叫內部`__create_pgd_mapping_locked`

### `__create_pgd_mapping_locked`

```c
static void __create_pgd_mapping_locked(pgd_t *pgdir, phys_addr_t phys,
                                        unsigned long virt, phys_addr_t size,
                                        pgprot_t prot,
                                        phys_addr_t (*pgtable_alloc)(int),
                                        int flags)
{
        unsigned long addr, end, next;
        // 計算轉換virt所用的pgd entry的位址
        pgd_t *pgdp = pgd_offset_pgd(pgdir, virt);

        /*
         * If the virtual and physical address don't have the same offset
         * within a page, we cannot map the region as the caller expects.
         */
        // 由於頁表轉換是以頁為單位，若phys和virt在頁中的offset不一樣則無法建立映射
        if (WARN_ON((phys ^ virt) & ~PAGE_MASK))
                return;
        // 使phys向下對齊頁
        phys &= PAGE_MASK;
        // addr為局部變數，紀錄目前要建立映射的虛擬位址，初始化為virt向下對齊頁
        addr = virt & PAGE_MASK;
        // end紀錄映射結束的虛擬位址，virt + size向上對齊頁
        end = PAGE_ALIGN(virt + size);
        // 這個迴圈一次建立一個pgd entry範圍的映射，若以48bits,4K頁為例就是2^39,
        // 512GB的區間
        do {
                // pgd_addr_end回傳(addr + 一個pgd entry的映射範圍)，或end，
                // 看哪個比較小
                next = pgd_addr_end(addr, end);
                // 把當前pgdp，
                // addr(這個iteration要建立映射的虛擬位址的開頭)
                // next(這個iteration要建立映射的虛擬位址結尾)
                // phys(這個iteration要建立映射的物理位址開頭)
                // prot, pgtable_alloc, flag傳入處理pud層級的函式
                alloc_init_pud(pgdp, addr, next, phys, prot, pgtable_alloc,
                               flags);
                // alloc_init_pud建立addr~next的映射，所以return後把phys
                // 往前推進next - addr
                phys += next - addr;
        // 換下一個pgd entry，也推進addr，如果addr == end則結束
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
        // p4d是如果要使用五層頁表才有的層級，四層以下p4dp = pgdp，目前假設四層頁表
        p4d_t *p4dp = p4d_offset(pgdp, addr);
        // 這邊假設p4d = pgd
        p4d_t p4d = READ_ONCE(*p4dp);
        // 如果pgdp指到的entry是none
        if (p4d_none(p4d)) {
                // 設定這個entry要指到table(指到pud)，且EL0不能執行
                p4dval_t p4dval = P4D_TYPE_TABLE | P4D_TABLE_UXN;
                phys_addr_t pud_phys;
                // 如果flags有NO_EXEC_MAPPING
                if (flags & NO_EXEC_MAPPINGS)
                        // 則EL1也不能執行
                        p4dval |= P4D_TABLE_PXN;
                BUG_ON(!pgtable_alloc);
                // 用傳進來的函式分配一個pud頁
                pud_phys = pgtable_alloc(PUD_SHIFT);
                // 用pud頁的物理位址和p4dval製作成pgd entry存進pgdp
                __p4d_populate(p4dp, pud_phys, p4dval);
                // 讀出來檢查
                p4d = READ_ONCE(*p4dp);
        }
        // 這裡的p4d必須要是正常的pgd entry值
        BUG_ON(p4d_bad(p4d));
        // 這行有點tricky，它負責把pud頁map到一個固定的虛擬位址上，為什麼要這樣做呢？
        // 因為執行到這邊的時候我們從pgdp讀出一個entry，裡頭有pud頁的物理位址，而要存取pud頁
        // 只有物理位址是不行的，要把pud頁map到虛擬位址上程式才能存取它，所以使用fixmap
        // 機制來把pud頁map到一個已知的虛擬位址，接下來就可以操作這個pud頁了
        // pudp = 轉換addr所使用的pud entry的虛擬位址
        pudp = pud_set_fixmap_offset(p4dp, addr);
        do {
                // 讀取pud entry
                pud_t old_pud = READ_ONCE(*pudp);
                // next = ((addr + 一個pud entry的映射範圍)，或end，看哪個小)
                next = pud_addr_end(addr, end);

                /*
                 * For 4K granule only, attempt to put down a 1GB block
                 */
                // 如果可以使用section mapping (一次map 1GB)
                if (pud_sect_supported() &&
                   // 而且目前要map的範圍(addr~next)和物理位址phys都對齊1GB
                   ((addr | next | phys) & ~PUD_MASK) == 0 &&
                    // 而且flags沒說不能用section(block) mapping
                    (flags & NO_BLOCK_MAPPINGS) == 0) {
                        // 在這個層級直接map一個1GB的頁
                        pud_set_huge(pudp, phys, prot);

                        /*
                         * After the PUD entry has been populated once, we
                         * only allow updates to the permission attributes.
                         */
                        // 檢查，跳過
                        BUG_ON(!pgattr_change_is_safe(pud_val(old_pud),
                                                      READ_ONCE(pud_val(*pudp))));
                } else {
                        // pudp: 當前pud entry的虛擬位址
                        // addr: 當前要map的虛擬位址起始
                        // next: 當前要map的虛擬位址終點
                        // phys: 當前要map的物理位址
                        // alloc_init_cont_pmd負責map addr~next
                        alloc_init_cont_pmd(pudp, addr, next, phys, prot,
                                            pgtable_alloc, flags);
                        BUG_ON(pud_val(old_pud) != 0 &&
                               pud_val(old_pud) != READ_ONCE(pud_val(*pudp)));
                }
                // alloc_init_cont_pmd建立addr~next的映射，所以return後把phys
                // 往前推進next - addr
                phys += next - addr;
        // 換下一個pud entry，也推進addr，如果addr == end則結束
        } while (pudp++, addr = next, addr != end);
        // 把剛剛map的pud頁解除映射
        pud_clear_fixmap();
}
```

看到這邊要來解釋一下函式邏輯，目前看到的call chain長這樣：

```bash
__create_pgd_mapping
--> __create_pgd_mapping_locked
    --> alloc_init_pud
        // 以下下面會說明
        --> alloc_init_cont_pmd
            --> init_pmd
                --> alloc_init_cont_pte
                    --> init_pte 
```

上面分析說明了：

`__create_pgd_mapping_locked` 負責填充pgd頁

`alloc_init_pud` 負責填充pud頁

那為什麼pmd頁和pte頁各需要兩個函數來處理呢？原因是ARMv8架構中頁表項有個contiguous bit，簡單來說如果一段連續的虛擬位址會經過頁表轉換出另一段連續的物理位址，軟體可以設置這個bit來優化TLB的表現，`alloc_init_cont_pmd` 和 `alloc_init_cont_pte` 盡量把傳入的位址範圍使用contiguous bit來建立mapping，而`init_pmd` 和`init_pte` 負責實際的pmd頁和pte頁。

### `alloc_init_cont_pmd`

```c
static void alloc_init_cont_pmd(pud_t *pudp, unsigned long addr,
                                unsigned long end, phys_addr_t phys,
                                pgprot_t prot,
                                phys_addr_t (*pgtable_alloc)(int), int flags)
{
        unsigned long next;
        // 讀取傳入的pudp
        pud_t pud = READ_ONCE(*pudp);

        /*
         * Check for initial section mappings in the pgd/pud.
         */
        // pud entry如果是section mapping的話就不會呼叫到我們了，所以代表出錯
        BUG_ON(pud_sect(pud));
        // 如果這個entry還沒建立
        if (pud_none(pud)) {
                // 設定這個entry要指到table(指到pmd)，且EL0不能執行
                pudval_t pudval = PUD_TYPE_TABLE | PUD_TABLE_UXN;
                phys_addr_t pmd_phys;
                // 如果flags有NO_EXEC_MAPPING
                if (flags & NO_EXEC_MAPPINGS)
                        // 則EL1也不能執行
                        pudval |= PUD_TABLE_PXN;
                BUG_ON(!pgtable_alloc);
                // 用傳進來的函式分配一個pmd頁
                pmd_phys = pgtable_alloc(PMD_SHIFT);
                // 用pmd頁的物理位址和pudval製作成pud entry存進pudp
                __pud_populate(pudp, pmd_phys, pudval);
                // 讀出來檢查
                pud = READ_ONCE(*pudp);
        }
        // 這裡的pud必須要是正常的pud entry值
        BUG_ON(pud_bad(pud));

        do {
                pgprot_t __prot = prot;
                // next = ((addr + 一個pmd entry的映射範圍)，或end，看哪個小)
                next = pmd_cont_addr_end(addr, end);

                /* use a contiguous mapping if the range is suitably aligned */
                // 如果addr, next, phys都對齊使用contiguous( bit) mapping的要求
                // e.g. 32MB
                if ((((addr | next | phys) & ~CONT_PMD_MASK) == 0) &&
                    // 而且flags沒說不能用contiguous mapping
                    (flags & NO_CONT_MAPPINGS) == 0)
                        // entry attributes加上contiguous bit
                        __prot = __pgprot(pgprot_val(prot) | PTE_CONT);
                // 進入處理pmd頁的函式
                init_pmd(pudp, addr, next, phys, __prot, pgtable_alloc, flags);
                // init_pmd建立addr~next的映射，所以return後把phys
                // 往前推進next - addr
                phys += next - addr;
        // 推進addr，如果addr == end則結束
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
        // 和前面pud_set_fixmap_offset的操作一樣，從pudp讀取pmd的物理位址，並把物理頁
        // map到一個特定的虛擬位址並回傳addr所使用的pmd entry的位址
        pmdp = pmd_set_fixmap_offset(pudp, addr);
        do {
                // 讀取pmd entry
                pmd_t old_pmd = READ_ONCE(*pmdp);
                // next = ((addr + 一個pmd entry的映射範圍)，或end，看哪個小)
                next = pmd_addr_end(addr, end);

                /* try section mapping first */
                // 如果addr, next, phys都對齊一個pmd section映射大小(通常是2MB)
                if (((addr | next | phys) & ~PMD_MASK) == 0 &&
                    // 而且flags沒說不能用section(block) mapping
                    (flags & NO_BLOCK_MAPPINGS) == 0) {
                        // 在這個層級直接map一個大頁(通常是2MB)
                        pmd_set_huge(pmdp, phys, prot);

                        /*
                         * After the PMD entry has been populated once, we
                         * only allow updates to the permission attributes.
                         */
                        BUG_ON(!pgattr_change_is_safe(pmd_val(old_pmd),
                                                      READ_ONCE(pmd_val(*pmdp))));
                } else {
                        // 進行contiguous pte的操作
                        alloc_init_cont_pte(pmdp, addr, next, phys, prot,
                                            pgtable_alloc, flags);

                        BUG_ON(pmd_val(old_pmd) != 0 &&
                               pmd_val(old_pmd) != READ_ONCE(pmd_val(*pmdp)));
                }
                // alloc_init_cont_pte建立addr~next的映射，所以return後把phys
                // 往前推進next - addr
                phys += next - addr;
        // 換下一個pmd entry，也推進addr，如果addr == end則結束
        } while (pmdp++, addr = next, addr != end);
        // 把剛剛map的pmd頁解除映射
        pmd_clear_fixmap();
}
```

接下來`alloc_init_cont_pte` 和`init_pte` 的操作邏輯跟pmd基本一樣，可以自己研究看看：

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
                // 這行真正把phys和prot組合成pte entry，寫進ptep指向的地方
                // 完成一個頁最終的頁表映射
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
