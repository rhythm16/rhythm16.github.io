---
title: "Linux ARM64 KASLR 實作分析(1): 內核映像位址隨機化"
date: 2022-12-22 14:20:15
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

## 前言

學作業系統其實很早就聽過有KASLR(Kernel Address Space Layout Randomization)這個東西了，聽起來就很酷，不過因為程度尚淺，再加上KASLR這種安全相關主題在學作業系統時不算是早期就會接觸到的核心功能，例如虛擬記憶體，檔案系統，CPU排程等等，所以一直沒有機會研究其實作機制。之前有空的時候有稍微深入google一些網路上的相關文章，可惜的是大多數KASLR相關資源只有概念上的解釋，或者非常簡單的帶過，缺的就是一個深入說明實作的文章。

最近剛好有點時間，就趁機trace了這個我一直都很好奇的機制，會以兩篇文章進行簡單的總結，第一篇是內核映像位址的隨機化，第二篇是線性位址的隨機化，希望有所幫助。

> 若沒有arm64 Linux kernel開機流程的概念，這篇應該會很難看懂

## KASLR實作機制

KASLR實作和ARM64 Linux kernel開機流程有緊密的關係，不過這篇目的是介紹KASLR而不是開機流程，所以會先說明執行環境，再從開機非常早期的`__primary_switch` 開始用程式講解。

### 執行環境說明

開機到`__primary_switch` 時MMU尚未開啟，CPU各個重要的系統暫存器已經初始化完成，identity map的頁表也已經初始化，`x22` 存著FDT (Flat Device Tree)的位址。

### `__primary_switch`

`__primary_switch` 的意思是"primary CPU switch on MMU"

```c
SYM_FUNC_START_LOCAL(__primary_switch)
        adrp    x1, reserved_pg_dir
        adrp    x2, init_idmap_pg_dir
        // 開啟MMU，使用init_idmap_pg_dir裡準備好的identity map
        bl      __enable_mmu
#ifdef CONFIG_RELOCATABLE
        // x23 = kernel image開頭的實體位址
        adrp    x23, KERNEL_START
        // x23 = x23 % 2MB，大概率等於0，因為規定kernel實體對齊 = 2MB
        and     x23, x23, MIN_KIMG_ALIGN - 1
#ifdef CONFIG_RANDOMIZE_BASE
        // x0 = FDT 位址，當作__pi_kaslr_early_init的參數
        mov     x0, x22
        // 臨時使用init_pg_end當作stack空間
        adrp    x1, init_pg_end
        mov     sp, x1
        // 不太確定這行指令的意義，可能就簡單清零
        mov     x29, xzr
        // 見下方說明，返回之後x0 = KASLR使用的隨機偏移
        bl      __pi_kaslr_early_init
        // x24 = x0 % 2MB
        and     x24, x0, #SZ_2M - 1             // capture memstart offset seed
        // x0 = x0 往下2MB對齊
        bic     x0, x0, #SZ_2M - 1
        // x23 = x23 + x0 = 0 + 隨機偏移
        orr     x23, x23, x0                    // record kernel offset
#endif
#endif
        // 對KASLR不重要
        bl      clear_page_tables
        // 這個呼叫建立kernel要跳到高位址之後所使用的mapping
        // 重要的是這個函式使用KIMAGE_VADDR + x23來算出kernel運行時
        // 的虛擬位址來建立RWX mapping，所以這個頁表的映射是已經隨機化的了
        bl      create_kernel_mapping

        adrp    x1, init_pg_dir
        // 在高位址使用這個頁表
        load_ttbr1 x1, x1, x2
#ifdef CONFIG_RELOCATABLE
        // 見下方說明
        bl      __relocate_kernel
#endif
        // 見下方說明(__primary_switch末尾)
        ldr     x8, =__primary_switched
        adrp    x0, KERNEL_START                // __pa(KERNEL_START)
        br      x8
SYM_FUNC_END(__primary_switch)
```

