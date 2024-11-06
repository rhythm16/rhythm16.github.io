---
title: "Why does the stage 2 page tables need to be  cleared after a reinitialization of a vCPU?"
date: 2024-11-06 22:01:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
categories: technical
---

> Originally I wanted to do better job at presenting the contents of this article, but there is just too much information and required knowledge. Now this just looks like some notes that I took down 😅, sorry about that. You are welcome to email if you have any questions.

“guest”, “VM”, and “guest VM” are used interchangably in this article.

Arm Architecture Reference Manual K.a is assumed.

## The Piece of Code that I Couldn’t Understand

I saw this during my code-reading:

```c
static int kvm_arch_vcpu_ioctl_vcpu_init(struct kvm_vcpu *vcpu,
					 struct kvm_vcpu_init *init)
{

[...]
	/*
	 * Ensure a rebooted VM will fault in RAM pages and detect if the
	 * guest MMU is turned off and flush the caches as needed.
	 *
	 * S2FWB enforces all memory accesses to RAM being cacheable,
	 * ensuring that the data side is always coherent. We still
	 * need to invalidate the I-cache though, as FWB does *not*
	 * imply CTR_EL0.DIC.
	 */
    // reinit this vCPU
	if (vcpu_has_run_once(vcpu)) {
        // if no FEAT_FWB
		if (!cpus_have_final_cap(ARM64_HAS_STAGE2_FWB))
            // unmap all stage 2 mappings
			stage2_unmap_vm(vcpu->kvm);
		else
			icache_inval_all_pou();
	}

[...]

}
```

I’ll assume there’s not FEAT_FWB in this article.

Now I don’t really understand why reinitializing a vCPU requires KVM to unmap all stage2 mappings. Let’s check the comment first.

> Ensure a rebooted VM will fault in RAM pages

This is just describing what the code is doing, not the reason, so not helpful.

> and detect if the guest MMU is turned off and flush the caches as needed.

Now this piece of code is not “detecting” anything (aside from FWB), let alone detecting if the guest’s MMU is off, so not sure what this is about. Next the comments says flush the caches as needed, sounds like a reasonable thing to do, but when is this needed is not explained.

## What does the Mailing List Say?

