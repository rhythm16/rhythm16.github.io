---
title: PARange, SError, SEA小記錄
date: 2026-04-11 22:21:15
summary:
tags:
  - ARMv8
categories: technical
---

> 處理器架構：ARMv8

最近跟大佬主管聊天聊到標題中的幾個概念，相對以前非常表面的理解，聊完後整體認知又變得更具體了，記錄一下

## PARange

以一個抽象的模型思考的話，PARange的意思是CPU能夠向interconnect輸出最多幾個bits的位址。舉例來說如果最多是40個bits，程式硬要存取位址 ≥ (1 << 40) 的話，CPU內部自己就會發現無法滿足要求，而觸發一個 address size fault。

## SEA

SEA全稱是synchronous external abort，我學到這可能發生的例子是：在PARange中的位址中，不會所有位址都對應到硬體資源，如記憶體或IO設備。那如果CPU發出請求要存取沒有對應到硬體資源的位址時，就有可能會產生SEA，具體來說是CPU向interconnect發出請求，interconnect回應無法滿足，產生SEA。

## SError

SError舉的例子是：CPU若decode了很多記憶體位址的存取，在memory attributes允許的條件下，是有可能一次向interconnect發出許多請求的。如果這些請求中有的出現錯誤，有的沒有，那CPU會很尷尬。假設program order第一個請求失敗，第二個成功，那得把第二個請求的結果給undo才能報告一個synchronous的錯誤(SEA)，(接下來是我猜測)但因為這個同時發請求的動作跟branch prediction speculative execution不一樣，沒有辦法rollback，所以只好報告一個non synchronous error (SError interrupt)。要注意的是這類錯誤是產生SEA還是SError都是不一定的，要看具體硬體如何設計。

SError的缺點是它是asynchronous的，所以在觸發的當下，幾乎沒什麼有用資訊給軟體修正錯誤，例如fault pc，fault address都是無法得知的。

## Synchronous vs Precise

這個看Arm ARM就可以了(R_TNVSL)，Synchronous比Precise嚴格，Synchronous一定Precise，Precise不一定Synchornous，主要的差異在於Synchronous一定是因為執行指令，還有exception return address valid，例如page fault, hvc。Precise只要CPU狀態與memory狀態是consistent with CPU執行到exception當下的指令即可，所以包含了IRQ。


