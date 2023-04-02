---
title: ICS01 datalab
date: 2023-02-07 20:16:22
tags: ICS
updated:
type:
comments:
description:
keywords:
top_img:
mathjax:
katex:
aside:
aplayer:
highlight_shrink:
---

# ICS Lab1: Data Lab



## Introduction

这是ICS课程的第一个lab，内容是熟悉位运算并通过位操作实现一些功能。  

对于lab我的看法是不要浪费太多的时间，能讨论就一起讨论着做，能够参考前辈的工作就不用太多独立思考。只要不是CtrlCV，依样画葫芦的收获往往比自己研究来得快。lab的有些部分尤其是CMU移植过来之后加上的在最后一两个PKU魔改的part不是一般人轻易能够搞定的（$dalao$请直接忽略我的话因为$dalao$也不需要参考这种辣鸡文章吧），有时间不如多看CSAPP或者去做往年题。因为我是大二才转到信科学院的（匿名严肃谴责ljl老师招生不守信用事前一套背后一套），所以周围也没有什么要好的同学可以讨论作业，因此完成lab花费了我非常多的时间（尤其是后半学期），但是只是在绞尽脑汁研究奇技淫巧，对于核心内容的掌握没有太大帮助。这也是我决定重新写这一套lab并且分享过程的原因。  

Data Lab是第一个lab，内容也是非常简单尤其是网上有大量的资料可以参考。许多位运算的技巧感觉并不是那么容易想到，所以我觉得只要能够按照解答看明白是在做什么操作就差不多了。在此我的建议是抓紧配置好自己的Linux环境，不要到了后面才遇到问题又要重新配置环境，那就非常头痛了。

## Data Lab

### restrictions
扯了这么多，现在开始看lab的内容。解压文件包之后不必惊慌，只要看`bits.c`文件。框架已经写好了，只需在限定的op数目内实现功能就好了。不要忘了看具体要求——只能使用8位以下的立即数，不能使用条件语句或者循环，只能使用受限的运算符：
$$! \quad ˜ \quad \& \quad ˆ \quad | \quad + \quad << \quad >>$$

## Bit Manip

- minusOne

就是返回-1，考虑直接把每一位都变成1就好。
```C
/* 
 * minusOne - return a value of -1 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 2
 *   Rating: 1
 */
int minusOne(void) {
  return ~0;
}
```

- implication

蕴涵算子, `iff`前真后假时返回0，列出真值表猜表达式。
```C
/* 
 * implication - return x -> y in propositional logic - 0 for false, 1
 * for true
 *   Example: implication(1,1) = 1
 *            implication(1,0) = 0
 *   Legal ops: ! ~ ^ |
 *   Max ops: 5
 *   Rating: 2
 */
int implication(int x, int y) {
  return (!x)|y;
}
```

- leastBitPos(x) 

找到非零的最低位。
```C
/* 
 * leastBitPos - return a mask that marks the position of the
 *               least significant 1 bit. If x == 0, return 0
 *   Example: leastBitPos(96) = 0x20
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 2 
 */
int leastBitPos(int x) {
  return ((x+(~0))^x)&x;
}
```

- rotateLeft(x,n)

把x向高位方向旋转n位。
```C
/* 
 * rotateLeft - Rotate x to the left by n
 *   Can assume that 0 <= n <= 31
 *   Examples: rotateLeft(0x87654321,4) = 0x76543218
 *   Legal ops: ~ & ^ | + << >> !
 *   Max ops: 25
 *   Rating: 3 
 */
int rotateLeft(int x, int n) {
  return (x<<n) | (~((~0) << n) & (x>>(32+1+(~n))));
}
```

- conditional(x,y,z)

返回x ? y : z，考虑!运算可以得到0或者1，左移31位再算数右移回来就可以把每一位都变成相同，这相当于是一个有选择的mask，可以用来做选择。
```C
/* 
 * conditional - same as x ? y : z 
 *   Example: conditional(2,4,5) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
int conditional(int x, int y, int z) {
  return (~(((!x)<<31)>>31) & y) | ((((!x)<<31)>>31) & z);
}
```

- bang(x) 

