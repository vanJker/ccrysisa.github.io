---
title: "你所不知道的 C 语言: bitwise 操作"
subtitle:
date: 2024-02-23T13:13:33+08:00
# draft: true
# author:
#   name:
#   link:
#   email:
#   avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Sysprog
  - C
  - Bitwise
categories:
  - Linux Kernel Internals
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: true
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

> Linux 核心原始程式码存在大量 bit(-wise) operations (简称 bitops)，颇多乍看像是魔法的 C 程式码就是 bitops 的组合。

<!--more-->

- {{< link href="https://hackmd.io/@sysprog/c-bitwise" content="原文地址" external-icon=true >}}

## 复习数值系统

- [x] YouTube: [十进制，十二进制，六十进制从何而来？阿拉伯人成就了文艺复兴？[数学大师]](https://www.youtube.com/watch?v=8J7sAYoG50A) 
- [x] [你所不知道的 C 语言: 数值系统](https://hackmd.io/@sysprog/c-numerics)
- [x] [解读计算机编码](https://hackmd.io/@sysprog/binary-representation)

### 位元组合

一些位元组合表示特定的意义，而不是表示数值，这些组合被称为 **trap representation**

- C11 6.2.6.2 Integer types
> For unsigned integer types other than unsigned char, the bits of the object representation shall be divided into two groups: value bits and padding bits (there need not be any of the latter). If there are N value bits, each bit shall represent a different power of 2 between 1 and 2N−1, so that objects of that type shall be capable of representing values from 0 to 2N−1 using a pure binary representation; this shall be known as the value representation. The values of any padding bits are unspecified.

`uintN_t` 和 `intN_t` 保证没有填充位元 (padding bits)，且 `intN_t` 是二补数编码，所以对这两种类型进行位操作是安全的。

- C99 7.18.1.1 Exact-width integer types
> The typedef name intN_t designates a signed integer type with width N, no padding bits, and a two’s complement representation.

{{< admonition info >}}
有符号整数上也有可能产生陷阱表示法 (trap representation)

补充资讯: [CS:APP Web Aside DATA:TMIN: Writing TMin in C](https://csapp.cs.cmu.edu/3e/waside/waside-tmin.pdf)
{{< /admonition >}}

### 位移运算

位移运算的未定义情况:

**C99 6.5.7 Bitwise shift operators**

- 左移超过变量长度，则运算结果未定义
> If the value of the right operand is negative or is greater than or equal to the width of the promoted left operand, the behavior is undefined.

- 对一个负数进行右移，C 语言规格未定义，作为 implementation-defined，GCC 实作为算术位移 (arithmetic shift)
> If E1 has a signed type and a negative value, the resulting value is implementation-defined.

### Signed & Unsigned

当 Unsigned 和 Signed 混合在同一表达式时，Signed 会被转换成 Unsigned，运算结果可能不符合我们的预期 (这里大赞 Rust，这种情况会编译失败:rofl:)。案例请参考原文，这里举一个比较常见的例子:

```c
int n = 10; 
for (int i = n - 1 ; i - sizeof(char) >= 0; i--)
    printf("i: 0x%x\n",i);
```

这段程式码会导致无限循环，因为条件判断语句 `i - sizeof(char) >= 0` 恒为真 (变量 i 被转换成 Unsigned 了)。

- 6.5.3.4 The sizeof operator
> The value of the result is implementation-defined, and its type (an unsigned integer type) is size_t, defined in <stddef.h> (and other headers).

- 7.17 Common definitions <stddef.h>
> `size_t` which is the unsigned integer type of the result of the sizeof operator

### Sign Extension

将 w bit signed integer 扩展为 w+k bit signed integer，只需将 sign bit 补充至扩展的 bits。

数值等价性推导:
- positive: 显然是正确的，sign bit 为 0，扩展后数值仍等于原数值
- negitive: 将 w bit 情形时的除开 sign bit 的数值设为 U，则原数值为 $2^{-(w-1)} + U$，则扩展为 w+k bit 后数值为 $2^{-(w+k-1)} + 2^{w+k-2} + ... + 2^{-(w-1)} + U$，因为 $2^{-(w+k-1)} + 2^{w+k-2} + ... + 2^{w-1} = 2^{-(w-1)}$，所以数值依然等价。

> $2^{-(w+k-1)} + 2^{w+k-2} + ... + 2^{w-1}$ 可以考虑从左往右的运算，每次都是将原先的数值减半，所以最后的数值为 $2^{-(w+k-1)}$

所以如果 n 是 signed 32-bit，则 `n >> 31` 等价于 `n == 0 ? 0 : -1`。在这个的基础上，请重新阅读 [解读计算机编码](https://hackmd.io/@sysprog/binary-representation) 中的 abs 和 min/max 的常数时间实作。

## Bitwise Operator

- [x] [Bitwise Operators Quiz Answers](http://doc.callmematthi.eu/C_Bitwise_Operators.html)
- [x] [Practice with bit operations](https://pconrad.github.io/old_pconrad_cs16/topics/bitOps/)
- [x] [Bitwise Practice](https://web.stanford.edu/class/archive/cs/cs107/cs107.1202/lab1/practice.html)

> Each lowercase letter is 32 + uppercase equivalent. This means simply flipping the bit at position 5 (counting from least significant bit at position 0) inverts the case of a letter.

> The gdb print command (shortened p) defaults to decimal format. Use `p/format` to instead select other formats such as `x` for hex, `t` for binary, and `c` for char.

```c
// unsigned integer `mine`, `yours`
remove yours from mine                            mine = mine & ~yours
test if mine has both of two lowest bits on       (mine & 0x3) == 0x3
n least significant bits on, all others off       (1 << n) - 1
k most significant bits on, all others off        (~0 << (32 - k)) or
                                                  ~(~0U >> k)

// unsigned integer `x`, `y` (right-shift: arithmetic shift)
x &= (x - 1)                                      clears lowest "on" bit in x
(x ^ y) < 0                                       true if x and y have opposite signs
```

程序语言只提供最小粒度为 Byte 的操作，但是不直接提供 Bit 粒度的操作，这与字节顺序相关。假设提供以 Bit 为粒度的操作，这就需要在编程时考虑 **大端/小端模式**，极其繁琐。

### bitwise & logical

位运算满足交换律，但逻辑运算并不满足交换律，因为短路机制。考虑 Linked list 中的情形:

```c
// list_head *head
if (!head || list_empty(head))
if (list_empty(head) || !head)
```

第二条语句在执行时会报错，因为 `list_empty` 要求传入的参数不为 NULL

### Right Shifts

对于 Unsigned 或 positive sign integer 做右移运算时 `x >> n`，其最终结果值为 $\lfloor x / 2^n \rfloor$。

因为这种情况的右移操作相当于对每个 bit 表示的 power 加上 $-n$，再考虑有些 bit 表示的 power 加上 $-n$ 后会小于 0，此时直接将这些 bit 所表示的值去除即可 (因为在 integer 中 bit 的 power 最小为 0，如果 power 小于 0 表示的是小数值)，这个操作对应于向下取整。

```
    00010111 >> 2  (23 >> 4)
->  000101.11      (5.75)
->  000101         (5)
```

## bitwise 实作

- Vi/Vim 为什么使用 hjkl 作为移动字符?
> 當我們回顧 1967 年 ASCII 的編碼規範，可發現前 32 個字元都是控制碼，讓人們得以透過這些特別字元來控制畫面和相關 I/O，早期鍵盤的 "control" 按鍵就搭配這些特別字元使用。"control" 組合按鍵會將原本字元的第 1 個 bit 進行 XOR，於是 H 字元對應 ASCII 編碼為 100_1000 (過去僅用 7 bit 編碼)，組合 "control" 後 (即 Ctrl+H) 會得到 000_1000，也就是 backspace 的編碼，這也是為何在某些程式中按下 backspace 按鍵會得到 ^H 輸出的原因。相似地，當按下 Ctrl+J 時會得到 000_1010，即 linefeed

{{< admonition >}}
where n is the bit number, and 0 is the least significant bit
{{< /admonition >}}

{{< link href="https://github.com/ccrysisa/LKI/blob/main/c-bitwise" content=Source external-icon=true >}}

### Set a bit

```c
unsigned char a |= (1 << n);
```

### Clear a bit

```c
unsigned char a &= ~(1 << n);
```

### Toggle a bit

```c
unsigned char a ^= (1 << n);
```

### Test a bit

```c
bool a = (val & (1 << n)) > 0;
```

### The right/left most byte

```c
// assuming 16 bit, 2-byte short integer
unsigned short right = val & 0xff;        /* right most (least significant) byte */
unsigned short left  = (val >> 8) & 0xff; /* left  most (most  significant) byte */

// assuming 32 bit, 4-byte int integer
unsigned int right = val & 0xff;        /* right most (least significant) byte */
unsigned int left  = (val >> 24) & 0xff; /* left  most (most  significant) byte */
```

### Sign bit

```c
// assuming 16 bit, 2-byte short integer, two's complement
bool sign = val & 0x8000;

// assuming 32 bit, 4-byte int integer, two's complement
bool sign = val & 0x80000000;
```

### Uses of Bitwise Operations or Why to Study Bits

- Compression
- Set operations
- Encryption

> 最常见的就是位图 (bitmap)，常用于文件系统 (file system)，可以节省空间 (每个元素只用一个 bit 来表示)，可以很方便的进行集合操作 (通过 bitwise operator)。

```
x ^ y = (~x & y) | (x & ~y)
```

## 影像处理

- [x] Stack Overflow: [what (r+1 + (r >> 8)) >> 8 does?](https://stackoverflow.com/questions/30237567/what-r1-r-8-8-does)

在图形引擎中将除法运算 `x / 255` 用位运算 `(x+1 + (x >> 8)) >> 8` 来实作，可以大幅度提升计算效能。

### 案例分析

实作程式码: [RGBAtoBW](https://github.com/charles620016/embedded-summer2015/tree/master/RGBAtoBW)

给定每个 pixel 为 32-bit 的 RGBA 的 bitmap，提升效能的方案:

- 建立表格加速浮点运算
- 减少位运算: 可以使用 pointer 的 offset 取代原本复杂的 bitwise operation

```c
bwPixel = table[rgbPixel & 0x00ffffff] + rgbPixel & 0xff000000;
```

只需对 RGB 部分建立浮点数表，因为 `rgbPixel & 0xff00000` 获取的是 A，无需参与浮点运算。这样建立的表最大下标应为 0x00ffffff，所以这个表占用 $2^{24} Bytes = 16MB$，显然这个表太大了 **not cache friendly**

```c
bw = (uint32_t) mul_299[r] + (uint32_t) mul_587[g] + (uint32_t) mul_144[b];
bwPixel = (a << 24) + (bw << 16) + (bw << 8) + bw;
```

分别对 R, G, B 建立对应的浮点数表，则这三个表总共占用 $3 \times 2^8 Bytes < 32KB$ **cache friendly**

## 案例探讨

{{< admonition info >}}
- [位元旋转实作和 Linux 核心案例](https://hackmd.io/@sysprog/bitwise-rotation)
- [reverse bit 原理和案例分析](https://hackmd.io/@sysprog/bitwise-reverse)
{{< /admonition >}}