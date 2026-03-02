---
title: "A bit about ARM VHE"
date: 2026-03-03 20:44:00
summary:
tags:
  - ARMv8
categories: technical
---

> Architecture: ARMv8

## Introduction

TBH, this post only exists because I don’t want to leave 2025 with just one single post, so I just kind of chose something to write about :) I haven’t posted anything until December because there was some changes that aren’t exactly small to my life during this time… Doesn’t matter really, I believe I should have more time for writing posts now. Let’s keep going.

So I went back looking at the Linux booting code again, and found the VHE related initialization was interesting, but going straight into that feels too quick, so I decided to first give an introduction to VHE, then come back to the booting code for a later article.

## What is VHE?

VHE is short for Virtualization Host Extension, an extension of the Arm architecture. Without VHE, EL2 differs from EL1:

- Unlike EL1, which has two translation table base registers TTBR0_EL1 & TTBR1_EL1, EL2 has only one, TTBR0_EL2

- The format of the translation table entries are different between EL1 and EL2

- There is only a physical timer for EL2, unlike EL1 with both virtual and physical timers

- etc..

With that, If we would like to run Linux, which is written to run in EL1, in EL2, in order to make use of EL2’s virtualization features directly, then that would not work, at least not without a huge modification effort.

### Benefits of running in EL2

The motivation for running Linux in EL2 is:

Linux KVM on Arm originally implements “split-mode” virtualization, where most of the kernel is in EL1, and a small, specially written EL2 part that runs in EL2 to provide virtualization functionalities. This forces EL1 hardware to be shared between the host and guest VMs, to run a guest VM:

1. host Linux EL1 jumps to EL2

2. EL2 saves the context and registers of host EL1

3. EL2 restores guest VM’s registers and context

4. EL2 jumps back to EL1 to run the guest VM

If we are able to run Linux in EL2, then host Linux wouldn’t need to share EL1 hardware with guest VMs, simplifying the process:

1. Linux in EL2 restores guest VM’s registers and context

2. EL2 jump back to EL1 to run the guest VM

For workloads that incur frequent VM exits, like interrupt heavy workloads, this can improve performance significantly.

### VHE

To get Linux running in EL2 and improve performance, Arm extends the architecture to allow kernels written for EL1 to run in EL2 with minimal code modifications, EL2 with VHE introduces changes include:

- Two translation table base registers, TTBR0_EL2 & TTBR1_EL2

- Same translation table entry format as EL1

- Both virtual timer and physical timer

- etc..

Basically extending EL2 to make it compatible with EL1, and one special thing:

- when EL2 accesses “EL1” registers, the hardware redirects the accesses to the EL2 version

What does that mean? For instance, when VHE EL2 reads TTBR0_EL1, the hardware actually returns the value in TTBR0_EL2. You might ask if that is the case, then how does EL2 get the value of the real TTBR0_EL1? Answer to that is it must read TTBR0_EL12 instead.

This behavior is for minimizing the amount of code required to change in the kernel (Linux) in order to run in EL2. Originally Linux uses EL1 registers to manage the system, they would all need to be modified to access EL2 instead. Even if that is done, what if someone wants to go back to running Linux in EL1? Do we duplicate Linux into EL1-Linux and EL2-Linux? Or we decide at configuration/run time?

With this behavior this problem is resolved, the hardware will automatically redirect the accesses for us depending on the EL.

Example:

```plain
/* VHE EL2 */
mrs x0, TTBR0_EL1  # x0 gets TTBR0_EL2    <---- automatic redirection by HW！
mrs x0, TTBR0_EL2  # x0 also gets TTBR0_EL2
mrs x0, TTBR0_EL12 # x0 gets TTBR0_EL1

/* non-VHE EL2 */
mrs x0, TTBR0_EL1  # x0 gets TTBR0_EL1
mrs x0, TTBR0_EL2  # x0 gets TTBR0_EL2

/* EL1 */
mrs x0, TTBR0_EL1  # x0 gets TTBR0_EL1  <----
```
