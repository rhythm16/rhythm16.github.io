---
title: Linux/ELF動態鏈接部分機制(GOT&PLT)
date: 2022-10-20 21:20:15
summary: 
tags: 
  - tag1
  - tag2
  - tag3
categories: OMG
---

在讀作業系統、計算機結構等科目或是在學習程式的過程中，會讀到一個機制叫做"動態鏈接(dynamic linking)"，它可以使得多個程式共用函數，進而得到節省記憶體的效果。

一個例子就是C語言<stdio.h>中的 printf()，如果系統中所有要對終端輸出字串的程式都把printf的程式碼包含進原始碼再編譯，不僅會浪費磁碟空間（儲存了一大堆printf），程式同時執行時主記憶體也就被很多個printf函數所佔用。

所以說比較有效率的方式是讓系統的主記憶體只存著一份printf函數，其他所有程式要輸出字串時讓CPU跳到唯一存放printf的地方執行，結束再跳回原來的程式就行了。之所以叫做動態，就是因為程式編譯時，並不會知道所呼叫的函數在哪裡，無法在編譯時「鏈接」，只能在程式從硬碟載入記憶體時動態地在系統的幫助下完成鏈接。

* [鏈接(linking)](https://en.wikipedia.org/wiki/Linker_(computing))是指讓一隻程式在呼叫函數或使用外部變數時知道要往哪裡找

這篇文章希望能夠為學過程式語言，但是對於程式究竟如何真正運作有好奇的人提供一部分更深入的內容，共分成三大段一小段：
1. 第一大段「Linux/ELF動態鏈接流程概述」說明GOT和PLT是什麼，還有他們在系統動態鏈接中扮演什麼角色，以及解釋其中所使用到的名詞和先備知識。
2. 第二大段開始以組合語言的層級(instruction/assembly level)分析一個簡單的hello world C程式的main函數部分。
3. 第三大段深入分析程式如何在GOT、PLT、動態鏈接器的幫助下實現動態鏈接。
4. 最後一小段提出一些在研究中引導出的一些雜談/問題，可以思考看看(其實就是我也不懂的部分XD)

## 1. Linux/ELF動態鏈接流程概述
ELF檔案格式（當前linux上的主流執行檔格式）儲存在硬碟中時，分成許多section，如下圖 [ELF格式wiki](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
![ELF格式wiki](https://i.imgur.com/WoGNVvG.png)
其中幾個section在圖中有顯示，ELF header紀錄整個檔案的總體訊息，Program header table紀錄檔案從硬碟被載入主記憶體時的編排方式(會和存在硬碟中不大一樣)，Section header table紀錄各section訊息(如每個section名稱、長度、位置等等)，.text是程式碼，.rodata以及.data都是存資料用。
除了上述的幾個常見的section，要了解動態鏈接必須多知道幾個section，主要就是.got和.plt這兩個。

### GOT (Global Offset Table)
GOT存著很多個一樣大小的元素，形成一個table(基本上就是array)。每個元素都是一個指標(所以說32位元機器一個元素4個byte，64位元就是8個byte)，指向程式所需要的變數或者是函數，也有可能是系統需要的指標。

### PLT (Procedure Linkage Table)
PLT存著很多個一樣大小的元素，形成一個table(基本上就是array)。每個元素都是一小段程式碼，第一個元素是公共plt，負責呼叫動態鏈接器。從第二個開始每個元素分別對應到一個動態鏈接的函數，會使用該函數所對應之GOT元素。
> 意思就是一個動態鏈接函數對應到一個GOT元素(指標)，一個PLT元素(執行碼)。CPU執行PLT存的程式碼，使用GOT存的指標想辦法跳到該函數

整個動態鏈接程序就是
1. 主程式第一次呼叫函數
2. 程式跳到該函數的plt元素
3. 該函數之plt使用其對應到的GOT元素，想要跳到目標函數
4. (a)因為第一次呼叫，尚未鏈接，GOT指到的地方一樣是函數的plt，結果又跳回該plt元素，把代表該函數的數字推到stack上
(b)plt繼續執行，跳到公共plt
5. 公共plt執行一些東西
6. 公共plt呼叫動態鏈接器
7. 動態鏈接器從stack觀察主程式想使用哪個函數，更改第3步所使用的GOT元素的內容，使其指向函數的真正位置，並引導程式執行目的函數
8. 之後再次呼叫該函數時，第3步時plt使用的GOT已經存著函數的真正位址，就會直接跳到函數執行，這個動態鏈接的方式叫做Lazy Linking。

下圖舉例鏈接一個叫foo的函數，數字n的位置就是第n步指令的所在之處，注意{系統用GOT}不是真的只有兩個，詳細情形會在第三大段說明
![](https://i.imgur.com/sxo16t3.png)


## 2. hello world.c 分析

> 系統實際的implementation非常複雜，而且不同版本行為都可能略有改變甚至完全不同

以下環境使用
* Linux 4.13.0-36-generic on x86_64
* Ubuntu 16.04.1
* gcc 5.4.0

我們從一個最簡單，第一堂程式課會教的hello world程式開始：
```clike
#include <stdio.h>
int main()
{
    printf("hello world!\n");
}
```

編譯，執行
```
$ cc hello.c #hello.c is the file name of the program above
$ ./a.out
hello world!
$
```

接下來觀察gcc的輸出，使用objdump看main：
（程式中有很多其他輔助執行的函數，先略過他們，有些後面會提到）

```
$ objdump -d ./a.out
...
0000000000400526 <main>:
  400526:	55                   	push   %rbp
  400527:	48 89 e5             	mov    %rsp,%rbp
  40052a:	bf c4 05 40 00       	mov    $0x4005c4,%edi
  40052f:	e8 cc fe ff ff       	callq  400400 <puts@plt>
  400534:	b8 00 00 00 00       	mov    $0x0,%eax
  400539:	5d                   	pop    %rbp
  40053a:	c3                   	retq   
  40053b:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
...
```

### 上面在講什麼？
`0000000000400526 <main>`是指程式的main函數放在ELF檔離檔案開頭526（16進位）的位置開始，多的400000是gcc設定程式載入記憶體時main在400526的地方。
接著有8行組合語言，每行編排方式左至右分別為：
1. 位址，離檔案開始幾個byte+400000（如第一行的400526，這裡省略了前面的10個0）
2. 機器語言（如第一行`55`，也是16進位）
3. 機器語言所對應的組合語言（如第一行 `push %rbp`）

兩個16進位字元就是一個byte，所以像第二行`48 89 e5`兩個一組隔開就是這個原因。
而組合語言的部分，第一個字是指令（`push`, `mov`等等），之後是使用的暫存器或所使用的資料，
'%' 代表暫存器，後面接暫存器的名字，如%rbp就是rbp暫存器。
'$' 指數字，如$0x0就是16進位0，就是0。
'#' 是objdump幫我們加的註解，幫助我們理解使用。
'<>' 告訴我們位址指到程式哪邊，可以見到gcc把printf優化成了puts

> r開頭的暫存器是64位元，40052a的edi是rdi暫存器的下半32bit部分，同樣400534的eax是rax暫存器的下半32bit部分

如果只是要了解程式在做什麼的話，機器語言的部分是不需要看的，但接下來有部分會探討一些機器語言的編碼方式，因為挺有趣的

### main函數流程
* 400526 & 400527：建構[stack frame](https://en.wikipedia.org/wiki/Call_stack#STACK-FRAME)，先把指到stack base的rbp推到stack上(400526)，再把rsp 複製到rbp，設置好給main使用的stack。
* 40052a：這個指令幫我們準備參數，拿來餵給幫我們輸出的函數，在x86架構的linux平台上函數的第一個參數是被放在rdi暫存器傳入的，不過需要的空間沒有那麼大，所以使用edi暫存器(rdi的下32bit部分)，所傳入的資料是一個指標，使用readelf看看傳入的4005c4指到哪裡：
```
$ readelf -S ./a.out
There are 31 section headers, starting at offset 0x19d8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
...
  [16] .rodata           PROGBITS         00000000004005c0  000005c0
       0000000000000011  0000000000000000   A       0     0     4
...
```
執行`readelf -S` 可以看到執行檔裡頭的各個區段(section)的資訊，可以看到`.rodata`這個section的`Address`從4005c0開始，而`Size`是11(注意都是16進位)，也就是說`.rodata`包含了我們剛剛在關注的4005c4位址。
`.rodata`是read only data的意思，現在使用`objdump -s`來看看`.rodata`放了什麼～
```
$ objdump -s ./a.out

./a.out:     file format elf64-x86-64

Contents of section .interp:
 400238 2f6c6962 36342f6c 642d6c69 6e75782d  /lib64/ld-linux-
 400248 7838362d 36342e73 6f2e3200           x86-64.so.2.    
 ...
Contents of section .rodata:
 4005c0 01000200 68656c6c 6f20776f 726c6421  ....hello world!
 4005d0 00                                   .               
 ...
```
一樣關注`.rodata`部分就好，可以確認左邊是從4005c0開始，沒有錯。中間就是`.rodata`裡頭的詳細資料了，右邊則是`objdump`好心幫我們把可以轉換成ascii code的字元顯示出來的樣子，只有一個點代表數值無法顯示成ascii字元。
數一下4005c4在哪裡，原來就是`hello world!`的起始位置，終於揭曉了，上上面
`mov    $0x4005c4,%edi`就是在把指到`hello world!`字串的指標放進暫存器edi裡面！
```
這樣子數：（空白也是一個byte, 對應到0x20）

4005c4----------|
4005c3-------|  |
4005c2-----| |  |
4005c1---| | |  |
         | | |  |
4005c0-01000200 68656c6c 6f20776f 726c6421  ....hello world!
```

* 40052f: `callq`，跳轉至程式400400位置。
> 這邊可以看一下機器語言`e8 cc fe ff ff`，前面`e8`對應到callq，後面則是一個參數，用來計算要跳去的位址，這個指令用的計算方式是使用現在`%rip`中的值加上參數，作為下一個`%rip`的數值，也就是要跳去的地方。
在x86_64架構中，`%rip`存放著下一個即將執行的指令位址(所謂的instruction pointer/program counter)，所以說執行40052f時`%rip`存著的是400534。

> 注意參數的部分因為這台機器為[little-endian](https://en.wikipedia.org/wiki/Endianness)，所以寫成`cc fe ff ff`，而不是`ff ff fe cc`，而且這個數就是[2's complement表示法](https://en.wikipedia.org/wiki/Two%27s_complement)的負308(10進位)，負134(16進位)，而400534-134 = 400400，果然！

> 從這個的格式可以看出offset的範圍應該是4個byte能夠表示的範圍，也就是2^32^，使用[線上x86_64 disassembler](https://defuse.ca/online-x86-assembler.htm#disassembly2)測試了幾個數值之後發現offset在CPU中會自動被sign extend，以符合64位元的數值運算，而且offset有分正負，
所以至多可以表示%rip-2^31^ ~ %rip+(2^31^-1)。

* 400534: `mov    $0x0,%eax` 這個mov把0放進%eax，eax在這用於函數回傳值，正在準備為main函數回傳0，可以對應到程式中的`return 0;`，雖然我們沒有寫，但gcc編譯器還是幫我們加上去了。
* 400539: 把前面指令(400526)所推到stack上的rbp給pop回來
* 40053a: 接著return回呼叫main的環境。
* 40053b: 我們的main函數理論上40053a就返回了，不會執行到這裡，我觀察程式中其他可執行的區段發現很多在結束後都有類似這幾個nop指令，我推測是為了讓各區段結尾對齊至位址個位數為0，這部分我不確定，歡迎高手指點。

## 3. GOT、PLT、動態鏈接器之舞
> 這一大段可以對照第一大段的流程概述，會比較好理解！
> 
看完main做了什麼，可以來看程式如何呼叫外部函數了，動態鏈接的開端是main中40052f位址的指令`callq 400400`，對應到前面流程的第1步。
同樣的使用`objdump -d`觀察400400究竟在做什麼：
```
$ objdump -d ./a.out

./a.out:     file format elf64-x86-64
...
Disassembly of section .plt:

00000000004003f0 <puts@plt-0x10>:
  4003f0:	ff 35 12 0c 20 00    	pushq  0x200c12(%rip)        # 601008 <_GLOBAL_OFFSET_TABLE_+0x8>
  4003f6:	ff 25 14 0c 20 00    	jmpq   *0x200c14(%rip)        # 601010 <_GLOBAL_OFFSET_TABLE_+0x10>
  4003fc:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000400400 <puts@plt>:
  400400:	ff 25 12 0c 20 00    	jmpq   *0x200c12(%rip)        # 601018 <_GLOBAL_OFFSET_TABLE_+0x18>
  400406:	68 00 00 00 00       	pushq  $0x0
  40040b:	e9 e0 ff ff ff       	jmpq   4003f0 <_init+0x28>
...
```
main一跳到400400又執行了另一個跳轉(jmpq)指令，跳到`*0x200c12(%rip)`這個地方。執行到400400時，`%rip`的內容是400406，而`*0x200c12(%rip)`這個表達式的意思是把`%rip`中的內容加上0x200c12作為指標，其所指到的記憶體內容。
`%rip`現在是下一個指令的400406，400406 + 200c12 = 601018
> 第一次研究到這邊的時候一直搞不清楚，原來這裡是兩層間接
> 1. 使用計算出的601018是一個記憶體位址（指標），那個位址存放的資料是要放進`%rip`的，作為下一個指令的位址（還是一個指標）
> 2. CPU再將新的`%rip`中的值(記憶體位址601018所存的資料)作為記憶體位址，去記憶體取得下一個指令執行。

所以說下一個指令的位址是存在記憶體601018的地方，同樣使用`objdump -s`觀察：
```
$ objdump -s ./a.out

./a.out:     file format elf64-x86-64
...
Contents of section .got:
 600ff8 00000000 00000000                    ........        
Contents of section .got.plt:
 601000 280e6000 00000000 00000000 00000000  (.`.............
 601010 00000000 00000000 06044000 00000000  ..........@.....
 601020 16044000 00000000                    ..@.....        
 ...
 ```
 > 之所以有.got和.got.plt是因為ELF檔案把GOT拆成變數用的部分(.got)以及函數用的部分(.got.plt)，這個例子中只注意.got.plt
 
 .got.plt就是第一大段所說的GOT，現在的狀態如下：
 ```
0x601000 .got.plt[0] = 0x0000000000600e28
0x601008 .got.plt[1] = 0x0000000000000000
0x601010 .got.plt[2] = 0x0000000000000000
0x601018 .got.plt[3] = 0x0000000000400406
 ```
 
 兩個character一個byte，所以601018在0604開始的地方。`%rip`是64-bit，也就是8個byte，0604開始的8個byte會被放入`%rip`，作為下一個指令的位址，`06044000 00000000`，但真的放進去時會變成`00000000 00400406`，因為little-endian!
 
從400400搞了那麼多，結果竟然是跳到400400的下個地方400406(這裡對應流程的第2,3步)。

400406：這個指令簡單，把0推到stack上(第4a步)。這個0代表第一個動態鏈接函數，鏈接器才知道是我們要的puts。

40040b: 接著跳到4003f0(第4b步)，公共.plt部分。(這邊可以算算如何得到4003f0的！)

4003f0: 把`0x200c12(%rip)`推到stack上，計算方式與之前相同，可以直接看`objdump`告訴我們的，是601008，程式的global offset table+8，GOT的第二個元素。

4003f6: 再次跳轉(第6步)，到601010，程式的global offset table+10(這個10也是16進位)，GOT的第三個元素。

到了這裡其實已經沒辦法再靜態分析下去了，因為第一大段明明說第6步該呼叫動態鏈接器了，而如果看上面.got.plt的內容會發現執行檔601008和601010都只是一大堆0，鏈接器應該不在位址0的地方吧？

解答是.got.plt的前三個元素是有特殊意義的，系統在載入程式的時候，會主動把GOT第二和第三個元素應該要有的資料放進去，而因為那些資料一定要等程式載入記憶體準備執行了才會知道，所以編譯器先在那些地方都填入0。

參考這張圖，原擷取自[CSAPP](https://csapp.cs.cmu.edu)，但我在[這裡](https://www.freebuf.com/articles/system/135685.html)看到的
![](https://i.imgur.com/4ukPcxp.png)
GOT[1] (我們例子的601008)是鏈接器需要的資料，而GOT[2] (我們例子的601010)就是動態鏈接器所在的位址！

接下來我們使用gdb來看看程式運行時GOT是不是真的被改變了！
```
$ gdb ./a.out
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
...
(gdb) b main
Breakpoint 1 at 0x40052a
(gdb)
```
第一個指令輸入`b main`在main設置斷點，可以觀察gdb特別跳過前兩個設定stack frame的指令，把斷點設在40052a
接著執行程式，讓它停在斷點：
```
(gdb) r
Starting program: /home/rhythm/a.out 

Breakpoint 1, 0x000000000040052a in main ()
(gdb) 
```
這時候程式已經開始了，來看看GOT的內容：
> 如果你的GOT[1],GOT[2]並沒有改變，請看結尾的第一個部分
```
(gdb) x/8xw 0x601000
0x601000:	0x00600e28	0x00000000	0xf7ffe168	0x00007fff
0x601010:	0xf7dee870	0x00007fff	0x00400406	0x00000000
```
指令x是examine，檢視，斜線後面分別指定檢視幾個單位、輸出格式和單位，8xw就是8單位，16進位，word為單位(4個byte)
```
x / 8 x w
|   | | |
|   | | ⎣_以word(4bytes)為一個單位
|   | |
|   | |
|   | ⎣_x代表用16進位顯示
|   |
|   顯示8個單位
|
examine(檢視)
```
解讀一下輸出（x/8xw的4個byte裡頭的endianness幫我們調整好了，但是4個bytes之間沒有）
```
0x601000 .got.plt[0] = 0x0000000000600e28
0x601008 .got.plt[1] = 0x00007ffff7ffe168
0x601010 .got.plt[2] = 0x00007ffff7dee870
0x601018 .got.plt[3] = 0x0000000000400406
```
.got.plt第二項和第三項真的被改變了，看看公共plt要跳到的601010是哪裡吧：
```
(gdb) disassemble 0x00007ffff7dee870
Dump of assembler code for function _dl_runtime_resolve_avx:
   0x00007ffff7dee870 <+0>:	push   %rbx
   0x00007ffff7dee871 <+1>:	mov    %rsp,%rbx
   0x00007ffff7dee874 <+4>:	and    $0xffffffffffffffe0,%rsp
   0x00007ffff7dee878 <+8>:	sub    $0x180,%rsp
   0x00007ffff7dee87f <+15>:	mov    %rax,0x140(%rsp)
...
   0x00007ffff7dee9ab <+315>:	vmovdqa 0xc0(%rsp),%ymm6
   0x00007ffff7dee9b4 <+324>:	vmovdqa 0xe0(%rsp),%ymm7
---Type <return> to continue, or q <return> to quit---
```
持續按enter可以繼續顯示更多，從第一行結果可以看到第6步真的跳進動態鏈接器的函數`_dl_runtime_resolve_avx`了！
至於動態鏈接器如何完成工作就不說明了，因為我也不知道。不過來看一下第3步使用的GOT元素有沒有成功地被鏈接器改成我們想要的函數位址
```
(gdb) b *main+0xe     #在callq結束的地方設置斷點(鏈接器已完成工作)
Breakpoint 2 at 0x400534
(gdb) c               #c是continue，繼續執行
Continuing.
hello world!          #字串被印出

Breakpoint 2, 0x0000000000400534 in main ()
(gdb) x/8xw 0x601000  #再看一次.got.plt內容
0x601000:	0x00600e28	0x00000000	0xf7ffe168	0x00007fff
0x601010:	0xf7dee870	0x00007fff	0xf7a7c690	0x00007fff
```
第四個元素從原本的`0x00400406 0x00000000`被鏈接器改成`0xf7a7c690 0x00007fff`，看一下變成哪個地方了：
```
(gdb) disassemble 0x00007ffff7a7c690
Dump of assembler code for function _IO_puts:
   0x00007ffff7a7c690 <+0>:	push   %r12
   0x00007ffff7a7c692 <+2>:	push   %rbp
   0x00007ffff7a7c693 <+3>:	mov    %rdi,%r12
   0x00007ffff7a7c696 <+6>:	push   %rbx
   0x00007ffff7a7c697 <+7>:	callq  0x7ffff7a98720 <strlen>
...
   0x00007ffff7a7c74c <+188>:	jae    0x7ffff7a7c7f0 <_IO_puts+352>
   0x00007ffff7a7c752 <+194>:	lea    0x1(%rax),%rdx
---Type <return> to continue, or q <return> to quit---
```
終於！我們要的puts函數的指標被存入.got.plt了(第7步)，這下子下一次呼叫printf時，從main進到puts的plt就可以直接跳到函數了！

## 4. 結尾
其實我在做這篇文章中的小實驗的時候並不是很順利，我一開始想說用筆電原
生Linux 5.3 Ubuntu 18.4，測試過程發現程式開始時GOT[1],GOT[2]的內容並沒有在程式載入被填上鏈接器需要的資料和鏈接器的函數位址，反而GOT[3]，也就是動態鏈接器要負責改成我們所需函數的位置，在載入時就已經是我們要的函數位址了，我真的不知道是什麼神秘機制（求高手指點），而且不知道是不是因為[ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)，執行時整個程式的offset也跟存在ELF中的不一樣。
後來換使用amazon ec2的Linux 4.15 Ubuntu 18.04原本可以，寫到一半不知道是不是遇到系統更新，突然變成像上面那樣直接在載入時就GOT就鏈接好了，最後在windows上開了個相對舊版的Linux/Ubuntu才得到這篇文章中的行為。
後來我又試了另一台Linux 5.5 Arch Linux，是預期中的行為，所以也不知道是不是linux版本所造成的差異。

還有，原本文章主題是「Linux動態鏈接機制」，後來想想才意識到只講GOT/PLT，鏈接器如何運作都不會，標題還這樣下絕對是貽笑大方，才改成現在的標題XD

### 主要參考資料
這篇文章絕大部分的內容都來自下面三個連結，他們對於GOT/PLT機制的說明都更加詳細，也說得更好，但更重要的是他們都有講到一件這篇沒有的內容，就是為什麼要有GOT/PLT這樣的設計。這裡只有機械式的帶過流程，目的是希望讓一個對這方面不熟但有興趣的人能更無痛理解這個小主題，還有實際地呈現系統運作的奧妙！
* [计算机原理系列之八 ——– 可执行文件的PLT和GOT](https://luomuxiaoxiao.com/?p=578)
* [Linux中的GOT和PLT到底是个啥？](https://www.freebuf.com/articles/system/135685.html)
* [PLT and GOT - the key to code sharing and dynamic libraries](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html)
### 其他參考資料
* [Linux x86 Program Start Up](http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html)，我們分析是直接從main函數開始，這篇就講Linux的運行跑到main之前到底要做哪些事(很多！)
* [virtual memory](https://en.wikipedia.org/wiki/Virtual_memory)，這篇文章不會用到，但我怕有人以為ELF檔的位址就是實際的位址所以還是在這裡提一下有這個東西

### 一些問題/小討論
* 為什麼.rodata區段前面有四個bytes`01000200`? 
[Why static string in .rodata section has a four dots prefix in GCC?](https://stackoverflow.com/questions/34733136/why-static-string-in-rodata-section-has-a-four-dots-prefix-in-gcc)
* gcc在什麼情況下會把printf()換成puts()，如果原始程式有簡單printf() (可用puts代替的)，以及會使用到printf中puts沒有的功能的話，編譯器還是會把簡單的printf()換成puts()嗎，還是全部使用printf()?
* 動態鏈接器如何完成它的工作？
這個問題應該就非常複雜了，會牽扯到整個OS的運行，鏈接器至少要知道有哪些函式庫在記憶體中，還有當前實體記憶體配置狀態等與系統運作密切相關的內容
* 前面有一個部分在講解程式有個方法可以跳%rip-2^31^ ~ %rip+(2^31^-1)，如果程式超大要跳超過這個範圍會如何反應？
* 為什麼有個0x400000的offset?
與linker script有關，[Why Linux/gnu linker chose address 0x400000?](https://stackoverflow.com/questions/14314021/why-linux-gnu-linker-chose-address-0x400000)
