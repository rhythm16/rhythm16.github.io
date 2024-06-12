---
title: "把`char`cast成`unsigned int`會sign extend
"
date: 2024-06-12 12:54:15
summary:
tags:
  - C
categories: technical
---

看一下這段code:

```c
#include <stdio.h>

int main() {
	char a = 0xFF;
	unsigned char b = 0xFF;
	printf("%x, %x\n", (unsigned int)a, (unsigned int)b);
	return 0;
}
```

如標題所說結果是:

```c
ffffffff, ff
```

其實不知道大多數人是覺得這樣是預期中的行為還是不是XD 我是有一點點意外就是了

來看一下為什麼:

在(當前)最新的C標準draft中的 6.3.1.3 Signed and unsigned integers 可以看到

> 1 When a value with integer type is converted to another integer type other than bool, if the value can be represented by the new type, it is unchanged.

> 2 Otherwise, if the new type is unsigned, the value is converted by repeatedly adding or subtracting one more than the maximum value that can be represented in the new type until the value is in the range of the new type.

> 3 Otherwise, the new type is signed and the value cannot be represented in it; either the result is implementation-defined or an implementation-defined signal is raised.

第一點很合理，如果目標型別可以表示被轉換的值，那麼型別轉換過去之後值不變。

第二點則是我們的例子適用的情況，但他寫得有點難理解: 如果目標型別(`unsigned int`)沒辦法表示要被轉換的值(0xFF, -1)，則把目標型別能表示的最大值+1，然後用被轉換的值去加或減它，直到能夠被目標型別表示為止。

換句話說，如果被轉換的值是X，目標型別能表示的最大值是Y，則計算

X + (Y + 1)\
X + 2(Y + 1)\
X + 3(Y + 1)\
…

或者

X - (Y + 1)\
X - 2(Y + 1)\
X - 3(Y + 1)\
…

看哪個先能夠被目標型別表示

以我們的例子來說，目標型別(`unsigned int`)能表示的最達值是 2^32 - 1，所以Y = 2^32 - 1，Y + 1就是2^32，接著我們的X是(-1)，直接看出X + (Y + 1) → -1 + (2^32 - 1 + 1) → -1 + 2^32 → 2^32 - 1，即可被`unsigned int`表示(`0xFFFFFFFF`)。

也就是說，`char`轉換成`unsigned int`會sign extend，因為對於所有代表負值的`char`都不能直接被`unsigned int`表示，而進行X + (Y + 1)運算的結果就是2^32 - |X|，其實就是two’s complement X的表示。

有趣的是在6.2.5 Types中:

> 20 The three types char, signed char, and unsigned char are collectively called the character types. The implementation shall define char to have the same range, representation, and behavior as either signed char or unsigned char.

也就是說依照implementation, `char`也可能是`unsigned char`，那麼上述的狀況就不成立，因為0xFF就是255而不是-1了。

順便提一下，`char` 是一個integer type (6.2.5 Types):

> There are five standard signed integer types, designated as `signed char`, `short int`, `int`, `long int`, and `long long int`.

還有，例子中的

```c
char a = 0xFF;
```

其實是個implementation defined的行為，因為從6.4.4.1 Integer constants中的規則來看`0xFF`這個東西會被判斷成`int`，所以值是255，在我們的case中是無法被`char`表示的(-128 \~ 127)，所以轉換適用上面第三點。
