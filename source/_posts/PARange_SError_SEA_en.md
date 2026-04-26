---
title: Notes on PARange, SError, SEA
date: 2026-04-27 21:01:25
summary:
tags:
  - ARMv8
categories: technical
---

> Architecture：ARMv8

I discussed the topics in the title with my boss recently, and learned quite a lot. Previously I only understood these ideas in an abstract way.

## PARange

Abstractly, PARange stands for the number of bits the CPU is able to send to the interconnect as addresses. For example if the maximum is 40 bits, and software accesses an address ≥ (1 << 40) the CPU will detect it can’t satisfy the access and trigger an address size fault.

## SEA

SEA stands for synchronous external abort, what I learned was: In the entire PARange, not every part of it will map to hardware, like RAM or IO devices. Then if the CPU tries to access an address without hardware, an SEA could happen. Concretely, the interconnect will see this when receiving the request, and replies the request is unsatisfiable, which incurs an SEA.

## SError

The example for SError is: If the CPU decodes a bunch of memory accesses, and if memory attributes allows it, the CPU can send multiple requests to the interconnect in parallel. And if some of these requests succeed and some fail, the situation becomes awkward for the CPU. Say the first request fails, but the second succeeds, the second result must be undone in order to report a synchronous error (SEA), (the following is just my guess) but since this parallel sending of requests is different from branch prediction speculative execution, it can’t be rollbacked, so the only way to report the error is via a non synchronous error (SError interrupt). Note that whether an SEA or SError is triggered for this kind of situation is hardware implementation defined.

The downside of SError is that it is asynchronous, therefore there is little information left to software to fix the problem at the fault. For instance the fault pc and the fault address are both not available.

## Synchronous vs Precise

This is clearly stated in the Arm ARM (R_TNVSL). Synchronous is stricter than precise, meaning synchornous is precise, precise may not be synchronous. The main difference is that synchronous must be caused by an attempt to execute an instruction, and the exception return address is valid, like page fault or hvc. Precise only requires the CPU and memory state is consistent with the CPU’s execution up to the point of the exception, therefore it includes IRQ.


