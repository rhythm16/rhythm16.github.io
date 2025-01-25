---
title: "為什麼ARM Nested Virtualization的VNCR page 沒有CNT{P, V}\_TVAL_EL0"
date: 2025-01-25 14:54:15
summary:
tags:
  - Nested Virtualization
  - ARMv8
categories: technical
---

最近在研究ARM架構對於nested virtualization的支持和KVM ARM nested virtualization的實作，所以也就讀到了Generic timer的虛擬化。在觀察FEAT_NV2和Generic timer的相互作用時，我發現VNCR page中沒有CNT{P, V}\_TVAL_EL0，經過一些思考我發現這其實是FEAT_NV2與Generic Timer之間的架構衝突，而ARM Reference Manual中沒有寫明，所以還挺有趣的。

在說明之前，先來簡單說明一下FEAT_NV2與Generic Timer:

## FEAT_NV2

FEAT_NV2的全名叫做Enhanced Nested Virtualization，在沒有FEAT_NV2而只有FEAT_NV的情況下，執行guest EL2時基本上無論存取EL2或者EL1系統暫存器時都需要trap到EL2，這是因為guest EL2跑在真實CPU的EL1上，EL2暫存器當然不能讓guest EL2存取，而EL1暫存器技術上可以，但guest EL2存取EL1暫存器的目的是準備guest EL1/0的執行環境，自己的EL2環境則是不變的，而如果真的讓guest EL2存取EL1暫存器則會造成guest EL2跑的真EL1環境改變，造成錯誤。那在EL2和EL1系統暫存器的存取都要trap到真EL2的情況下，正確性是有了，但效能就變得很差，執行時guest EL2會頻繁地trap到真EL2。

為了改善效能，FEAT_NV2出現了。支持FEAT_NV2的硬體在執行guest EL2時，以下兩種系統暫存器的存取會被硬體轉換成記憶體存取:

1. EL1系統暫存器

2. 只影響EL1/0執行環境的EL2系統暫存器

每個被轉換的系統暫存器都有一個自己的記憶體位址，這個位址由offset + base address組成，offset被ARM架構各別指定，base address則是被VNCR_EL2所指定。這個優化的原理是源於guest EL2幫guest EL1/0準備執行環境時其實不需要所有存取都馬上產生效果，而只要在guest EL1/0開始跑之前有把效果達到就好，那硬體就先幫忙把想要改變的值都導到記憶體，真的eret開始跑guest EL1/0時真正的EL2再幫忙把guest EL1/0所需的環境依照記憶體內容搭建好就可以了，從而避免guest EL2每存取一次系統暫存器都要trap的效能問題。

## Generic Timer

每個ARM CPU上有多個Generic Timer，每個均由三個暫存器所構成，在EL1/0有2個timer可以用，physical timer和virtual timer，暫存器如下:

physical timer:

- CNTP_CTL_EL0

- CNTP_CVAL_EL0

- CNTP_TVAL_EL0

virtual timer:

- CNTV_CTL_EL0

- CNTV_CVAL_EL0

- CNTV_TVAL_EL0

CNT{P, V}\_CTL_EL0為控制暫存器，可以開關timer，mask interrupt等等，而CNT{P, V}\_{C, T}VAL_EL0則是負責設定timer何時觸發

CVAL: 寫入T則代表設定時間T，當system count >= T時timer中斷便會觸發

TVAL: 寫入t則代表設定t個count之後觸發，也就是system count >= (現在的system count) + t時timer中斷觸發

重要的是CVAL與TVAL為硬體提供設定同一個timer的兩種不同方式，而不是有兩個timer可以用，所以寫入CVAL時TVAL會跟著改變，反過來也是。

## 兩者之間的衝突

理解FEAT_NV2與Generic timer之後可以來談他們的衝突了，其實也不複雜，如果觀察FEAT_NV2中VNCR page中的各個暫存器，會發現有CNTV_CVAL_EL0，CNTV_CTL_EL0，  CNTP_CVAL_EL0，CNTP_CTL_EL0這四個暫存器，而不見CNT{P, V}\_TVAL_EL0的影子，為什麼呢?

其實是因為上面講到的，”CVAL與TVAL為硬體提供設定同一個timer的兩種不同方式”，用反的來解釋必較容易，如果VNCR page CVAL與TVAL都有，那一個guest hypervisor (guest EL2)執行時，寫了TVAL之後，再寫CVAL，也就是說VNCR page中CVAL和TVAL的位址都被改變，那現在換host hypervisor執行了，要怎麼分辨CVAL還是TVAL的位址的值才是最新的，要拿來更新CNT{P, V}\_{C, T}VAL_EL0呢? 又或者，guest hypervisor寫了TVAL，再讀CVAL，guest hypervisor就會發現CVAL的值竟然沒有跟的TVAL的寫入而被更新，不符合架構的要求。

為了解決這個問題，FEAT_NV2設計上只好被迫選擇CVAL跟TVAL其中一個來放進VNCR page中得到優化，而選擇CVAL的原因我猜是因為Linux使用CVAL比較多? 不太確定XD。至於TVAL的存取就必須trap到EL2來處理。
