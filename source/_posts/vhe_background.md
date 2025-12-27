---
title: "ARM VHE的一些相關知識"
date: 2025-12-28 17:13:00
summary:
tags:
  - ARMv8
categories: technical
---

> 處理器架構：ARMv8

## 前言

老實說，這篇是因為我不想要今年只有寫一月那一篇文章所以才趕快找個東西寫的 :)，直到12月才寫第二篇是因為我的生活在這中間發生了一些不小的變化… 不重要，反正之後應該很有機會可以更常寫東西嘍～ 寫是一定要寫的，莫忘初心

最近又回去看了一下開機的流程，覺得Linux在VHE相關初始化判斷挺有趣的，所以想來介紹一下，不過寫一寫發現不先給點背景知識好像會跳太快，所以這篇先簡單介紹一下VHE，之後再來講講開機相關的部份。

## VHE是啥

VHE的全稱是Virtualization Host Extension，是Arm架構的一個延展，在沒有VHE的狀況下，EL2是一個和EL1不相同的執行環境：

- EL2不像EL1有兩個translation table base register TTBR0_EL1 & TTBR1_EL1，EL2只有一個TTBR0_EL2

- EL2 translation table entry的格式和bit field定義和EL1 translation table entry不一樣

- EL2只有一個physical timer，不像EL1有virtual timer和physical timer

- etc..

也就是說，如果我們想要把Linux這種為EL1編寫的內核跑在EL2上，讓它能直接使用EL2提供的虛擬化功能，作為一個虛擬機器監測器的話，那是不行的，至少在不劇烈改動程式的情況下沒辦法。

### Linux跑在EL2的好處

簡單說明一下為什麼有把Linux跑在EL2的動機：

原先Linux KVM在Arm上提供虛擬化功能的方式是把絕大部分的Linux跑在EL1，然後有一小部份，專門為EL2設計的程式碼跑在EL2為EL1提供虛擬化服務。在這個設定下EL1的宿主Linux (host Linux) 就必須和虛擬機器分時共享EL1硬體，要執行虛擬機器的時候必須：

1. host Linux EL1跳到EL2

2. EL2保存host Linux EL1的暫存器，執行狀態等等

3. EL2恢復虛擬機器的暫存器，執行狀態等等

4. EL2跳到虛擬機器的EL1開始執行

而如果可以將Linux直接在EL2執行的話，host Linux就不需要和虛擬機器共享EL1硬體資源，步驟會變得更簡單：

1. Linux在EL2直接恢復EL1虛擬機器的暫存器，執行狀態等等

2. EL2跳到虛擬機器EL1開始執行

在系統頻繁進出VM的場景，例如VM需要接收很多中斷時，第一種效能會差第二種很多。

### VHE

為了讓Linux能夠跑在EL2，實現上面說的效能提昇，Arm直接延展架構，讓kernel (如Linux) 能夠在只有少量改動的情況下跑在EL2，EL2經過VHE延展後有以下變化：

- 有兩個translation table base register, TTBR0_EL2 & TTBR1_EL2

- EL2 translation table entry的格式和bit field定義和EL1 translation table entry一樣

- EL2也像EL1有virtual timer和physical timer

- etc..

其實就是把EL2延展成兼容EL1，其中有一點比較特別：

- EL2存取”EL1”暫存器時，會自動改成存取EL2暫存器

什麼意思呢？舉例來說，VHE的EL2讀取TTBR0_EL1，硬體會自動讀取TTBR0_EL2。咦，那如果真的要讀取TTBR0_EL1怎麼辦呢？那要改成讀TTBR0_EL12。

這個特性是為了讓已經存在的kernel，如Linux，可以有較小的改動就能在EL2上執行。因為如果想將Linux跑在EL2，原本Linux中的程式在管理系統的時候全部都是用EL1暫存器，那就要全部改寫成存取EL2的版本才行，因為在EL2存取EL1暫存器對Linux本身沒有意義。再來，假設真的都改好了，那在另一個場景，想要把Linux拿回去跑在EL1又怎麼辦，要複製成兩種，一個EL1-Linux，一個EL2-Linux嗎？還是每一次存取都要判斷是EL1還是EL2?

有了以上的功能，這部份就不用改了，硬體會自動導向，在EL1就存取EL1版本，在EL2就存取EL2版本的暫存器。

範例：

```plain
/* VHE EL2 */
mrs x0, TTBR0_EL1  # x0讀到TTBR0_EL2    <---- 硬體自動導向！
mrs x0, TTBR0_EL2  # x0也是讀到TTBR0_EL2
mrs x0, TTBR0_EL12 # x0讀到TTBR0_EL1

/* non-VHE EL2 */
mrs x0, TTBR0_EL1  # x0會讀到TTBR0_EL1
mrs x0, TTBR0_EL2  # x0讀到TTBR0_EL2

/* EL1 */
mrs x0, TTBR0_EL1  # x0會讀到TTBR0_EL1  <----
```

可以看到，VHE讓程式不用改就能存取到對應EL的暫存器。

## VHE的開關與偵測

EL2程式可以透過HCR_EL2.E2H (EL2 Host)這個bit來控制系統是否使用VHE，寫入1開啟，寫入0關閉，不過，在沒有VHE的系統上寫入1當然沒有效果，而另外有些系統則是VHE強制設定開啟，即EL2不能以非VHE模式運作，HCR_EL2.E2H寫入0也沒有效果。

所以說，有這三種CPU：

1. 沒有VHE功能

2. 有VHE功能，而且軟體可以透過HCR_EL2.E2H切換開啟關閉

3. 有VHE功能，但是不能關閉

1和(2, 3)的分辨可以用ID_AA64MMFR1_EL1.VH來判定，如果值是1，代表系統有VHE功能(1)，如果值是0，代表沒有VHE功能，只能是2或3。

那怎麼分辨2或3呢？可以利用ID_AA64MMFR4_EL1.E2H0，這個判定沒有非常直觀，要注意一下，0代表有VHE並且可以關閉VHE，所以是上面的(2)的情況，小於0則是VHE強制開啟。

|  | ID_AA64MMFR1_EL1.VH == 0 | ID_AA64MMFR1_EL1.VH == 1 | 
|---|---|---|
| ID_AA64MMFR4_EL1.E2H0 == 0 | 沒有VHE功能 (1) | 可以自由開關VHE (2) | 
| ID_AA64MMFR4_EL1.E2H0 < 0 | 違反架構（不應該有這種情況） | VHE強制開啟 (3) | 
