---
title: "Linux ARM64 KASLR 實作分析(2): 線性位址隨機化"
date: 2023-04-15 20:30:15
summary:
tags:
  - Linux
  - KASLR
  - ARMv8
categories: technical
---

> Linux版本：v6.0
>
> 處理器架構：ARMv8

這篇接續[Linux ARM64 KASLR 實作分析(1): 內核映像位址隨機化](../kaslr/)

## 線性位址隨機化實作機制

### `memstart_offset_seed` 的來由

上篇說到在`__primary_switch` 中，程式呼叫了`__pi_kaslr_early_init` 來獲得KASLR使用的隨機偏移，並且把低20位存在暫存器x24中。接著在 `__primary_switched` 中把x24的值存到`memstart_offset_seed`: 

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

`memstart_offset_seed` 在`arch/arm64/kernel/kaslr.c` 中被定義:

```c
u16 __initdata memstart_offset_seed;
```

可以看到它是個u16，所以上面使用`strh`，使得其剩下16位的隨機值。

### `arm64_memblock_init`

接下來的初始化流程會執行到`arm64_memblock_init` 函式

```c
start_kernel
 --> setup_arch
      --> arm64_memblock_init
      --> paging_init 
```

擷取與KASLR有關的部份：

```c
void __init arm64_memblock_init(void)
{

        [...]

        /*
         * Select a suitable value for the base of physical memory.
         */
        // 設定memstart_addr為DRAM開始的位址(然後做一個對齊)
        memstart_addr = round_down(memblock_start_of_DRAM(),
                                   ARM64_MEMSTART_ALIGN);

        [...]

        // 如果KASLR開啟
        if (IS_ENABLED(CONFIG_RANDOMIZE_BASE)) {
                // 取得16位的隨機值
                extern u16 memstart_offset_seed;
                // 讀取CPU能夠支援的物理位址區間大小
                u64 mmfr0 = read_cpuid(ID_AA64MMFR0_EL1);
                int parange = cpuid_feature_extract_unsigned_field(
                                        mmfr0, ID_AA64MMFR0_PARANGE_SHIFT);
                // linear_region_size為kernel計算出之線性映射區間大小
                // linear_region_size減去物理位址區間大小，得到range
                // 所以這個range代表物理位址在線性映射區間中有多少移動空間
                s64 range = linear_region_size -
                            BIT(id_aa64mmfr0_parange_to_phys_shift(parange));
    
                /*
                 * If the size of the linear region exceeds, by a sufficient
                 * margin, the size of the region that the physical memory can
                 * span, randomize the linear region as well.
                 */
                // 如果隨機值非0，而且移動空間大於ARM64_MEMSTART_ALIGN (隨機化空間夠大)
                // 則用改變memstart_addr的方式來隨機化線性空間
                if (memstart_offset_seed > 0 && range >= (s64)ARM64_MEMSTART_ALIGN) {
                        range /= ARM64_MEMSTART_ALIGN;
                // 下面這行要減的值不是很好理解，可以用數學的方式重新排列想像成
                // (range * ARM64_MEMSTART_ALIGN) * (memstart_offset_seed >> 16)
                // 也就是 (range原本的值，因為前面range被除ARM64_MEMSTART_ALIGN) * 
                // (memstart_offset_seed / (1 << 16) 也就是一個0~1的隨機值)
                // 所以算下來就是一個 (0~原本的range) 的隨機值
                        memstart_addr -= ARM64_MEMSTART_ALIGN *
                                         ((range * memstart_offset_seed) >> 16);
                }
        }
 
        [...]

}
```

那為何改變`memstart_addr` 就會改變線性空間位址呢？

### 線性映射隨機化

我們來看一下真正要隨機化線性空間的頁表操作。線性空間的頁表是在`paging_init` → `map_mem`中建立的：

```c
static void __init map_mem(pgd_t *pgdp)
{

        [...]

        /* map all the memory banks */
        // 這個迴圈把系統對所有物理記憶體區間都呼叫__map_memblock製作線性映射
        for_each_mem_range(i, &start, &end) {
                if (start >= end)
                        break;
                /*
                 * The linear map must allow allocation tags reading/writing
                 * if MTE is present. Otherwise, it has the same attributes as
                 * PAGE_KERNEL.
                 */
                // start為起始物理位址
                // end為結束的物理位址
                // pgdp為swapper_pg_dir，kernel之後即將使用的頁表的根
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

`__create_pgd_mapping` 在 [Linux ARM64 `__create_pgd_mapping` 分析](../page_table/) 有詳細的介紹，這邊重點在於`[start - end)` 這段物理位址區間在這裡被建立了一個到`[__phys_to_virt(start) - __phys_to_virt(end))` 的頁表映射，也就是說`__phys_to_virt` 決定了線性映射的虛擬位址

### `__phys_to_virt`

macro定義在`arch/arm64/include/asm/memory.h` 中：

```c
#define __phys_to_virt(x)       ((unsigned long)((x) - PHYS_OFFSET) | PAGE_OFFSET)
```

而在同個檔案中 `PHYS_OFFSET` 基本被定義為`memstart_addr`

```c
extern s64                      memstart_addr;
/* PHYS_OFFSET - the physical address of the start of memory. */
#define PHYS_OFFSET             ({ VM_BUG_ON(memstart_addr & 1); memstart_addr; })
```

這樣子就接回`memstart_addr` 了，`memstart_addr` 在`arm64_memblock_init` 中被減去一個隨機值，導致`PHYS_OFFSET` 和`__phys_to_virt` 被隨機化，所以`paging_init` → `map_mem` → `___map_memblock` → `__create_pgd_mapping` 製作的線性映射的虛擬位址也就被隨機化了。
