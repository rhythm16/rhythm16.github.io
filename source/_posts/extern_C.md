---
title: extern "C" 如何使系統函式庫兼容C與C++
date: 2020-08-23 15:55:15
summary:
tags:
  - C
categories: technical
---

## 原始的問題
故事是這樣的：
如果我們有個C函式庫，裡面有函數例如：
```clike
/* cfunc.c */
 
void awesome_C_function()
{
    /* does nothing */
}
```
你可能會想說：C函式庫，C++程式應該也可以用吧
```cpp
/* main.cpp */

#include <iostream>

void awesome_C_function();

int main()
{
    awesome_C_function();
    return 0;
}
```
那編譯看看：
```bash
$ gcc -c cfunc.c
$ g++ -c main.cpp
$ g++ main.o cfunc.o -o a.out
main.o: In function `main`:
main.cpp:(.text+0x5): undefined reference to `awesome_C_function()`
collect2: error: ld returned 1 exit status
```
問號？？為什麼會這樣勒
原因是g\++在編譯C++程式時，會對符號(變數與函式名稱等等)進行符號修飾([name mangling](https://en.wikipedia.org/wiki/Name_mangling))，導致鏈結器在鏈結時找不到對應的符號名稱，對應到最後一行`collect2: error: ld returned 1 exit status`

如果使用`readelf`工具檢視各個編譯出的.o檔的符號表，就可以看出端倪：(注意不同版本之編譯器、作業系統結果可能不完全一樣)
main.o:
```bash
$ readelf -s main.o #-s表示顯示符號表

Symbol table `.symtab' contains 21 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
...
    16: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z18awesome_C_functionv
...
```
cfunc.o:
```bash
$ readelf -s cfunc.o

Symbol table `.symtab' contains 21 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
...
     8: 0000000000000000     0 FUNC    GLOBAL DEFAULT    1 awesome_C_function
...
```
雖然我們source code都是寫`awesome_C_function()`，因為C++有符號修飾，變成了兩個不同名字的符號，產生錯誤。

## extern "C" 如何解決問題
好在，C++有`extern "C"`語法，可以定義C符號，使其不做符號修飾，若修改main.cpp成
```cpp
/* main.cpp */

#include <iostream>

extern "C" {
void awesome_C_function();
}

int main()
{
    awesome_C_function();
    return 0;
}
```
則程式可正常編譯，執行：
```bash
$ g++ -c main.cpp
$ g++ main.o cfunc.o -o a.out
$ ./a.out
$
```
太好了，問題乍看之下解決了，不過如果我們想把函式庫標頭另外放在一個檔案，供人`#include`，會出現C不支援`extern "C"`的問題，也就是說如果我們這樣寫awesome.h:
```cpp
/* awesome.h */
extern "C" {
void awesome_C_function();
}
```
main.cpp:
```cpp
/* main.cpp */

#include <iostream>
#include "awesome.h"

int main()
{
    awesome_C_function();
    return 0;
}
```
上面的main.cpp使用g++編譯完全沒有錯誤，但如果今天想寫C，並使用`awesome.h`函式庫的話：
```clike
/* main.c */

#include "awesome.h"

int main()
{
    awesome_C_function();
    return 0;
}
```
使用gcc編譯會直接報錯，錯誤內容就不貼出了，反正gcc在抱怨他根本看不懂include進來的extern "C"語法啊，那是C++專屬的。

難道一個編譯好的C函式庫，要配套兩種標頭檔嗎，一個給C用，一個給C\++用？
答案是不需要，使用編譯前的預處理器可以幫我們解決問題！
在編譯C\++程式時，C++的編譯器會定義"__cplusplus"這個macro，所以只要把awesome.h寫成這樣就解決問題了。
```cpp
/* awesome.h */

#ifdef __cplusplus
extern "C" {
#endif

void awesome_C_function();

#ifdef __cplusplus
}
#endif
```

如此一來，無論是C還是C++程式，都可以`#include "awesome.h"`了，預處理器會自動的依情況展開，編譯、鏈結都會成功。

這樣的技巧再幾乎所有的系統標頭檔都有用到。
本文內容來自
**俞甲子、石凡、潘愛民 (2009). ⟪程序員的自我修養⟫**
是我自己閱讀消化之後所做之筆記。