### `kaslr_early_init`

`kaslr_early_init` 所在的檔案`arch/arm64/kernel/pi/kaslr_early.c` 會因為Makefile的操作而使得所有裡面定義的符號都多上一個`__pi_` 前綴，所以上面使用`bl __pi_kaslr_early_init` 呼叫。

```c
asmlinkage u64 kaslr_early_init(void *fdt)
{
        u64 seed;
        // KASLR若於cmaline中被關閉則退出
        if (is_kaslr_disabled_cmdline(fdt))
                return 0;
        // 從傳入的FDT位址分析是否有random seed 
        seed = get_kaslr_seed(fdt);
        // 若沒有：
        if (!seed) {
                // 如果CPU沒有生成隨機數的能力，或者取得隨機數失敗則退出
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
        // 我想了很久覺得這行的詳細解釋實在是不容易，牽扯到Linux的整體記憶體布局，
        // 決定先簡單解釋就好：
        // 這行使用random seed計算出一個適合的KASLR隨機偏移，並回傳
        return BIT(VA_BITS_MIN - 3) + (seed & GENMASK(VA_BITS_MIN - 3, 0));
}
```

### `__relocate_kernel`

去翻`__relocate_kernel` 實際的程式，會發現我把`CONFIG_RELR` 的部份拿掉了，這是因為我沒有研究那塊XD，所以無法和大家講解，不過不影響分析，而`defconfig` 也沒有用這個設定所以目前先跳過

這個函式是KASLR非常重要的一個函式，它負責runtime relocation(執行時重定位)，Linux Kernel 編譯過程會在每個存取全局位址的地方產生一個relocation entry，而工具鏈在生成有KASLR功能的內核時，因為不知道最後執行的位址，所以必須要為直接使用全局位址的地方都生成一個relocation entry，讓動態鏈接器(kernel沒有，所以就是kernel自己)能夠在執行時，知道運行時位址之後進行重定位。 這些relocation entries會被集中起來成為kernel的一部分，`__relocate_kernel` 會iterate過所有的entries並且進行重定位。 

