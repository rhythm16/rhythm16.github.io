---
title: 第一次給Linux Kernel發patch
date: 2020-10-28 21:20:15
summary:
tags:
  - Linux
  - LKML
categories: technical
---

一路的故事是滿意外的，有天跟同學吃飯聊到了我讀的32bit x86架構的memory reference，又再一次提醒了我我對於64bit一無所知的事實。回家手癢google了"x86_64 memory reference"，前幾個搜尋結果是linux kernel的[documentation](https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt)，瀏覽了一下竟然發現有句話讀起來怪怪的：
> "16M TB" might look weird at first sight, but it's an easier to visualize size
notation than "16 EB", which..(後省略)

"easier"後面少了一個"way"啊！
這在平常可能不算什麼，就少個字，也不至於影響閱讀，但我馬上想到了[jserv教授](http://wiki.csie.ncku.edu.tw/User/jserv)之前在facebook發文提到了一個四歲小孩在爸爸工作時看到了kernel排版少了一個'='，爸爸就幫忙發了一個[patch](https://www.reddit.com/r/linux/comments/2pqqla/kernel_commit_4_year_old_girl_fixes_formatting_to/)，也被接受了，說明這種重要的大專案連排版這種小錯也是要修正的。

我之前就聽過linux kernel mailing list跟貢獻給linux kernel的一些規則，不過一直沒有深入了解，剛好趁著找到文件缺漏的機會可以學習一下，畢竟菜雞如我就算想主動貢獻這類開源專案也會因為不知道要寫/改什麼code而無法參與。

## 資料們
一開始不外乎就是開始各種google
我的流程幾乎完全照著這個
* [纪念第一次向Linux内核社区提交patch](https://zhuanlan.zhihu.com/p/87530337)

還有以下這幾個也稍微讀了一遍：
* [10 things you need to do before sending your patch to LKML](https://yaapb.wordpress.com/2012/12/14/10-things-you-need-to-do-before-sending-your-patch-to-lkml/)
* [Documentation/process/submitting-patches](https://www.kernel.org/doc/html/latest/process/submitting-patches.html#submittingpatches)
* [Why does the Linux kernel repository have only one branch? [closed]
](https://stackoverflow.com/questions/30268332/why-does-the-linux-kernel-repository-have-only-one-branch/30268416)

連結裡頭跟網路上其他資料的流程不一定100%吻合，我認為是因為有些步驟被一些腳本自動化了，所以做法有新舊差異

## 背景
> 這段非常大略的描述linux kernel社群運作方式，很多細節省略。我的經驗與所知非常非常的少，很有可能有錯誤、缺漏或容易產生誤會的地方

Linux版本號意義可以參考[這篇](https://hackmd.io/@rhythm/B10t9i_Xw)。
Linux kernel開發社群的溝通方式是plain old email，要找人討論、聯絡維護者就是寄信 (github 上面的linux kernel只是一個mirror，即時顯示最新狀態)。社群有一個主要的mailing list，叫做Linux Kernel Mailing List (LKML)，就是個email廣播空間，可以進行討論，或是宣告事項。寄信到那個mailing list，所有有訂閱的人都會收到那封email。
所以對，LKML每天都有好幾百甚至上千封email，訂閱方式google查就有，訂閱前請三思。只想看看內容可以到[這個連結](https://lkml.org)
Linux kernel的大頭是初始作者本人Linus Torvalds，基本上code/文字進到了他本人的git repository，就是"進了linux kernel"，但是一般人不會直接寄信給Linus，而是把自己修改的內容寄給Linus下面負責的維護者，並同時Cc LKML，等待自己的patch慢慢送到上游，最後進到Linus的repo。
維護者與維護者的溝通，還有Linus與維護者們的溝通大多不會Cc LKML，他們的工作方式我還沒有深入去查過。

## 步驟
第一步就是弄個source code來：
也可以用`git clone`，但下載實在太久了所以我下載tarball
```bash
$ wget https://github.com/torvalds/linux/archive/v5.9.tar.gz
$ tar -zxvf v5.9.tar.gz # decompress and untar
```
因為是下載壓縮包，所以是還沒有使用git:
```bash
$ cd linux-5.9
$ git init
$ git add .
$ git commit -m "orig"
$ git branch develop
$ git checkout develop
# 以下 config 後面script會用到！
$ git config --global user.email "r09922117@csie.ntu.edu.tw"
$ git config --name user.name "Wei Lin Chang"
```
接著就進source把少的"way"加上去
```bash
$ vim Documentation/x86/x86_64/mm.rst
# 改好之後就commit
$ git add .
$ cd ../../.. # back to root directory
# commit:
$ git commit -s -v
```
commit不加 -m 會自動跳出一個GNU nano讓你編輯commit message，而-s是跳出時自動加入一行`Signed-off-by: "name" <email>`的簽名，這是標準發kernel patch要加上的，所以建議使用。-v則是在下方顯示你所做更改的diff訊息，只是幫助撰寫commit message用。

交patch是有標準格式的，如果沒弄好好心的maintainer可能會回信給你提醒你哪裡錯了，如果太誇張我猜他就直接不理你了，以下是格式：
```bash
subsystemName: short description
# blank line
patch content description
# blank line
Signed-off-by: "name" <email>
```
* 第一行是子系統名稱以及patch標題，應盡量精簡到位，讓維護者一下就知道你做了什麼
* 第二部分是patch的內容，時態用現在式，語態用主動形式
* 第三部分是使用-s生成的簽名
* 中間要有空行

如果有不清楚的就回去看你改的檔案的commit歷史看看之前的人怎麼寫的，以下是我最後的message:
```
Documentation: x86: fix a missing word in x86_64/mm.rst.

This patch adds a missing word in x86/x86_64/mm.rst, without which
the note reads awkwardly.

Signed-off-by: Wei Lin Chang <r09922117@csie.ntu.edu.tw>
```
接著生成patch:
```bash
$ git format-patch master
```
這指令與master branch比較，生出.patch檔，會放在root directory
再來用script檢查patch有沒有問題：
(光這個檢查的script就快7000行XD)
```
$ ./scripts/checkpatch.pl 0001-Documentation-x86-fix-a-missing-word-in-x86_64-mm.rs.patch
total: 0 errors, 0 warnings, 8 lines checked

0001-Documentation-x86-fix-a-missing-word-in-x86_64-mm.rs.patch has no obvious style problems and is ready for submission.
```
然後也是用script查看這個patch要給誰：
```bash
$ ./scripts/get_maintainer.pl -f Documentation/x86/x86_64/mm.rst

Thomas Gleixner <tglx@linutronix.de> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
Ingo Molnar <mingo@redhat.com> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
Borislav Petkov <bp@alien8.de> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
x86@kernel.org (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
"H. Peter Anvin" <hpa@zytor.com> (reviewer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
Jonathan Corbet <corbet@lwn.net> (maintainer:DOCUMENTATION)
linux-kernel@vger.kernel.org (open list:X86 ARCHITECTURE (32-BIT AND 64-BIT))
linux-doc@vger.kernel.org (open list:DOCUMENTATION)
```
使用`git send-email`來送信，先寄一次給自己測試看看
```bash
$ git send-email --to r09922117@csie.ntu.edu.tw \
  0001-Documentation-x86-fix-a-missing-word-in-x86_64-mm.rs.patch
```
確認格式都正確沒有跑掉，就可以正式寄給maintainer & LKML了！
`git send-email`執行之後會請你確認，然後輸入密碼，寄出
```bash
$ git send-email --to corbet@lwn.net \
                 --cc tglx@linutronix.de \
                 --cc mingo@redhat.com \
                 --cc bp@alien8.de \
                 --cc x86@kernel.org \
                 --cc hpa@zytor.com \
                 --cc linux-kernel@vger.kernel.org \
                 --cc linux-doc@vger.kernel.org \
  0001-Documentation-x86-fix-a-missing-word-in-x86_64-mm.rs.patch
```
我收件人選擇Jonathan Corbet是因為顯示了他是Documentation的maintainer。

## 後記與一些疑問

* 我寄了patch(10/15)之後幾天都沒有收到回覆，以為會石沈大海，結果(10/22)收到了回覆：
```
Applied, thanks.

jon
```
* 又過了幾天我去查才發現Jonathan Corbet就是Linux Device Driver作者本人XDD我完全有眼不識泰山，竟然有機會跟大師互發email實在是太酷了
* 我一開始拿來改的tree似乎是錯誤的，我要改Documentation的話感覺應該是要使用Jonathan Corbet在lwn.net上的git tree來改才對
* 這個Documentation的patch真的是超級新手友善，因為不是跑code所以很多的測試都省掉了，改code官方建議是要找四五台電腦compile跑看看確認沒問題再提patch
* 現在的linux kernel是用sphinx來把.rst變成html，所以理論上我改Documentation也應該用sphinx測試過一次XD
* 10/25 Linux v5.10-rc1釋出，可以在信裡面看到Linus列了裡面有我commit的Jonathan Corbet的commit
* 我沒有注意到merge window的問題，但運氣太好了剛好就在v5.10的其中發出去，不知道如果沒有在merge window中發patch會怎樣
* [Why Github can't host the Linux Kernel Community](https://blog.ffwll.ch/2017/08/github-why-cant-host-the-kernel.html)

## LMKL.org上的連結
* [我的patch](https://lkml.org/lkml/2020/10/15/60)
* [Jonathan Corbet的回覆](https://lkml.org/lkml/2020/10/21/669)
* [Jonathan的GIT PULL](https://lkml.org/lkml/2020/10/23/849)
* [Linus: Linux v5.10-rc1](https://lkml.org/lkml/2020/10/25/267)
