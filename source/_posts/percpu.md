---
title: "KVM ARM: EL2 per cpu變數(1): 定義及存取"
date: 2022-11-10 22:20:15
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

在Linux kernel 5.10週期，KVM ARM開發者們為了為[google pkvm](https://www.youtube.com/watch?v\=wY-u6n75iXc)做準備，在code base許多地方做了翻修，這篇所介紹的EL2 per cpu變數也是其中一項。而因為打算討論的內容有點多，所以預期分成兩個部份：

1. 定義及存取

2. per cpu變數初始化

除了這些之外還有很多相關的內容，例如barriers, preemption, interrupts等和per cpu變數相關的議題，但我目前並不是很熟悉所以無法多做討論，有興趣的讀者可以自行上網查詢資料。

### Per CPU 變數

Per CPU變數簡而言之就是每一個CPU核心各有一份的變數，程式存取per cpu變數時會依照當下的CPU核心去存取對應的變數，不會互相干擾。

Linux kernel的per cpu變數實作機制網路上已經有許多很好的資料可以參考，這篇主要會聚焦在KVM ARM為了EL2環境所量身訂做的per cpu變數實作機制。

## 實作機制

### 定義

EL2的per cpu變數API其實是和一般的一樣，先看如何定義per cpu變數：

```c
// file: arch/arm64/kvm/hyp/nvhe/switch.c
DEFINE_PER_CPU(unsigned long, kvm_hyp_vector);
```

macro展開之後會變成：

```c
__attribute__((section(".data..percpu" ""))) __typeof__(unsigned long) kvm_hyp_vector;
```

這個結果也和一般的per cpu變數一樣，難道EL2使用的per cpu變數和EL1 kernel使用的都放在同一個section裡面嗎？ 答案是否定的，KVM EL2的檔案會由一個特別的linker script來鏈結，也就是`arch/arm64/kvm/hyp/nvhe/hyp.lds`，這個linker script會藉由把EL2的sections都改名，來把kernel和hypervisor的sections都分開，以下是用`defconfig` 編譯產生的`hyp.lds`

> `hyp.lds`不會在源碼中出現，因為`hyp.lds`是從同個directory的`hyp.lds.S` 在編譯時預處理所產生的

觀察一下`hyp.lds`:
```c
SECTIONS {
    .hyp.idmap.text : {
        __hyp_section_.hyp.idmap.text = .;
        *(.idmap.text .idmap.text.*)
    }
    .hyp.text : {
        __hyp_section_.hyp.text = .;
        *(.text .text.*)
    }
    .hyp.data..ro_after_init : {
        __hyp_section_.hyp.data..ro_after_init = .;
        *(.data..ro_after_init .data..ro_after_init.*)
    }
    .hyp.rodata : {
        __hyp_section_.hyp.rodata = .;
        *(.rodata .rodata.*)
    }
    . = ALIGN((1 << 12));
    .hyp.data..percpu : {
        __hyp_section_.hyp.data..percpu = .;
        __per_cpu_start = .;
        *(.data..percpu..first)
        . = ALIGN((1 << 12));
        *(.data..percpu..page_aligned)
        . = ALIGN((1 << (6)));
        *(.data..percpu..read_mostly)
        . = ALIGN((1 << (6)));
        *(.data..percpu)
        *(.data..percpu..shared_aligned)
        __per_cpu_end = .;
    }
    .hyp.bss : {
        __hyp_section_.hyp.bss = .;
        *(.bss .bss.*)
    }
}
```

可以看到上面例子的`kvm_hyp_vector` 原本在`.data..percpu` section裡，用`hyp.lds` 鏈結之後就會變到`.hyp.data..percpu` 了

### 存取

接著看一個存取的例子：

```c
// file: arch/arm64/kvm/hyp/nvhe/switch.c
write_sysreg(this_cpu_ptr(&kvm_init_params)->hcr_el2, hcr_el2);
```

`this_cpu_ptr(&var)`會回傳屬於當前cpu那份叫做var的per cpu變數的位址

這行程式也使用了非常多macro，展開之後如下（不用仔細看）：

```c
do {
    u64 __val =
        (u64)(
            (
                {
                    do {
                        const void *__vpp_verify = (typeof((&kvm_init_params) + 0))((void *)0);
                        (void)__vpp_verify;
                    } while (0);
                    (
                        {
                            unsigned long __ptr;          
                            __asm__ ("" : "=r"(__ptr) : "0"((typeof(*(&kvm_init_params)) *)(&kvm_init_params)));
                            (typeof((typeof(*(&kvm_init_params)) *)(&kvm_init_params))) (__ptr + ((__hyp_my_cpu_offset())));
                        }
                    );
                }
            )->hcr_el2
        );
        asm volatile("msr " "hcr_el2" ", %x0" : : "rZ" (__val));
} while (0);
```

其中`write_sysreg()`和`hcr_el2`的部份是目前不關心的，刪減之後剩下：

```c
// macro expansion of this_cpu_ptr(&kvm_init_params):
(
    {
        // 這個 do-while(0) 只是靜態分析傳入的&kvm_init_params是不是一個per cpu變數，現在可以略過
        do {
            const void *__vpp_verify = (typeof((&kvm_init_params) + 0))((void *)0);
            (void)__vpp_verify;
        } while (0);
        (
            {
                unsigned long __ptr;
                // 基本上就是 __ptr = (unsigned long)&kvm_init_params 的意思
                __asm__ ("" : "=r"(__ptr) : "0"((typeof(*(&kvm_init_params)) *)(&kvm_init_params)));
                // 回傳 __ptr + __hyp_my_cpu_offset()
                (typeof((typeof(*(&kvm_init_params)) *)(&kvm_init_params))) (__ptr + ((__hyp_my_cpu_offset())));
            }
        );
    }
)
```

從以上分析可以看出per cpu變數的操作基本上就是去拿一個base pointer，然後加上一個隨cpu變化的offset，去拿到屬於此cpu的位址。

`__hyp_my_cpu_offset()`的實作如下：

```c
static inline unsigned long __hyp_my_cpu_offset(void)
{
        /* 
         * Non-VHE hyp code runs with preemption disabled. No need to hazard
         * the register access against barrier() as in __kern_my_cpu_offset.
         */
        return read_sysreg(tpidr_el2);
}
```

而去取得各個per cpu的offset就是去讀`tpidr_el2`這個系統暫存器。`tpidr_el2`就是一個專門給軟體使用的暫存器（當然每個CPU核心各有一個），KVM ARM這裡就拿來放per cpu變數的offset。

最後來看一個在組合語言中使用per cpu變數的例子：

```c
// file: arch/arm64/kvm/hyp/nvhe/host.S
get_host_ctxt  x0, x1
```

這也是一個macro，效果是把一個在其他地方定義的per cpu `struct kvm_host_data kvm_host_data`的位址放到`x0`，`x1`則是暫時使用的暫存器，macro展開如下：

```c
// 用兩個指令把 kvm_host_data 的位址用pc relative的方式讀到x0
// 1. 把 kvm_host_data 的位址 bit 12以上的高位讀到x1
adrp    x1, kvm_host_data
// 2. 把 (kvm_host_data 的位址低12位 + x1) 到 x0，形成完整的位址
add     x0, x1, #:lo12:kvm_host_data
// 讀 tpidr_el2
mrs     x1, tpidr_el2
// kvm_host_data + tpidr_el2 = 這個cpu的kvm_host_data per cpu變量的位址
add     x0, x0, x1
add     x0, x0, #0
```

作法和C code一樣是使用`tpidr_el2`來當作offset，合情合理。