> [這個reference](https://github.com/ARM-software/abi-aa/blob/main/aaelf64/aaelf64.rst) 記載了ARM relocation entry的各種格式，這邊用到的只有一種。

 ELF relocation entry的格式如下：

```c
typedef struct {
        // 需要重定位的位址，注意這個位址是編譯時決定的位址
        Elf64_Addr      r_offset;
        // 符號表索引以及重定位種類
        Elf64_Xword     r_info;
        // 一個重定位過程需要加上的值，意義由重定位種類決定
        Elf64_Sxword    r_addend;
} Elf64_Rela;
```

KASLR只處理一種重定位，種類是`R_AARCH64_RELATIVE` ，這個種類的重定位處理方式為把`r_offset` 這個編譯虛擬位址對應的運行虛擬位址指向的地方改成：”`r_addend` 加上隨機偏移(也就是運行位址和編譯位址的差)”。這樣講可能很難理解，我們用一個例子說明，今天某個指令想要把`sym` 這個符號的位址load進暫存器`x0` ，那麼假設編譯結果是`ldr x0, #a0` ，這裡`#a0` 只是舉例，假設`sym` 這個符號在編譯時決定的位址是`0x500` ，假設`ldr x0, #a0` 指令的位址是`0x1000`，這個case產生的relocation entry就是`r_offset = 0x10a0` ，`r_addend = 0x500` 。再假設這個指令運行時位址是`0x3000` (所以`sym` 運行位址是`0x2500`) ，和編譯時差(偏移)`0x2000`，我們需要做的事情就是讀出`r_addend (0x500)` ，加上`0x2000`(運行和編譯的偏移)，然後存到`r_offset` \+ (運行和編譯的偏移)，也就是把`0x2500` 存到`0x10a0 + 0x2000 = 0x30a0` 這個位址。真正運行時`x0` 經過`ldr` 就會存著`0x2500` ，也就是運行時`sym` 的位址。

```c
SYM_FUNC_START_LOCAL(__relocate_kernel)    
        /*    
         * Iterate over each entry in the relocation table, and apply the    
         * relocations in place.    
         */    
        // x9 = 第一個entry的實體位址
        adr_l   x9, __rela_start  
        // x10 = 最後一個entry的結束的實體位址
        adr_l   x10, __rela_end    
        // x11 = kernel預設的虛擬位址
        mov_q   x11, KIMAGE_VADDR               // default virtual offset    
        // x11 += 隨機偏移，製作出隨機虛擬位址
        add     x11, x11, x23                   // actual virtual offset    
        // x9 >= x10就跳到結束的地方
0:      cmp     x9, x10    
        b.hs    1f    
        // x12 = r_offset, x13 = r_info 並把x9移到下一個entry
        ldp     x12, x13, [x9], #24    
        // x14 = r_addend
        ldr     x14, [x9, #-8]   
        // r_info是R_AARCH64_RELATIVE嗎
        cmp     w13, #R_AARCH64_RELATIVE    
        // 不是就跳過，處理下一個entry
        b.ne    0b    
        // x14 = r_addend + 隨機偏移
        add     x14, x14, x23                   // relocate  
        // 把(r_addend + 隨機偏移)這個值寫入(r_offset + 隨機偏移)這個地方，
        // 完成重定位
        str     x14, [x12, x23]    
        b       0b    
1:
        ret
SYM_FUNC_END(__relocate_kernel)
```

### `__primary_switch` 末尾

`bl      __relocate_kernel` 結束之後馬上就出現了一個全局位址的讀取：

```c
        // 這個讀取會產生一個relocation entry，而就在__relocate_kernel被重定位，
        // 所以x8會拿到一個加過隨機偏移的__primary_switched的位址，
        // 而create_kernel_mapping建立了有加上隨機偏移的mapping，
        // 也在load_ttbr1 x1, x1, x2 安裝進ttbr1_el1，一切都打理完畢，
        // 可以br跳到高位址繼續執行了
        ldr     x8, =__primary_switched
        adrp    x0, KERNEL_START                // __pa(KERNEL_START)
        br      x8
```

## 鏈接選項

ARM64 Makefile 中有特別針對KASLR的鏈接選項：

```c
ifeq ($(CONFIG_RELOCATABLE), y)                                   
# Pass --no-apply-dynamic-relocs to restore pre-binutils-2.27 behaviour                                   
# for relative relocs, since this leads to better Image compression                                   
# with the relocation offsets always being zero.                                   
LDFLAGS_vmlinux         += -shared -Bsymbolic -z notext \                                   
                        $(call ld-option, --no-apply-dynamic-relocs)                                   
endif                                   
```

有四個選項：

* `—no-apply-dynamic-relocs` 似乎和跳過的`CONFIG_RELR` 有關，不確定

* `-shared` 代表產生一個shared library

* `-Bsymbolic` 代表綁定這個shared library裡面所有的全局變數引用到這個shared library本身，目的應該是因為kernel不會有外部引用，所以不要讓`ld` 產生外部引用的relocation entry

* `-z notext` 讀了`ld` 的man page還是不大確定

## 一些疑問

雖然經過上面的分析可以了解KASLR的運作方式，但也引出了不少問題，希望之後繼續累積能夠回答它們吧

* 上面四個鏈接選項實際意義是什麼？加上與否的差別是什麼？

* 我原本以為關閉KASLR以後所有的relocation entry都會消失，但經過測試他們依然存在，這雖然不影響非KASLR kernel的運作但難道不會浪費空間嗎？

* Linux kernel如何只讓工具鏈只生成一種`R_AARCH64_RELATIVE` 的relocation type？哪種情況會出現其他種類？
