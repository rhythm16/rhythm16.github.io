---
title: "KVM ARM: EL2 per cpu variable (2): Initialization"
date: 2022-11-30 23:14:15
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

The last post explained how EL2 per cpu variables are defined and used. To review: EL2 per cpu variables are accessed by first acquiring the base address in the `.hyp.data..percpu` section, then add the per cpu offset stored in `tpidr_el2` to get the final address. There are two questions to be answered:

1. How are the EL2 per cpu area allocated?

2. How are the per cpu offsets calculated and installed into `tpidr_el2`?

This post will answer these questions.

## Memory Allocation

EL2 per cpu area’s memory is allocated in the KVM ARM initialization function `init_hyp_mode` :

```c
/*
 * Allocate and initialize pages for Hypervisor-mode percpu regions.
 */
for_each_possible_cpu(cpu) {
        struct page *page;
        void *page_addr;
        // allocate memory with alloc_pages
        // nvhe_per_cpu_order() calculates how large the per cpu area is in pages,
        // then take log2 e.g. 8 pages = 2^3 pages，returns 3.
        page = alloc_pages(GFP_KERNEL, nvhe_percpu_order());
        if (!page) {
                err = -ENOMEM;
                goto out_err;
        }
        // turn the struct page returned into linear address
        page_addr = page_address(page);
        // copy the base values in .hyp.data..percpu into the allocated area
        // in case some per cpu variables are initialized
        // CHOOSE_NVHE_SYM is used to change the symbol's name into an nvhe symbol name
        // CHOOSE_NVHE_SYM(__per_cpu_start) can be thought as the start of .hyp.data..percpu
        memcpy(page_addr, CHOOSE_NVHE_SYM(__per_cpu_start), nvhe_percpu_size());
        // store the allcated linear address into an EL1 array
        kvm_arm_hyp_percpu_base[cpu] = (unsigned long)page_addr;
}
```

Map the allocated area to EL2 after allocation (same, in `init_hyp_mode`):

```c
for_each_possible_cpu(cpu) {
        // get the EL1 linear address that should be mapped to EL2 for this CPU
        char *percpu_begin = (char *)kvm_arm_hyp_percpu_base[cpu];
        char *percpu_end = percpu_begin + nvhe_percpu_size();

        /* Map Hyp percpu pages */
        // self explanatory
        err = create_hyp_mappings(percpu_begin, percpu_end, PAGE_HYP);
        if (err) {
                kvm_err("Cannot map hyp percpu region\n");
                goto out_err;
        }

        /* Prepare the CPU initialization parameters */
        // see the next section for explanation
        cpu_prepare_hyp_mode(cpu);
}
```

## Setting up tpidr_el2

We have discussed how the EL2 per cpu areas are allocated and how they are mapped to EL2. Next up is the offset calculation and `tpidr_el2` installation.

### Calculating per cpu offset

`cpu_prepare_hyp_mode(cpu)` is responsible for filling up a `struct kvm_nvhe_init_params` . This structure stores the initial values of EL2 system registers, including `tpidr_el2` . The relavent parts are listed below:

```c
static void cpu_prepare_hyp_mode(int cpu)
{
        // use per_cpu_ptr_nvhe_sym to get the linear address of the symbol
        // for the current cpu
        // in this case, the symbol kvm_init_params
        struct kvm_nvhe_init_params *params = per_cpu_ptr_nvhe_sym(kvm_init_params, cpu);
        
        [...]

        /*
         * Calculate the raw per-cpu offset without a translation from the
         * kernel's mapping to the linear mapping, and store it in tpidr_el2
         * so that we can use adr_l to access per-cpu variables in EL2.
         * Also drop the KASAN tag which gets in the way...
         */
        // here we calculate the per cpu offset
        // formula: A - B where:
        // A: the start of the per cpu area allocated using alloc_pages (linear address)
        // B: start of the base per cpu area (linear address)
        params->tpidr_el2 = (unsigned long)kasan_reset_tag(per_cpu_ptr_nvhe_sym(__per_cpu_start, cpu)) -
                            (unsigned long)kvm_ksym_ref(CHOOSE_NVHE_SYM(__per_cpu_start));

        [...]
}
```

### Installing `tpidr_el2`

After saving the per cpu offset in `params→tpidr_el2` , the next step is to install the value in `tpidr_el2` , let’s first check out the call stack of the initialization of KVM ARM:

```c
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

`hyp_install_host_vector` runs in EL1, it calls a hypercall to enter EL2, passing in a physical pointer to a `struct kvm_nvhe_init_param` to initialize EL2. `___kvm_hyp_init` does the actual initialization.

```c
static void hyp_install_host_vector(void)
{
        struct kvm_nvhe_init_params *params;
        struct arm_smccc_res res;

        [...]
        // get the EL1 linear address of the current cpu's `kvm_init_params`
        params = this_cpu_ptr_nvhe_sym(kvm_init_params);
        // this does an `hvc`, passing in
        // 1. function ID for `__kvm_hyp_init`
        // 2. physical address of `kvm_init_params` of the current cpu
        // 3. address of the local variable `res`, used to output exit code
        arm_smccc_1_1_hvc(KVM_HOST_SMCCC_FUNC(__kvm_hyp_init), virt_to_phys(params), &res);
        WARN_ON(res.a0 != SMCCC_RET_SUCCESS);
}
```

We’ll skip how that line `arm_smccc…` actually calls `hvc` , in short, EL2 can check `x0` for `__kvm_hyp_init` ‘s ID to confirm the reason why EL1 called `hvc` , and receive the pointer to `kvm_init_params` in `x1` . Then the execution goes to:

```c
// arch/arm64/kvm/hyp/nvhe/hyp-init.S
        /*
         * Only uses x0..x3 so as to not clobber callee-saved SMCCC registers.
         *
         * x0: SMCCC function ID
         * x1: struct kvm_nvhe_init_params PA
         */
__do_hyp_init:
        /* Check for a stub HVC call */ // not in this case
        cmp     x0, #HVC_STUB_HCALL_NR
        b.lo    __kvm_handle_stub_hvc
        // check for __kvm_hyp_init (yes)
        mov     x3, #KVM_HOST_SMCCC_FUNC(__kvm_hyp_init)
        cmp     x0, x3
        // jump to 1:
        b.eq    1f
    
        mov     x0, #SMCCC_RET_NOT_SUPPORTED
        eret
        // move param's physical address to x0
1:      mov     x0, x1
        // move return address to x3, in case ___kvm_hyp_init overwrites it
        // (___kvm_hyp_init) does not change x3
        mov     x3, lr
        // jump to the processing function
        bl      ___kvm_hyp_init   // Clobbers x0..x2
        // recover lr
        mov     lr, x3
    
        /* Hello, World! */
        // put value indicating success into x0
        mov     x0, #SMCCC_RET_SUCCESS
        // return back to EL1
        eret
```

Lastly let’s check out `___kvm_hyp_init` :

Actually `tpidr_el2` is set up in the very first two instructions, I’ll let you see for your self how it does it, you can also see how other EL2’s system registers are initialized in this function.

> Note that EL2 MMU is not enabled yet when this function is executed, therefore all memory accesses use physical addresses here.

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