返回!x，只有x各位全0时才能返回1，否则返回0。考虑寻求全0的特点。
```C
/* 
 * bang - Compute !x without using !
 *   Examples: bang(3) = 0, bang(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
int bang(int x) {
  return 1 & (((~x)&~(~x+1))>>31);
}
```


## Two’s Complement Arithmetic

- oneMoreThan(x,y)

x+1与y异或操作，还要考虑溢出的情况。
```C
/* 
 * oneMoreThan - return 1 if y is one more than x, and 0 otherwise
 *   Examples oneMoreThan(0, 1) = 1, oneMoreThan(-1, 1) = 0
 *   Legal ops: ~ & ! ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int oneMoreThan(int x, int y) {
  return !((x+1) ^ y) & !(!(y ^ (1 << 31)));
}
```

- fitsBits(x,n) 

比较x最后n位的32位扩展的值是不是和x一样即可。
```C
/* 
 * fitsBits - return 1 if x can be represented as an 
 *  n-bit, two's complement integer.
 *   1 <= n <= 32
 *   Examples: fitsBits(5,3) = 0, fitsBits(-4,3) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int fitsBits(int x, int n) {
  return !((x << (32 + ~n + 1) >> (32 + ~n + 1)) ^ x);
}
```

- multFiveEighths(x)

*5就是左移2位再加上自身，然后处理向0舍入的细节。
```C
int multFiveEighths(int x) {
  int y = (x << 2) + x;
  int z = y >> 3;
  int round = !!(0x07 & y);
  int mask = (y >> 31) & 0x1;
  int ans = z + (mask & round);
  return ans;
}
```
- satMul2(x)

判断左移之后有没有变符号来判断溢出。
```C
/*
 * satMul2 - multiplies by 2, saturating to Tmin or Tmax if overflow
 *   Examples: satMul2(0x30000000) = 0x60000000
 *             satMul2(0x40000000) = 0x7FFFFFFF (saturate to TMax)
 *             satMul2(0x60000000) = 0x80000000 (saturate to TMin)
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 3
 */
int satMul2(int x) {
  int y = x << 1;
  int Tmin = 1 << 31;
  int mask = ((Tmin) & (y ^ x)) >> 31;
  int ans1 = y & (~mask);
  int ans2 = mask & (~(((Tmin) & x) >> 31) ^ ((Tmin)));
  return ans1 + ans2;
}
```

- modThree(x) 

这个附加的puzzle写得我非常郁闷，而且回课时lab也没有放出参考的代码。
大致思路就是各个位上的值mod3的结果是会循环的，所以想办法把这些结果叠回到低位上面然后处理负数之类的细节问题。
希望有好方法的dalao可以指点。
```C
/*
 * modThree - calculate x mod 3 without using %.
 *   Example: modThree(12) = 0,
 *            modThree(2147483647) = 1,
 *            modThree(-8) = -2,
 *   Legal ops: ~ ! | ^ & << >> +
 *   Max ops: 60
 *   Rating: 4
 */
int modThree(int x) {
  int mask,y,z,a,b,c,tmp;
    mask = (x >> 31);
    x = (x^(x>>31))+((x>>31)&1);
    //cout<<"absx = "<<x<<endl;
    tmp = (0xFF << 8) + 0xFF;
    y = ((x >> 16) & tmp) + (x & tmp);
    y = (((1 << 16) & y) >> 16) + (y & tmp);
    z = ((y >> 8) & (0XFF)) + (y & (0XFF));
    z = (((1 << 8) & z) >> 8) + (z & (0xFF));
    a = ((z >> 4) & (0Xf)) + (z & (0Xf));
    a = (((1 << 4) & a) >> 4) + (a & (0xf));
    //cout<<"a = "<<a <<' '<<a % 3 << endl;
    b = ((a >> 2) & (0X3)) + (a & (0X3));
    b = (((1 << 2) & b) >> 2) + (b & (0x3));
    //cout<<"b = "<<b <<' '<<b % 3 << endl;
    c = b & (b>>1);
    c = c + (c << 1);
    b = b + ~c + 1;
    //cout<<"b = "<<b<<endl;
    //cout<<"mask = "<<mask<<endl;
    //cout<<(~mask)<<endl;
    //cout<<xx%3<<' '<<ans<<endl;
  return (mask & (~b + 1)) + ((~mask) & (b));
}
```