It’s time to turn to the mailing list after not understanding some code. `git blame` and checking the commits often leads us to the email thread discussing the code. This piece of code has gone through a few modifications, [this thread](https://lore.kernel.org/all/20200415072835.1164-1-yuzenghui@huawei.com/) has more helpful discussions, and Alexandru Elisei explained why the original author Christoffer Dall adds `stage2_unmap_vm` :

> I had a chat with Christoffer about stage2_unmap_vm, and as I understood it, the purpose was to make sure that any changes made by userspace were seen by the guest while the MMU is off. When a stage 2 fault happens, we do clean+inval on the dcache, or inval on the icache if it was an exec fault. This means that whatever the host userspace writes while the guest is shut down and is still in the cache, the guest will be able to read/execute.
>
> This can be relevant if the guest relocates the kernel and overwrites the original image location, and userspace copies the original kernel image back in before restarting the vm.

## Analysis

The sequence of events that is described in the mailing list is:

1. guest runs and accesses memory

2. some interrupt or exception causes the CPU to switch back to host userspace

3. userspace pauses the guest, prepares to restart the VM, so it rewrites all guest memory, vCPU registers, etc.

4. userspace calls KVM API to reinitialize vCPU

5. guest restarts, and the initial state of the MMU is off

I think of two issues that could happen during this process.

1. coherency gets broken in the cache, because the guest and the host uses separate address spaces, for example

   1. guest accesses address virtual address X, where the corresponding physical address is P, and X(P) is in the cache

   2. host then rewrites guest memory, accesses virtual address Y, with the same physical address P, so Y(P) and X(P) are both in the cache

   3. hardware decides to write back (clean) Y(P), and invalidate

   4. hardware later decides to write back (clean) X(P), and invalidate, this causes the new data (Y) to be overwritten by the old data (X), so we lose the new updates by the host

2. after restart, at step 5, guest accesses the old data (the data before restart), instead of the new data written by the host in step 3 above

   1. before guest restarts, its contents are in both the cache and memory

   2. at step 3 above, host rewrites guest memory, but the updated data are only in the cache, not flushed to main memory yet

   3. guest restarts, and since MMU is off, CPU directly reads data from memory instead of the cache, hence accessing old data

### Problem 1

Let’s first discuss if issue 1 can happen or not, there are two cases:

First, if guest’s memory access attributes are identical to the host, then it’s simple

```plain
D8.17.1 Data and unified caches

R_JHVQL
For data and unified caches, if all data accesses to an address do not use mismatched memory attributes, then the 
use of address translation is transparent to any data access to the address.

I_FHPPX
The properties of data and unified caches are consistent with implementing the caches as physically-indexed, 
physically-tagged caches.

D8.17.2 Instruction caches

R_YXNGL & R_LYZYY
If all instruction fetches to an address do not use mismatched memory attributes, then the use of address 
translation is transparent to any instruction fetch to the address.
```

The cache acts like a PIPT cache in this case, so no coherency problem from different VAs mapping to the same PA. Ordering won’t be an issue either, because switching exception levels is a context synchronization event. However, because there aren’t guarantees of coherency between instruction cache and data cache, KVM must invalidate the instruction cache before restarting the vCPU.

Second, even if the memory access attributes of the guest is different from the host, there wouldn’t be a problem as well, even if we can see:

```plain
B2.15.1.2 Non-shareable Normal memory
A location in Normal memory with the Non-shareable attribute does not require the hardware to make data accesses 
by different observers coherent, unless the memory is Non-cacheable. For a Non-shareable location, if other 
observers share the memory system, software must use cache maintenance instructions, if the presence of caches 
might lead to coherency issues when communicating between the observers. This cache maintenance requirement 
is in addition to the barrier operations that are required to ensure memory ordering.
```

It is known that the host uses inner cacheable, inner shareable attributes, and the stage 2 attributes set for the guest is also inner shareable, therefore the guest would not be using a non-shareable attribute, so there won’t be a coherency problem. The rules for combining stage 1 and stage 2 attributes are listed below.

{% asset_img "image.png" "" %}

{% asset_img "image1.png" "" %}

Now that both the host and the guest uses the inner shareable attribute, we can see:

```plain
B2.15.1.1.1 Shareable, Inner Shareable, and Outer Shareable Normal memory
Each Inner Shareability domain contains a set of observers that are data coherent for each member of that set for 
data accesses with the Inner Shareable attribute made by any member of that set.
```

Very safe. Still got to flush the instruction cache though.

No cacheability concerns here because the host and the guest uses the cache by definition, so the attribute must be inner cacheable. All caches in an operating system are expected to be in the same inner shareability domain.

To clarify, the need for cache maintenance operations in the statement below applies to the cases where there may be arbitrary cacheability, which is not our case. We know both the guest and the host uses inner cacheable attribute.

```plain
D8.17.3 Cache maintenance requirements due to changing memory region attributes
The behaviors caused by mismatched memory attributes mean that if any of the following changes are made to the 
Inner Cacheability or Outer Cacheability attributes in translation table entries, then software is required to ensure 
that any cached copies of affected locations are removed from the caches, typically by cleaning and invalidating the 
locations from the cache levels that might hold copies of the locations affected by the attribute change:
• A change from Write-Back to Write-Through.
• A change from Write-Back to Non-cacheable.
• A change from Write-Through to Non-cacheable.
• A change from Write-Through to Write-Back.
```

### Problem 2

This is the primary situation that was discussed in the mailing list thread, and it does have the possiblity of happening. The reason why unmapping stage 2 helps solves this issue is because after restarting the guest will keep doing stage 2 page faults, and KVM cleans + invalidates cache to PoC for VA when handling the stage 2 page faults, when doing these cache maintenance operations:

```plain
D7.5.9.5 Effects of instructions that operate by VA to the PoC
For Normal memory that is not Inner Non-cacheable, Outer Non-cacheable, cache maintenance instructions that 
operate by VA to the PoC must affect the caches of other PEs in the shareability domain described by the shareability 
attributes of the VA supplied with the instruction.
```

All CPUs using the inner/non shareable attributes will see the results of the clean + invalidate.

But do we ***have to*** unmap? Can’t we just clean + invalidate all VA ranges? It might be because unmapping delays the cache maintenance operations. A small comparison between these two methods:

unmap:

- walk all guest stage 2 page tables and clear everything

- guest page faults when accessing each page, and KVM does clean and invalidate then

clean + invalidate

- walk all guest stage 2 page tables and clean + invalidate everything

It’s an interesting question whether which one is better, but my guess is that the original author chose to unmap everything because it’s just easier to implement XD

## Background

- Linux assumes all CPUs under its control is in the same inner shareability domain, which is also assumed by the ARM architecture

- The normal memory attributes that Linux uses are Inner Shareable, Inner Write-Back Cacheable Non-transient Outer Write-Back Cacheable Non-transient

- device memory is not cached

- when stage 2 translation is activated, stage 1 memory access attirbutes (controlled by the guest) is combined with KVM-controlled stage 2 memory access attritubes

## Related ARM Architecture Reference Manual Sections

- B2.12 Caches and memory hierarchy

- B2.15 Memory types and attributes

- B2.16 Mismatched memory attributes

- D7.5 Cache support

- D8.2.12 The effects of disabling an address translation stage

- D8.6 Memory region attributes

- D8.17 Caches

