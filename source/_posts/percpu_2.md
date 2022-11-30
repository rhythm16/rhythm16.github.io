---
title: "KVM ARM: EL2 per cpu變數(2): 初始化"
date: 2022-11-24 20:54:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
  - per cpu variables
categories: technical
---

> Linux版本：v6.0
>
> 處理器架構：ARMv8
>
> KVM品種：NVHE

## 前言

上一篇說明了KVM ARM EL2 per cpu變數如何宣告與使用。簡單複習一下，EL2 per cpu變數的存取方式是先取得被放在`.hyp.data..percpu` section的基地址，然後加上存放在`tpidr_el2` 的offset算出最終的位址。經過這樣的講解，有兩個自然的問題：

1. EL2 per cpu變數使用的記憶體是如何分配的？

2. 關鍵的`tpidr_el2` 裡頭存的offset是如何計算並存進去的？

這就是本篇要解答的內容，start!

## 記憶體分配

KVM ARM初始化的重要函式`init_hyp_mode` 中會替EL2 per cpu變數分配記憶體：

```c
/*
 * Allocate and initialize pages for Hypervisor-mode percpu regions.
 */
// 對每一個可用的cpu:
for_each_possible_cpu(cpu) {
        struct page *page;
        void *page_addr;
        // 使用alloc_pages分配EL2 per cpu記憶體
        // nvhe_per_cpu_order()計算per cpu記憶體有多大，單位是幾個頁，
        // 然後取log2 e.g. 8頁 = 2^3頁，回傳3.
        page = alloc_pages(GFP_KERNEL, nvhe_percpu_order());
        if (!page) {
                err = -ENOMEM;
                goto out_err;
        }
        // 把alloc_pages回傳的struct page指標轉換成該頁之線性地址
        page_addr = page_address(page);
        // 把.hyp.data..percpu裡原本的內容複製到剛分配出的記憶體，以免
        // 有已經初始化過的變數
        // CHOOSE_NVHE_SYM macro用於把符號名稱改為nvhe的命名空間
        // CHOOSE_NVHE_SYM(__per_cpu_start)可以先理解為.hyp.data..percpu的開頭
        memcpy(page_addr, CHOOSE_NVHE_SYM(__per_cpu_start), nvhe_percpu_size());
        // 把分配出的線性位址存進一個EL1的陣列
        kvm_arm_hyp_percpu_base[cpu] = (unsigned long)page_addr;
}
```

分配完記憶體之後就是要把那些空間map給EL2(一樣在`init_hyp_mode`)：

```c
// 對每一個可用的cpu:
for_each_possible_cpu(cpu) {
        // 取得當前cpu應該map給EL2的per cpu空間(EL1線性地址)
        char *percpu_begin = (char *)kvm_arm_hyp_percpu_base[cpu];
        char *percpu_end = percpu_begin + nvhe_percpu_size();

        /* Map Hyp percpu pages */
        // 應該不用說明
        err = create_hyp_mappings(percpu_begin, percpu_end, PAGE_HYP);
        if (err) {
                kvm_err("Cannot map hyp percpu region\n");
                goto out_err;
        }

        /* Prepare the CPU initialization parameters */
        // 準備各個之後要在EL2安裝的系統暫存器的值(見下節)
        cpu_prepare_hyp_mode(cpu);
}
```

## 設定tpidr_el2

前面說明了如何分配EL2 per cpu變數使用的記憶體與替EL2建造page table的過程，接下來要做的就是計算各個cpu的offset還有實際安裝該值進`tpidr_el2` 。

### 計算per cpu offset

`cpu_prepare_hyp_mode(cpu)` 負責的工作在於填充一個`struct kvm_nvhe_init_params` ，這個結構存放初始化EL2各個系統暫存器的值，也就包含了`tpidr_el2` 該存的offset，列出目前關心的部份：

```c
static void cpu_prepare_hyp_mode(int cpu)
{
        // 使用per_cpu_ptr_nvhe_sym取得當前cpu EL2 per cpu變數在EL1的線性位址，
        // 這邊取得kvm_init_params這個per cpu變數
        struct kvm_nvhe_init_params *params = per_cpu_ptr_nvhe_sym(kvm_init_params, cpu);
        
        [...]

        /*
         * Calculate the raw per-cpu offset without a translation from the
         * kernel's mapping to the linear mapping, and store it in tpidr_el2
         * so that we can use adr_l to access per-cpu variables in EL2.
         * Also drop the KASAN tag which gets in the way...
         */
        // 這邊就是計算offset的地方了，
        // 被減數：使用alloc_pages分配出的per cpu區域的開頭的EL1線性位址
        // 減數：基per cpu區域的開頭的EL1線性位址
        params->tpidr_el2 = (unsigned long)kasan_reset_tag(per_cpu_ptr_nvhe_sym(__per_cpu_start, cpu)) -
                            (unsigned long)kvm_ksym_ref(CHOOSE_NVHE_SYM(__per_cpu_start));

        [...]
}
```

### 安裝`tpidr_el2`

把offset存進`params->tpidr_el2`之後，接著就是進入EL2並把該值存入`tpidr_el2`系統暫存器，先來看一下KVM ARM初始化過程中和目前主題有關的call stack:

```
kvm_init()
--> kvm_arch_init()
    --> init_subsystems()
        --> on_each_cpu(_kvm_arch_hardware_enable())
            --> cpu_hyp_reinit()
                --> cpu_init_context()
                    --> cpu_init_hyp_mode()
                        --> hyp_install_host_vector()
                            --> ___kvm_hyp_init
```

`hyp_install_host_vector` 跑在EL1，會呼叫hypercall 同時傳入EL2初始化需要用到的`struct kvm_nvhe_init_param` ，接著進入EL2並執行`___kvm_hyp_init`做EL2相關設定，其中就包含設定`tpidr_el2` 。

```c
static void hyp_install_host_vector(void)
{
        struct kvm_nvhe_init_params *params;
        struct arm_smccc_res res;

        [...]
        // 取得當前cpu的`kvm_init_params`的EL1位址
        params = this_cpu_ptr_nvhe_sym(kvm_init_params);
        // 理解成呼叫`hvc`，傳入
        // 1. `__kvm_hyp_init`代表的數字，
        // 2. 當前cpu的`kvm_init_params`的物理位址
        // 3. 局部變數`res`的位址，用於回傳執行結果是否成功
        arm_smccc_1_1_hvc(KVM_HOST_SMCCC_FUNC(__kvm_hyp_init), virt_to_phys(params), &res);
        WARN_ON(res.a0 != SMCCC_RET_SUCCESS);
}
```

hvc那行如何進入EL2的細節先不討論，簡而言之`x0`會放入`__kvm_hyp_init`代表的數字(讓EL2的程式可以確認EL1為何呼叫`hvc` )，`x1` 會放入`kvm_init_params` 的物理位址，然後會跳到這邊：

```c
// arch/arm64/kvm/hyp/nvhe/hyp-init.S
        /*
         * Only uses x0..x3 so as to not clobber callee-saved SMCCC registers.
         *
         * x0: SMCCC function ID
         * x1: struct kvm_nvhe_init_params PA
         */
__do_hyp_init:
        /* Check for a stub HVC call */
        // 比較x0是否為stub hvc call的id (不是)
        cmp     x0, #HVC_STUB_HCALL_NR
        b.lo    __kvm_handle_stub_hvc
        // 比較x0是否為__kvm_hyp_init的ID (是)
        mov     x3, #KVM_HOST_SMCCC_FUNC(__kvm_hyp_init)
        cmp     x0, x3
        // 跳到1:
        b.eq    1f
    
        mov     x0, #SMCCC_RET_NOT_SUPPORTED
        eret
        // param的物理位址放到x0
1:      mov     x0, x1
        // 把存放return address的lr放到x3 以免被___kvm_hyp_init覆蓋掉
        // (___kvm_hyp_init)不會動到x3
        mov     x3, lr
        // 跳去內部處理函式
        bl      ___kvm_hyp_init   // Clobbers x0..x2
        // 復原lr
        mov     lr, x3
    
        /* Hello, World! */
        // 把成功的值放進x0
        mov     x0, #SMCCC_RET_SUCCESS
        // 回到EL1
        eret
```

最後看一下`___kvm_hyp_init` :

其實前兩行就是設定`tpidr_el2` 了，給你們揣摩它怎麼作到的吧，這裡也可以看到這個函式實際設定EL2執行環境的實作，至於講解細節的話…有空再說吧XD

> 這裡值得說明的是這邊EL2的MMU還沒開，所以都是直接使用實體位址進行操作

```c
SYM_CODE_START_LOCAL(___kvm_hyp_init)
        ldr     x1, [x0, #NVHE_INIT_TPIDR_EL2]
        msr     tpidr_el2, x1

        ldr     x1, [x0, #NVHE_INIT_STACK_HYP_VA]
        mov     sp, x1

        ldr     x1, [x0, #NVHE_INIT_MAIR_EL2]
        msr     mair_el2, x1

        ldr     x1, [x0, #NVHE_INIT_HCR_EL2]
        msr     hcr_el2, x1

        ldr     x1, [x0, #NVHE_INIT_VTTBR]
        msr     vttbr_el2, x1

        ldr     x1, [x0, #NVHE_INIT_VTCR]
        msr     vtcr_el2, x1

        ldr     x1, [x0, #NVHE_INIT_PGD_PA]
        phys_to_ttbr x2, x1
alternative_if ARM64_HAS_CNP
        orr     x2, x2, #TTBR_CNP_BIT
alternative_else_nop_endif
        msr     ttbr0_el2, x2

        /*
         * Set the PS bits in TCR_EL2.
         */
        ldr     x0, [x0, #NVHE_INIT_TCR_EL2]
        tcr_compute_pa_size x0, #TCR_EL2_PS_SHIFT, x1, x2
        msr     tcr_el2, x0

        isb

        /* Invalidate the stale TLBs from Bootloader */
        tlbi    alle2
        tlbi    vmalls12e1
        dsb     sy

        mov_q   x0, INIT_SCTLR_EL2_MMU_ON
alternative_if ARM64_HAS_ADDRESS_AUTH
        mov_q   x1, (SCTLR_ELx_ENIA | SCTLR_ELx_ENIB | \
                     SCTLR_ELx_ENDA | SCTLR_ELx_ENDB)
        orr     x0, x0, x1
alternative_else_nop_endif
        msr     sctlr_el2, x0
        isb

        /* Set the host vector */
        ldr     x0, =__kvm_hyp_host_vector
        msr     vbar_el2, x0

        ret
SYM_CODE_END(___kvm_hyp_init)
```
