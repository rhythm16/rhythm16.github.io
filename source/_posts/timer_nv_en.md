---
title: "Why aren’t CNT{P, V}\_TVAL_EL0 in the VNCR page?"
date: 2025-01-30 15:48:15
summary:
tags:
  - Nested Virtualization
  - ARMv8
categories: technical
---

Recently I have been looking at ARM architecture’s support for nested virtualization and its implementation in KVM ARM, so I also read about the virtualization of the generic timer. While reading about the nested virtualization of the generic timer, I found that, confusingly,  CNT{P, V}\_TVAL_EL0 is missing in the VNCR page. After thinking about it for a while I realized there’s an architectural conflict between FEAT_NV2 and the generic timer. Interestingly, the reason isn’t elaborated in the ARM Reference Manual.

Before explaining, I’ll give a brief overview of FEAT_NV2 and the Generic Timer.

## FEAT_NV2

FEAT_NV2 is ARM’s “Enhanced Nested Virtualization” support, and is based on FEAT_NV, which provides the basic features for nested virtualization. In a system without FEAT_NV2 and only FEAT_NV, the guest EL2 will trap to EL2 whenever it accesses EL1 or EL2 system registers. This is because the guest EL2 runs in real EL1, so the real EL2 registers cannot be directly accessed by the guest EL2 for obvious reasons. On the other hand, the real EL1 registers provide the environment in which the guest EL2 runs, not the guest EL1 environment that the guest EL2 expects to modify when it accesses EL1 registers, therefore the hypervisor must also trap guest EL2’s EL1 register accesses. Given the trapping of the EL1 and EL2 register accesses, the hypervisor can provide correctness of the virtualization, with the drawback of it being very slow.

To improve performance, FEAT_NV2 is introduced. FEAT_NV2 hardware turns system register accesses into memory accesses by guest EL2 when it accesses these two types of registers:

1. EL1 registers

2. EL2 registers that only affects the execution environment of EL1/0

Each register that is included in above has its own memory address, composed of a base address + an offset. The offset is given by the ARM architecture, and the base address is the value stored in VNCR_EL2. The idea is that when guest EL2 accesses guest EL1/0 registers, it does not need to take effect immediately, it only has to take effect before the guest EL1/0 starts to run. The hardware can cache the guest EL2 accesses in memory first, then flush it to the real hardware registers once guest EL1/0 is about to run. This reduces an huge amount of trapping.

## Generic Timer

Each ARM CPU has many Generic Timers in it, and each timer is controlled by 3 timer registers. There are 2 timers, physical and virtual timer available in EL1/0, their registers are:

physical timer:

- CNTP_CTL_EL0

- CNTP_CVAL_EL0

- CNTP_TVAL_EL0

virtual timer:

- CNTV_CTL_EL0

- CNTV_CVAL_EL0

- CNTV_TVAL_EL0

CNT{P, V}\_CTL_EL0 are control registers, used to enable/disable the timer, masking interrupts, etc., and CNT{P, V}\_{C, T}VAL_EL0 are used to set when the timer should go off.

CVAL: writing T into it means trigger the timer when system count >= T

TVAL: writing t into it means trigger the timer when system count >= current system count + t

Note that CVAL and TVAL are two different ways to set up the same timer, they do not represent two distinct timers, so when either CVAL or TVAL is written, the other one will have its content changed.

## The Conflict Between the Two

Now after discussing about FEAT_NV2 and the Generic Timer, let’s talk about the conflict between them. If you observe the registers that are listed in the VNCR page, you’ll find there are CNTV_CVAL_EL0, CNTV_CTL_EL0, CNTP_CVAL_EL0, CNTP_CTL_EL0, but not CNT{P, V}\_TVAL_EL0. Why aren’t all the timer registers in there?

This is because “CVAL and TVAL are two different ways to set up the same timer“ talked about above. If both CVAL and TVAL are in the VNCR page, then when a guest EL2 runs, after it writes to TVAL and CVAL, two places in the VNCR page are modified, how can the hypervisor know which value is the newest, to be updated to the hardware CNT{P, V}\_{C, T}VAL_EL0? They both represent the same timer after all. Moreover, if the guest EL2 writes to TVAL, then reads CVAL, it will find that CVAL isn’t updated after it wrote to TVAL, violating the architecture.

To address this problem, FEAT_NV2 is forced to compromise and choose only one of CVAL or TVAL to place in the VNCR page, I’m not really sure why CVAL is selected, perhaps Linux uses CVAL more frequently? IDK :) TVAL access must be trapped, in contrast.


