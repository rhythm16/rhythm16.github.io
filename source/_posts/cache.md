---
title: "為什麼KVM重新初始化vCPU時要清除所有stage 2地址轉換?"
date: 2024-10-30 12:54:15
summary:
tags:
  - Linux
  - KVM
  - ARMv8
categories: technical
---

> 對於這篇我原本是想要寫的順一點的，但寫到後來實在是太雜了也太多背景知識要補充，所以最後比較像是一份雜亂的筆記😅 有興趣的人如果有問題歡迎來信討論

此文中”guest”與”VM”與”guest VM”這三個詞是一樣的意思

Arm Architecture Reference Manual使用K.a版本

## 一個在讀code遇到的疑問

我在看KVM code時看到這一部分:

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
    // 此vCPU有跑過一次了，代表是重新初始化
	if (vcpu_has_run_once(vcpu)) {
        // 如果沒有FEAT_FWB功能
		if (!cpus_have_final_cap(ARM64_HAS_STAGE2_FWB))
            // 就把所有stage2 mapping給unmap
			stage2_unmap_vm(vcpu->kvm);
		else
			icache_inval_all_pou();
	}

[...]

}
```

現在都先假設沒有FEAT_FWB，此篇不考慮有FWB的情況

老實說，看不懂為什麼重新初始化就要把stage2給unmap掉。先來看看註解，第一部分就是針對此情況:

> Ensure a rebooted VM will fault in RAM pages

「在VM(和vCPU)重開機的情況下，確保vCPU重新page fault，再次建立stage 2 mapping」\
其實只是描述在做的事情，並沒有解釋原因

> and detect if the guest MMU is turned off and flush the caches as needed

「偵測guest是否將MMU關閉，並把該flush的給flush掉 (i.e. 把髒數據寫回)」

首先，這裡沒有程式在”偵測” guest有沒有把MMU關掉，所以這部分不知所云，後面說適時地把髒數據寫回，聽起來好像有必要，但是什麼時候需要呢? 也沒有解釋

## Mailing List怎麼說

遇到程式看不懂的時候就是挖mailing list的時候了，`git blame` 然後看一下commit通常就可以連到當初的email thread，這幾行程式經過了幾次的修改，討論比較完整的是[這串](https://lore.kernel.org/all/20200415072835.1164-1-yuzenghui@huawei.com/)，其中Alexandru Elisei說了為什麼當初Christoffer Dall要加上`stage2_unmap_vm`:

> I had a chat with Christoffer about stage2_unmap_vm, and as I understood it, the purpose was to make sure that any changes made by userspace were seen by the guest while the MMU is off. When a stage 2 fault happens, we do clean+inval on the dcache, or inval on the icache if it was an exec fault. This means that whatever the host userspace writes while the guest is shut down and is still in the cache, the guest will be able to read/execute.
>
> This can be relevant if the guest relocates the kernel and overwrites the original image location, and userspace copies the original kernel image back in before restarting the vm.

所以說unmap的用意是要讓guest重開機再次運行時能夠看見userspace (在跑guest之前) 所更改的記憶體內容，具體的例子是guest relocate kernel，並把原本kernel的位置給寫了別的東西，接著host userspace把VM暫停，把kernel image寫回原本的地方，接著重新開機。

## 分析

有了以上的補充之後先來整理一下這個過程發生了什麼事

1. guest正常運行，讀寫記憶體

2. 因interrupt, exception等因素切換回userspace

3. userspace暫停guest，並且打算將VM重新開機，所以重置並改寫了所有的guest memory，vCPU暫存器等等

4. userspace呼叫KVM API來重新init該vCPU

5. guest重新開始執行，重開機開始執行時，guest的MMU是關掉的

這個過程中，乍看之下如果KVM什麼都不做，可能會有2個問題:

1. guest在被暫停之前(第1步)因為使用與host不同的address space，所以coherency被打破，接著出現以下狀況

   1. 第一步時guest存取X虛擬位址，其實體位址為P，所以X(P)存在於cache中

   2. 第三步時host改寫guest記憶體，存取了Y虛擬位址，其實體位址也為P，所以Y(P)和X(P)存在於cache中

   3. 硬體決定把Y(P) clean到主記憶體，並且invalidate

   4. 硬體決定把X(P) clean到主記憶體，並且invalidate，結果最後主記憶體得到舊的數據，cache中的新數據也沒了

2. guest在第5步時存取到舊的值(guest被暫停前的值)，而不是第3步host改寫的值

   1. 第一步時guest存取記憶體，內容存在cache中，也被硬體flush到主記憶體

   2. 第三步時host改寫了guest的記憶體，但是內容只被存在cache中，沒有被硬體flush到主記憶體

   3. 第五步guest重新執行時，因為沒有打開MMU，導致CPU直接從主記憶體存取資料，就拿到了舊的資料

### 問題1

先來看看問題1有沒有可能出現，此狀況要分成2個case來討論:

第一，如果guest的memory access attributes與host相同，那比較簡單

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

在這個情況下cache就像是PIPT一樣，不會有不同VA造成的coherency問題，ordering方面也不會有問題，因為exception level切換屬於一種context synchronization event，保證了guest與host存取記憶體的順序。不過，因為instruction cache與data cache是沒有硬體coherent保證的，KVM在重新啟動vCPU之前必須要invalidate instruction cache。

第二，如果guest的memory access attributes與host不同，其實也不會有問題，即使有以下敘述:

```plain
B2.15.1.2 Non-shareable Normal memory
A location in Normal memory with the Non-shareable attribute does not require the hardware to make data accesses 
by different observers coherent, unless the memory is Non-cacheable. For a Non-shareable location, if other 
observers share the memory system, software must use cache maintenance instructions, if the presence of caches 
might lead to coherency issues when communicating between the observers. This cache maintenance requirement 
is in addition to the barrier operations that are required to ensure memory ordering.
```

host已知會使用inner cacheable, inner shareable，給guest的stage 2會也是inner shareable，所以guest最終使用的不會是non-shareable，所以也不會造成coherency問題，stage 1和stage 2 shareability合併的方式參照如下:

{% asset_img "image.png" "" %}

{% asset_img "image1.png" "" %}

那既然host與guest都是使用inner shareable，可以看到:

```plain
B2.15.1.1.1 Shareable, Inner Shareable, and Outer Shareable Normal memory
Each Inner Shareability domain contains a set of observers that are data coherent for each member of that set for 
data accesses with the Inner Shareable attribute made by any member of that set.
```

妥妥的，還是要flush instruction cache就是了

這邊沒有討論cacheability是因為by definition問題就說host與guest都使用cache存取了，所以一定是inner cacheable，畢竟一個作業系統是被預期跑在同一個inner shareability domain中的。

釐清一下，以下敘述所說需要cache maintenance是在有任意cacheability的情況，而現在guest & host都是inner cacheable所以沒問題

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

### 問題2

這個就是mailing list上主要在討論的狀況，那也確實會發生，我在上面貼的mailing list那一段中講到之所以把stage 2給unmap掉是因為如此一來guest在重新執行的時候就會不斷的stage 2 page fault，而KVM在處理page fault時會使用VA來clean + invalidate cache to PoC，而在做此cache maintenance時:

```plain
D7.5.9.5 Effects of instructions that operate by VA to the PoC
For Normal memory that is not Inner Non-cacheable, Outer Non-cacheable, cache maintenance instructions that 
operate by VA to the PoC must affect the caches of other PEs in the shareability domain described by the shareability 
attributes of the VA supplied with the instruction.
```

也就是說所有CPU只要使用inner/non shareable都會看到clean + invalidate的效果~~(因為沒有non shareable的情況)~~

那一定要unmap嗎? 不能只clean + invalidate嗎? 我猜想只是這樣可以達到lazy cache maintenance的效果，兩個做法要付出的work如下：

unmap:

- 爬整個guest s2並且清空stage 2 page tables

- guest每存取一個page就page fault並且clean + invalidate

clean + invalidate

- 爬整個guest s2並且為整個VA做clean + invalidate

究竟怎樣是比較划算也是個有趣的問題，不過我猜，當時選擇unmap的其中一個原因是實作比較簡單XD

## 背景知識

- Linux假設其控制的所有CPU均處於同一個inner shareability domain，ARM架構也做此假設

- Linux使用的normal memory attributes為Inner Shareable, Inner Write-Back Cacheable Non-transient Outer Write-Back Cacheable Non-transient

- device memory不會被緩存到cache中

- 在有stage 2 translation的情形之下，stage 1 translation的attributes (guest控制)會和KVM控制的stage 2 translation attributes合併

## 相關的ARM Architecture Reference Manual章節

- B2.12 Caches and memory hierarchy

- B2.15 Memory types and attributes

- B2.16 Mismatched memory attributes

- D7.5 Cache support

- D8.2.12 The effects of disabling an address translation stage

- D8.6 Memory region attributes

- D8.17 Caches