- ilog2(x) 

类似于试商
```C
/*
 * ilog2 - return floor(log base 2 of x), where x > 0
 *   Example: ilog2(16) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 90
 *   Rating: 4
 */
int ilog2(int x) {
  int a,b,c,d,e;
  a = !(!(x>>16)) << 4;
  x = x >> a;
  b = !(!(x>>8)) << 3;
  x = x >> b;
  c = !(!(x >> 4)) << 2;
  x = x >> c;
  d = !(!(x >> 2)) << 1;
  x = x >> d;
  e = !(!(x >> 1));
  return a + b + c + d + e;
}
```

## Floating-Point Operations

这部分内容有关浮点数运算，熟悉浮点数表示就不太难，也没什么好说明的直接上码吧。。。。。。

- float\_abs(x)
```C
/* 
 * float_abs - Return bit-level equivalent of absolute value of f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representations of
 *   single-precision floating point values.
 *   When argument is NaN, return argument..
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 10
 *   Rating: 2
 */
unsigned float_abs(unsigned uf) {
  int ans = uf & 0x7FFFFFFF;
  if(ans > 0x7F800000)     //NaN
    return uf;
  return ans;
}
```

- float\_i2f(x)
```C
/* 
 * float_i2f - Return bit-level equivalent of expression (float) x
 *   Result is returned as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point values.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_i2f(int x) {
  unsigned sign = x & (1 << 31);
  unsigned exp = 0;
  unsigned frac = 0;
  unsigned round = 0;
  unsigned absX = sign ? (~x + 1) : x;
  unsigned tmp = absX;
  while ((tmp = tmp >> 1))
      ++exp;
  frac = absX << (31 - exp) << 1;
  round = frac << 23 >> 23;
  frac = frac >> 9;
  if (round > 0xFF + 1) round = 1;
  else if (round < 0xFF + 1) round = 0;
  else round = frac & 1;
  return x ? (sign | ((exp + 0x7F) << 23) | frac) + round : 0;
}
```

- float\_f2i(x)
```C
/* 
 * float_f2i - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
int float_f2i(unsigned uf) {
  int x,y,ans1;
  x=(uf>>23)&0xFF;
  y=(uf & 0x007FFFFF)|(1 << 23);
  if(x>157)
    return 0x80000000u;
  if(x<127)
    return 0;
  if(x > 150)
    ans1 = y<<(x-150);
  else
    ans1 = y>>(150-x);
  if(((uf>>31)&0x1)==1)
    return ~ans1 + 1;
  else
    return ans1;
}
```

- float\_negpwr2(x)
```C
/* 
 * float_negpwr2 - Return bit-level equivalent of the expression 2.0^-x
 *   (2.0 raised to the power -x) for any 32-bit integer x.
 *
 *   The unsigned value that is returned should have the identical bit
 *   representation as the single-precision floating-point number 2.0^-x.
 *   If the result is too small to be represented as a denorm, return
 *   0. If too large, return +INF.
 * 
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
 *   Max ops: 30 
 *   Rating: 4
 */
unsigned float_negpwr2(int x) {
  unsigned INF = 0xFF << 23; // 阶码全1
  if (x > 149)
    return 0;
  if (x < ~(0x7F)+1)
    return INF;
  if(x <= 149 && x > 126)
    return 1<<(149+~x+1);
  if(x <= 126 && x >= ~(0x7F)+1)
    return (~x+1+127)<<23;
  return 2;
}
```

## Conclusion
`Datalab`到这里就完成了，每一年的题目都会有一些细微的改变，但无论如何这都是最简单的lab了（因为网上有太多的解答可以参考）。但是我一直不明白明明去年也是modthree却没有一个人乐意写一篇解题报告分享出来，也只能感叹现在的PKU越来越内卷，学生是真的没有时间造福后人了，当然也有可能是功利主义越来越严重吧。

另外，我在完成`Datalab`时参考了许多前人的文章，他们中有很多细节都给了我启示，也有一些代码copy之后无法正确运行。由于是快一年之后才想到要写这玩意儿，所以也没法给出具体的参考文献以及勘误。反正若有任何雷同，都算我剽窃吧。
