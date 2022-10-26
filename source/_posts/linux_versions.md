---
title: Linux版本號意義
date: 2020-08-30 15:00:15
summary:
tags:
  - Linux
categories: technical
---

在2011年7月以前，linux版本號有三個數字，例如`2.6.39`
其中
* 2 -> 主版本號
* 6 -> 副版本號，奇數為開發版本，偶數為穩定版本
* 39 -> 修訂版本號

2011/7/21，核心版本3.0釋出，Linus為了整齊把版本號換成兩個。
現在的開發流程，新核心版本會從一個為期兩週的"merge window"開始，Linus在這期間會接受各方面新功能的patch，把他們merge到核心中。Merge window過了之後，穩定化階段開始，持續的測試並debug新的核心，每週會發布一個"release candidate"，表示方法是在版本號之後加上`-rc#`，第幾個rc, '#'就代入那個數字，舉例現在最新的核心`5.9-rc2`就是第二個`5.9`版的release candidate。目前每個新核心大概會有8個rc，之後就會正式釋出，下一個版本的merge window重新開始。

上述的核心版本是最新的，版本號走在最前面的，叫做mainline kernel，由Linux原作者Linus Torvalds維護。

Mainline核心正式釋出之後，會轉移到stable分支上，這個分支由Greg Kroah-Hartman維護，這裡的核心會持續受到維護、debug等等，並在版本號上加上第三個數字表示修訂的版本。
通常，stable分支只會維護到最新開發循環的前一到兩個版本，例如現在是mainline `5.9`的`rc2`階段，`5.8`和`5.7`就在stable中維護，`5.6`就不再受到更新。

除此之外，有些核心版本被挑選為longterm，會受到持續的測試與維護，約一兩年。每年至少有一個版本會被選為longterm，滿足對於穩定核心的需求。

這樣看下來點進[https://www.kernel.org](https://www.kernel.org)就應該看得懂那些版本號的意思了！

參考：
Chris Simmonds. (2017). *Mastering Embedded Linux Programming Second Edition.* Packt Publishing
