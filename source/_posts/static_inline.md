---
title: 幹嘛static inline?
date: 2020-09-12 21:20:15
summary:
tags:
  - C
categories: technical
---

> 2020/11/25更新：\
這篇對一半吧，\
[這篇](https://read01.com/zh-tw/3APOEG.html#.X75Tay2l1QJ)比較完整，也比較正確

忘記在哪時候看到[這篇(我有所不知的 static inline function)](https://medium.com/@hauyang/我有所不知的-static-inline-b363892b7450)，讀完覺得頗有趣的，來記錄一下。
這篇是靠感覺打的，沒有實作跟去check組合語言，因為我覺得觀念對了，真的有不確定的地方再試就好。

## static
`static`代表符號只在這個編譯單元有效，編譯單元就是一個.c檔在預處理器處理之後的東東(include, define那些被展開之後)。

## inline
`inline`則是建議編譯器可以把函數內嵌在呼叫的地方，節省資源提升速度。

## 那幹嘛要寫在一起勒
其實可以反過來想，如果不一起寫，只寫`inline`，會怎樣？
好，現在編譯器看到一個`inline`函數，它會想：
> 哦哦寫code的人建議`inline`，ㄟ但他不是`static`耶，這樣行不通啊，如果有其他編譯單元宣告了這個函數，想要鏈接怎麼辦？
> 哎沒辦法，只好放棄`inline`了，把這個函數獨立編譯吧

上面那樣寫可能看不懂，來想想一個狀況：
```clike
/* foo.c */
inline void foo()
{
    /* does something */
}
```
```clike
/* main.c */
inline void foo();

int main()
{
    foo();
    return 0;
}
```
因為`foo()`沒有`static`，main.c是可以宣告並使用foo.c的`foo()`的，而且可以獨立編譯main.c和foo.c：
```clike
$ gcc -c main.c -o main.o
$ gcc -c foo.c -o foo.o
$ gcc main.o foo.o -o a.out
```
在獨立編譯main.c時，根本就沒有`foo()`的定義啊，所以也就不可能可以內嵌。

不過，如果是加了`static`，那麼就不會出現其他編譯單元宣告foo.c的`foo()`並獨自編譯的狀況，也就能順利內嵌函數了。
