---
title: "Casting a `char` to an `unsigned int` sign extends"
date: 2024-06-17 18:54:15
summary:
tags:
  - C
categories: technical
---

Let’s look at this snippet of code:

```c
#include <stdio.h>

int main() {
	char a = 0xFF;
	unsigned char b = 0xFF;
	printf("%x, %x\n", (unsigned int)a, (unsigned int)b);
	return 0;
}
```

As the title indicates, the result of the program is:

```plain
ffffffff, ff
```

I am actually unsure whether people would find this result unexpected or not, if anything, I am a bit surprised :)

Let’s see why:

Section 6.3.1.3 *Signed and unsigned integers* of the latest C standard draft shows

> 1 When a value with integer type is converted to another integer type other than bool, if the value can be represented by the new type, it is unchanged.

> 2 Otherwise, if the new type is unsigned, the value is converted by repeatedly adding or subtracting one more than the maximum value that can be represented in the new type until the value is in the range of the new type.

> 3 Otherwise, the new type is signed and the value cannot be represented in it; either the result is implementation-defined or an implementation-defined signal is raised.

The first part is self-explanatory. The second part is the one which applies to the example above, but the wording is a little convoluted. Simply put, if the type to be cast to (`unsigned int`) cannot represent the value being cast (0xFF, -1), then take the largest value which can be represented by the type to be cast to, add one to it, then use the result to add or subtract the value to be cast, until the new type can represent it.

In other words, if the value to be cast is X, and the maximum value that can be represented by the new type is Y, then calculate

X + (Y + 1)\
X + 2(Y + 1)\
X + 3(Y + 1)\
…

or

X - (Y + 1)\
X - 2(Y + 1)\
X - 3(Y + 1)\
…

and see which one can be represented by the type to be cast to first.

For our example, the maximum value of the type being cast to (`unsigned int`) is 2^32 - 1, so Y = 2^32 - 1, and Y + 1 is 2^32. Next, directly observe that X + (Y + 1) → -1 + (2^32 - 1 + 1) → -1 + 2^32 → 2^32 - 1 is a value representable by `unsigned int` (`0xFFFFFFFF`).

Therefore, casting a `char` to an `unsigned int` sign extends, since `unsigned int` is unable to represent negative values, and X + (Y + 1) is 2^32 - |X| for the negative value, which is the two’s complement representation of X.

Interestingly in 6.2.5 *Types*:

> 20 The three types char, signed char, and unsigned char are collectively called the character types. The implementation shall define char to have the same range, representation, and behavior as either signed char or unsigned char.

`char` can actually be `unsigned char` depending on the implementation, in such cases the argument does not stand, because 0xFF would be 255 and not -1.

Additionally, `char` is an integer type (6.2.5 *Types*):

> There are five standard signed integer types, designated as `signed char`, `short int`, `int`, `long int`, and `long long int`.

also, this in the example

```c
char a = 0xFF;
```

is actually implementation defined, from 6.4.4.1 *integer constants*, `0xFF` is categorized as an `int` , so the value would be 255, which cannot be represented by `char` (-128 \~ 127), and the third rule above applies to this cast/conversion.
