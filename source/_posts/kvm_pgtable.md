---
title: "KVM ARM: 新 page table walker"
date: 2022-11-03 17:20:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
  - page tables
categories: technical
---

> Linux版本：v6.0
>
> 處理器架構：ARMv8

## 前言

在Linux kernel 5.10週期，KVM ARM開發者們為了為[google pkvm](https://www.youtube.com/watch?v=wY-u6n75iXc)做準備，在code base許多地方做了翻修，今天就是介紹其中新設計的page table walker。

原先在KVM ARM中在做page table walk的時候，寫法就是單純的在需要的地方直接一路access然後dereference下去，e.g. `create_hyp_{p4d, pud, pmd, pte}_mappings` 這幾個函式。這樣做的缺點之一就是軟體在存取page tables時code很難重複使用，而新的作法在5.10中出現，把存取page table這樣的操作模組化，各個需要存取page table的地方都能共用同樣的程式。

## 重要結構

使用新的page table walker時，需要提供一些資訊：

1. 所要存取的page table (`struct kvm_pgtable`)

2. 想要對page table進行的操作，以及walk到哪裡時進行 (`struct kvm_pgtable_walker`)

3. 訪問哪個虛擬位址範圍 (`struct kvm_pgtable_walk_data`)

見以下說明：

### `kvm_pgtable`

紀錄一整個page table tree的metadata，以下簡單說明幾個重要的成員

```c
struct kvm_pgtable {  
        u32                                     ia_bits; // 這個page table所翻譯的virtual address是幾個bits
        u32                                     start_level; // 從第幾層開始
        kvm_pte_t                               *pgd; // 重要成員：root page table的linear map address
        struct kvm_pgtable_mm_ops               *mm_ops; // 操作此page table的相關函式e.g.申請&釋放記憶體
                                  
        /* Stage-2 only */  
        struct kvm_s2_mmu                       *mmu; // 略
        enum kvm_pgtable_stage2_flags           flags; // 略
        kvm_pgtable_force_pte_cb_t              force_pte_cb; // 略
};  
```

### `kvm_pgtable_walker`

提供使用者設定訪問page table時呼叫的函數以及何時呼叫，`cb` 代表"call back"

```c
struct kvm_pgtable_walker {
        const kvm_pgtable_visitor_fn_t          cb; // 在walk這個page table tree時會呼叫的函式
        void * const                            arg; // 傳遞給cb的參數
        const enum kvm_pgtable_walk_flags       flags; // 設定什麼時候要呼叫cb，有三個不互斥的選項：
// 1. KVM_PGTABLE_WALK_LEAF: 走到葉節點時呼叫
// 2. KVM_PGTABLE_WALK_TABLE_PRE: 訪問子節點前呼叫
// 3. KVM_PGTABLE_WALK_TABLE_POST: 訪問子節點後呼叫
};
```

### `kvm_pgtable_walk_data`

想要訪問的位址區間，可以看出`kvm_pgtable_walk_data` 實際上包含了前兩者的資訊

```c
struct kvm_pgtable_walk_data {
        struct kvm_pgtable              *pgt; // 指向要存取的page table metadata
        struct kvm_pgtable_walker       *walker; // 指向使用的walker

        u64                             addr; // walk起始位址
        u64                             end; // walk結束位址
};
```

## 新walker實作

接著就可以來看walker的入口點`kvm_pgtable_walk`:

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

這個函式預期`pgt` 和`walker` caller已經準備好了，再加上利用傳進來的`addr` 和`size` 製作出這次所使用的`kvm_pgtalbe_walk_data`，然後呼叫`_kvm_pgtable_walk`。

### `_kvm_pgtable_walk`

這個函式負責各個root page，見註解

```c
static int _kvm_pgtable_walk(struct kvm_pgtable_walk_data *data)
{
        u32 idx;
        int ret = 0;
        struct kvm_pgtable *pgt = data->pgt;
        u64 limit = BIT(pgt->ia_bits);

        if (data->addr > limit || data->end > limit) // 範圍檢查
                return -ERANGE;

        if (!pgt->pgd) // 檢查有沒有root page table
                return -EINVAL;
        // 在某些系統設定之下，ARMv8的stage 2 page table root page可以不只一頁，
        // 而是連續多頁(< 16頁)
        // 這個for loop就是loop過這n個root page, idx: 0 ~ n-1
        for (idx = kvm_pgd_page_idx(data); data->addr < data->end; ++idx) {
                // 每個循環ptep指向當前root page
                kvm_pte_t *ptep = &pgt->pgd[idx * PTRS_PER_PTE];
                // 呼叫更底層的實作函式
                ret = __kvm_pgtable_walk(data, ptep, pgt->start_level);
                if (ret)
                        break;
        }
 
        return ret;
}
```

### `__kvm_pgtable_walk`

這個函式loop過一個page的entries

```c
static int __kvm_pgtable_walk(struct kvm_pgtable_walk_data *data,
                >             kvm_pte_t *pgtable, u32 level)
{
        u32 idx;
        int ret = 0;
        // 如果當前的page table level超過最大值(4)就錯誤退出
        if (WARN_ON_ONCE(level >= KVM_PGTABLE_MAX_LEVELS))
                return -EINVAL;
        // 利用kvm_pgtable_idx計算開始訪問的index
        for (idx = kvm_pgtable_idx(data, level); idx < PTRS_PER_PTE; ++idx) {
                // 計算當前看的entry的位址
                kvm_pte_t *ptep = &pgtable[idx];
                // 如果走完範圍了就退出
                if (data->addr >= data->end)
                        break;
                // 把data和當前看的位址以及當前的level當作輸入
                // 呼叫實際操作的函式
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
        // pte: 當前entry的"值"
        kvm_pte_t *childp, pte = *ptep;
        // 這個entry指到的是不是一個table?
        bool table = kvm_pte_table(pte, level);
        enum kvm_pgtable_walk_flags flags = data->walker->flags;
        // 如果entry指到table而且訪問子節點前要呼叫
        if (table && (flags & KVM_PGTABLE_WALK_TABLE_PRE)) {
                // kvm_pgtable_visitor_cb只是一個簡單的wrapper幫忙
                // 用傳入的參數呼叫walker->cb
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_TABLE_PRE);
        }
        // 如果entry指到physical page(而不是下一層page table)
        // 而且訪問葉節點要呼叫cb
        if (!table && (flags & KVM_PGTABLE_WALK_LEAF)) {
                // 呼叫cb
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_LEAF);
                // 由於ptep這個位址的內容有可能在呼叫cb時改變，例如cb allocate了下一層
                // page table讓這個entry指到它，所以這邊更新pte的值
                pte = *ptep;
                // 也更新這個pte現在是不是指到table
                table = kvm_pte_table(pte, level);
        }

        if (ret)
                goto out;
        // 如果不是table(是葉節點)
        if (!table) {
                // 使data->addr對齊現在這個page table level translation的大小
                data->addr = ALIGN_DOWN(data->addr, kvm_granule_size(level));
                // 並使data->addr跨過現在這個level translation大小
                data->addr += kvm_granule_size(level);
                // 處理完葉節點之後就可以退出
                goto out;
        }
        // 執行到這裡代表當前entry指到table，所以使用kvm_pte_follow獲取下一層page table的位址
        // 值得注意的是entry是指向下一層table的實體位址，所以kvm_pte_follow裡面使用了
        // phys_to_virt把entry轉換成軟體存取的EL1虛擬記憶體位址
        childp = kvm_pte_follow(pte, data->pgt->mm_ops);
        // 呼叫__kvm_pgtable_walk去走下一層page table
        // 這邊的程式邏輯見下方說明
        ret = __kvm_pgtable_walk(data, childp, level + 1);
        if (ret)
                goto out;
        // 走完當前table entry而且訪問子節點之後要呼叫cb
        if (flags & KVM_PGTABLE_WALK_TABLE_POST) {
                ret = kvm_pgtable_visitor_cb(data, addr, level, ptep,
                                             KVM_PGTABLE_WALK_TABLE_POST);
        }

out:
        return ret;
}
```

Page table其實是一個多child的樹狀結構(4K page就是512個child)，這個page table walker使用了遞迴的方式來對其進行操作，還支持pre-order, post-order的邏輯，`level`代表第幾層，`__kvm_pgtable_visit`負責：

1. 在適當的條件對table進行操作(呼叫callbacks)
2. 推進`data->addr`的進度
3. 遇到`table`的時候遞迴呼叫`__kvm_pgtable_walk`

而`__kvm_pgtable_walk`負責一個page table裡面的entry的loop。

## 使用範例：`create_hyp_mappings`

Linux在初始化KVM的時候的執行模式是EL1，此時需要在進入EL2之前為其製作和設定好EL2所使用的page tables，使用的函式就是`create_hyp_mappings`。

`create_hyp_mappings`在KVM初始化重點函式之一`init_hyp_mode` 中被多次呼叫，分別替EL2 建立了以下幾個區域的mappings：

* EL2 code (`__hyp_text_start` \~ `__hyp_text_end`)
* EL2 read only data (`__hyp_rodata_start` \~ `__hyp_rodata_end`)
* EL1 read only data (`__start_rodata` \~ `__end_rodata`)
* EL2 BSS (`__hyp_bss_start` \~ `__hyp_bss_end`)
* EL1 BSS (`__hyp_bss_end` \~ `__bss_stop`)
* EL2 stack
* EL2 percpu area

> 這個函式只是建立page tables，並不會進到EL2啟動EL2的address translation機制

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
        // 使用kern_hyp_va把輸入的EL1虛擬位址始末轉換成EL2虛擬位址
        unsigned long start = kern_hyp_va((unsigned long)from);
        unsigned long end = kern_hyp_va((unsigned long)to);
        // 判斷處理器是否處於EL2 (VHE模式)
        if (is_kernel_in_hyp_mode())
                return 0;
        // pkvm相關，略
        if (!kvm_host_owns_hyp_mappings())
                return -EPERM;
        // 將起始與結束位址對齊頁邊界
        start = start & PAGE_MASK;
        end = PAGE_ALIGN(end);
        // 迴圈loop過輸入範圍，每一次迴圈會利用`kvm_kaddr_to_phys`計算對應的物理位址，呼叫`__create_hyp_mappings` ，並前進一個`PAGE_SIZE`
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

註解其實就對這個函式有不少說明，作為輸入的`from`, `to` 是想要map給EL2的EL1虛擬位址區間，`prot` 則是讀寫執行等權限設定。有趣的是並不需要指定要map給EL2的虛擬位址，EL2的虛擬位址規劃KVM自身有機制決定，具體來說就是利用`kern_hyp_va` 把EL1的虛擬位址轉換成EL2的虛擬位址。

### `__create_hyp_mappings`

鎖上`kvm_hyp_pgd_mutex` 然後呼叫`kvm_pgtable_hyp_map`，注意`hyp_pgtable`即為page table walk所需的`kvm_pgtable`

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
這個函式就會呼叫新的page table walker `kvm_pgtable_walk`了，呼叫之前製作所需的`kvm_pgtable_walker`(1)，和傳入的`hyp_pgtable`(參數`pgt`)一起傳給`kvm_pgtable_walk`(2)
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

可想而知，作為`cb` ，只在碰到葉節點(`flags: KVM_PGTABLE_WALK_LEAF`)會被呼叫的`hyp_map_walker` 就會負責申請各級的page tables並安裝進適當的page table entries。

