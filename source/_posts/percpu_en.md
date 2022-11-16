---
title: "KVM ARM: EL2 per cpu variable(1): Definition & Usage"
date: 2022-11-16 20:28:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
  - per cpu variables
categories: technical
---

> Linux version: v6.0
>
> Architecture: ARMv8
>
> KVM flavor: NVHE

## Introduction

During the 5.10 release cycle, KVM ARM had many code improvements in preparation for the [google pkvm](https://www.youtube.com/watch?v\=wY-u6n75iXc) project. This includes the EL2 per cpu variables. Because there are quite a lot to discuss, I decided to split this topic into two posts.

1. Definition & Usage

2. per cpu variable initialization

Besides these there are still many related aspects such as barriers, preemption, and interrupts. We’re not going to talk about them here, since I don’t think I have a solid enough understanding.

### Per CPU Variables

Per CPU variables are simply variables that each CPU core sees a local copy. Upon access, the copy belonging to the running cpu is accessed, this eliminates inter-cpu data races.

There are already a lot of resources on Linux kernel’s per cpu variables online, this series of posts will focus on KVM ARM’s per cpu variable implementation catered for the EL2 environment.

## Definition

The API to define a EL2 per cpu variable is the same as the normal one. Here’s an example:

```c
// file: arch/arm64/kvm/hyp/nvhe/switch.c
DEFINE_PER_CPU(unsigned long, kvm_hyp_vector);
```

This macro expands into:

```c
__attribute__((section(".data..percpu" ""))) __typeof__(unsigned long) kvm_hyp_vector;
```

This result is also the same as the regular per cpu variable. So, does EL2 and EL1 use the same section for per cpu variables? The answer is no, code written for KVM EL2 are linked using a special linker script `arch/arm64/kvm/hyp/nvhe/hyp.lds`. This linker script renames the input sections, effectively seperating the kernel and the hypervisor.

> You’ll find that `hyp.lds` does not apper in the source code, this is becuase `hyp.lds` is generated at compile time from `hyp.lds.S` in the same directory.

This is the `hyp.lds` generated using `defconfig`:

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

Note that `kvm_hyp_vector` used in the previous example is moved from `.data..percpu` section to `.hyp.data..percpu`.

## Usage

Here’s a per cpu variable usage example:

```c
// file: arch/arm64/kvm/hyp/nvhe/switch.c
write_sysreg(this_cpu_ptr(&kvm_init_params)->hcr_el2, hcr_el2);
```

`this_cpu_ptr(&var)` returns the address of the per cpu variable `var`.

Lots of recursive macro is used in this line of code, it expands into (just skim through it):

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

We’re not interested in `write_sysreg()` and `hcr_el2`, remove them:

```c
// macro expansion of this_cpu_ptr(&kvm_init_params):
(
    {
        // this do-while(0) is used to statically check whether the input kvm_init_params
        // is a per cpu variable or not, skip for now
        do {
            const void *__vpp_verify = (typeof((&kvm_init_params) + 0))((void *)0);
            (void)__vpp_verify;
        } while (0);
        (
            {
                unsigned long __ptr;
                // basically means __ptr = (unsigned long)&kvm_init_params
                __asm__ ("" : "=r"(__ptr) : "0"((typeof(*(&kvm_init_params)) *)(&kvm_init_params)));
                // return __ptr + __hyp_my_cpu_offset()
                (typeof((typeof(*(&kvm_init_params)) *)(&kvm_init_params))) (__ptr + ((__hyp_my_cpu_offset())));
            }
        );
    }
)
```

You can see that the basic mechanism behind per cpu variables is taking a base pointer and then add a cpu-specific offset to get the local copy’s address.

`__hyp_my_cpu_offset()`’s implementation:

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

It reads `tpidr_el2` to get the per cpu offset. `tpidr_el2` is a system register which hardware doesn’t use, software can use it as it sees fit. KVM ARM uses it to store the per cpu offset for each cpu.

Lastly lets look at how per cpu variables are used in assembly:

```c
// file: arch/arm64/kvm/hyp/nvhe/host.S
get_host_ctxt  x0, x1
```

This is also a macro, it places the address of the per cpu variable `struct kvm_host_data kvm_host_data` into `x0`, `x1` is a temporary register. The macro expands into:

```c
// the first two instructions read the address of kvm_host_data (base)
// in a pc relative way
// 1. load kvm_host_data's address's higher bits ([:12]) into x1
adrp    x1, kvm_host_data
// 2. x0 = ((&kvm_host_data)[11:0] + x1), creating the complete address
add     x0, x1, #:lo12:kvm_host_data
// read tpidr_el2
mrs     x1, tpidr_el2
// kvm_host_data + tpidr_el2 = address of the per cpu kvm_host_data
add     x0, x0, x1
add     x0, x0, #0
```

It uses `tpidr_el2` as the offset, just like the C implementation.
