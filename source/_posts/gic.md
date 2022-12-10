---
title: "GIC筆記-虛擬中斷直接注入之硬體處理"
date: 2022-12-10 20:54:15
summary:
tags:
  - GIC
  - ARMv8
categories: technical
---

## 前言

最近幾週讀了ARM官方的”GICv3 and GICv4 Software Overview”，這本書從軟體的角度帶讀者認識GICv3 v4，而為了不讓篇幅過長，省略了比較特殊的使用場景 e.g. 硬體只實作一個security state的狀況、使用legacy mode等等，閱讀完畢覺得是一個很好入門的介紹，是個可以讓人之後開始參考標準的spec “Arm® Generic Interrupt Controller Architecture Specification GIC architecture version 3 and version 4”的緩衝。

在讀這種文件因為細節非常多，很容易見樹不見林，看完了一堆細節好像都懂了放遠來看就發現思緒連不起來。這篇blog是把書中內容簡單整理成我自己認為好理解的思考方式，會跳過很多基本內容，也會做很多沒有解釋的假設(例如使用一般的設定值)，從一個中斷被外設觸發的流程往外擴展說明，希望也能對想學習這塊的讀者有所幫助。

閱讀過程中我使用Heptabase做了視覺化的筆記，方便理解，可以配合使用:

<https://app.heptabase.com/w/d4dfaf701a143f308454cb962ba9963472a27873bc9a2b2779e89d9430a972ac>

## Disclaimer

此篇文章跳過一般中斷的發送，只說明虛擬中斷直接注入，所以Collection Table, Collection ID的說明跳過。

此外，下面說明的hypervisor要做的工作列出的是必要但不充分的步驟，也很不詳細，只是架構上必須要完成的一些事情。

## 虛擬中斷直接注入步驟

我們從一個外設發送一個LPI(Local Peripheral Interrupt)開始，步驟如下:

1. 外設知道ITS(Interrupt Translation Service)在bus上的位址，然後向`GITS_TRANSLATOR` 寫入Device ID和Event ID，發送一個中斷

2. ITS感知有LPI產生之後，硬體讀取`GITS_BASER[0..7]` 中`Type` 為device table的暫存器得到Device Table的位址。Device Table是一張存在記憶體中的表，負責把Device ID翻譯成對應的Interrupt Translation Table的基地址

3. 硬體從Device Table中找Device ID對應的Interrupt Translation Table的基地址

4. 硬體從Interrupt Translation Table中找Event ID所對應的值，有兩種可能(以下步驟假設第二種)

   1. 對應出一個collection ID和INTID

   2. 對應出一個vINTID和vPE ID (硬體虛擬中斷注入)，有可能還有一個doorbell pINTID

5. 硬體從`GITS_BASER[0..7]` 中`Type` 為vPE Table的暫存器得到vPE Table的位址

6. 硬體從vPE Table中找第四步得到的vPE ID所對應的target Redistributor以及virtual LPI Pending Table address

7. 硬體觀察該target Redistributor的`GICR_VPENDBASER` 是否符合第六步得到的virtual LPI Pending Table address，有兩種可能(以下步驟假設符合)

   1. 符合，直接進行硬體虛擬中斷注入

   2. 不符合，若第四步Interrupt Translation Table有提供doorbell pINTID，則傳入實體中斷

8. 正在執行VM的PE直接被中斷，開始執行guest的中斷處理流程，不需退出到hypervisor，使用Virtual CPU Interface與硬體溝通，並直接寫`EOI` 暫存器來結束中斷處理

以上便是虛擬中斷注入的處理過程，軟體必須在這之前做好各種初始化，其中包含:

### Device Table

Device Table全局只有一份，軟體必須分配所需的記憶體空間，並把起始位址填入`GITS_BASER[0..7]` 中`Type` 為device table的暫存器，接著把每個Device ID對應的Interrupt Translation Table起始位址填入

### Interrupt Translation Table

軟體必須為系統中的Device ID各分配一個Interrupt Translation Table的空間，每個Table會被Device Table中對應的一個entry指到，裡面填上各個Event ID對應的vINTID:vPE ID值，以及可選的doorbell pINTID

### vPE Table

軟體必須分配一個vPE Table空間，並把起始位址填入`GITS_BASER[0..7]` 中`Type` 為vPE Table的暫存器，裡面填入vPE ID對應的target redistributor和vLPI Pending Table的位址

### 小結

以上三種表可以讓一個外設觸發的Device ID:Event ID轉換成

* vINTID (虛擬中斷號)

* target redistributor (哪個PE)

* vLPI Pending Table address (該PE是否正在執行對應的vPE)

這些資訊使硬體能進行虛擬中斷直接注入

## 表格填寫

以上三種表

* Device Table

* Interrupt Translation Table

* vPE Table

在填入內容時其實都不是直接在寫記憶體，而是利用向ITS發送指令的方式讓硬體自動填入正確的entries，而發送指令的方式是寫入一個也是存在記憶體中的command queue，command queue所使用的空間也是軟體必須負責初始化，並把起始位址，queue head/tail的位址填入`GITS_BASERn`, `GITS_CREADR`, `GITS_CWRITER` 暫存器中來告訴ITS要建立哪些mappings，這邊不深入解釋。

## hypervisor設定虛擬中斷直接注入

假設VM `x` 的 vPE `y`，跑在PE `z`上希望直接獲取Device ID `d`, Event id `e`的中斷，而虛擬中斷號是`f`:

1. 將`d:e`這個組合在對應的Interrupt Translation Table entry填入vINID \= `f`, vPE ID \= `y`

2. vPE Table中 `y`的entry要填上`z`的redistributor以及`y`所使用的vLPI Pending Table位址

## hypervisor搬遷vPE

假設hypervisor想把vPE `y` 要從PE `x1` 搬遷到`x2` 必須操作:

12. `x1` redistributor的`GICR_VPENDBASER` 清掉，搬到`x2` redistributor的`GICR_VPENDBASER`

13. 把vPE Table vPE ID `y` 的target redistributor從`x1` 的改成`x2` 的

## 各名詞性質整理

* PE: Processing Element

  * 硬體

* CPU Interface

  * 硬體，使用`mrs` , `msr` 指令控制

* Redistributor

  * 硬體，使用MMIO控制

  * 每個PE一份

* Distributor

  * 硬體，使用MMIO控制

  * 一個GIC一個

* Interrupt Translation Service

  * 硬體，使用MMIO控制

  * 可能多個，這篇文章假設一個

* Device Table

  * 軟體結構，必須分配記憶體

  * 要寫他必須使用command queue

* Interrupt Translation Table

  * 軟體結構，必須分配記憶體

  * 要寫他必須使用command queue

* vPE Table

  * 軟體結構，必須分配記憶體

  * 要寫他必須使用command queue

* Command Queue

  * 軟體結構，必須分配記憶體

  * 直接寫入指令使硬體操作

    * Device Table

    * Interrupt Translation Table

    * vPE Table

* LPI Configuration Table

  * 軟體結構，必須分配記憶體

  * 直接寫入

* LPI Pending Table

  * 軟體結構，必須分配記憶體

  * 直接寫入

* vLPI Configuration Table

  * 軟體結構，必須分配記憶體

  * 直接寫入

* vLPI Pending Table

  * 軟體結構，必須分配記憶體

  * 直接寫入
