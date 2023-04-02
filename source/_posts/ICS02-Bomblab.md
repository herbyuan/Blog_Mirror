---
title: ICS02 Bomblab
date: 2023-02-07 21:02:51
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


Bomblab是第二个lab，简单来说任务就是通过阅读反汇编得到的代码和GDB调试，输入正确的字符串以过关，错误则可能会引爆炸弹。

看着好像很恐怖，引爆一次炸弹就要扣除最终一定的分数，但其实丝毫不用担心。一方面虽然lab本来是做过处理只能在服务器上运行的，但是只要找到合适的入口你就可以跳过服务器检测从而在本地拆弹，另一方面只要会设置断点，每次都在explode_bomb（引爆炸弹函数）处设断点，该收手时就收手，根本不用担心炸弹爆炸。但是我不推荐第一种方式，因为要是你有这种绕过服务器检测函数的水平，这个lab对你来说应该是没有难度的，正常做就好了。

有一点十分遗憾的是，在我写这个解析的时候服务器已经关掉了，而且我也并不会修改HEX使得炸弹在本地运行。于是我只能分析拆弹的流程，不能给出具体运行的截图，在此说一声抱歉。

很多网上的解析上来就说啊好你看这六个阶段每个阶段怎么怎么做，隐藏炸弹怎么怎么进。但是我在做lab时每次最先遇到的问题都不是题目本身，而是怎么理解接近十页的writeup，怎么开始。Bomblab最基本的工具就是GDB和SSH，但是大部分的解析几乎都略过了这一部分，所以我当时甚至要花很长时间才能搞明白怎么得到反汇编的文件，唉。

下载得到自己编号的lab之后，使用objdump生成汇编文件，使用命令$objdump -d bomb > bomb.asm$，将bomb可执行文件反汇编并写入bomb.asm文件，可以看到每个phase对应一个函数，我们的目的就是通过阅读汇编代码研究怎样的输入可以不调用函数explode_bomb而正常退出。

第二个重要的工具是SSH，因为程序需要在远程服务器上运行。具体的登录方式下发的文件里面应该有，稍微学习一下即可。

关于gdb工具的使用，个人感觉不用知道太多，会设置断点，看寄存器内容就差不多了。其他指令可以在说明文档中找到，但是不用也没有关系。常用指令如下：

- `gdb bomb` 用gdb打开bomb
- `r` 运行可执行文件
- `b func` 在函数func开始处设置断点
- `b *0xFFFF` 在地址0xFFFF处设置断点
- `si` 从断点处逐句执行
- `x/s *0xFFFF` 以字符串形式输出给定地址存放的值
- `info r` 展示所有寄存器的信息

最后也是最重要的一点——`Ctrl+C`。任何时候感觉不对，直接终止gdb进程，或者也可以输入`kill`指令终止运行中的bomb程序。

## phase 1
```
000000000000130d <phase_1>:
    130d: 48 83 ec 08           sub    $0x8,%rsp
    1311: 48 8d 35 f8 19 00 00  lea    0x19f8(%rip),%rsi        # 2d10 <_IO_stdin_used+0x190>
    1318: e8 92 04 00 00        callq  17af <strings_not_equal>
    131d: 85 c0                 test   %eax,%eax
    131f: 75 05                 jne    1326 <phase_1+0x19>
    1321: 48 83 c4 08           add    $0x8,%rsp
    1325: c3                    retq   
    1326: e8 a1 07 00 00        callq  1acc <explode_bomb>
    132b: eb f4                 jmp    1321 <phase_1+0x14>
```

这是一个热身环节。看到函数strings_not_equal，且之后不等于0就要跳转到explode_bomb，得知这肯定是要输入一个字符串与某个存在的字符串匹配，这个字符串的地址应该存放在0x19f8(%rip)中，但是%rip要在运行中才能知道。

这是一个热身环节。看到函数strings_not_equal，且之后不等于0就要跳转到explode_bomb，得知这肯定是要输入一个字符串与某个存在的字符串匹配，这个字符串的地址应该存放在0x19f8(%rip)中，但是%rip要在运行中才能知道。

## phase 2
```
000000000000132d <phase_2>:
    132d: 53                    push   %rbx
    132e: 48 83 ec 20           sub    $0x20,%rsp
    1332: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
    1339: 00 00 
    133b: 48 89 44 24 18        mov    %rax,0x18(%rsp)
    1340: 31 c0                 xor    %eax,%eax
    1342: 48 89 e6              mov    %rsp,%rsi
    1345: e8 02 08 00 00        callq  1b4c <read_six_numbers>
    134a: 83 3c 24 01           cmpl   $0x1,(%rsp)
    134e: 75 07                 jne    1357 <phase_2+0x2a>
    1350: bb 01 00 00 00        mov    $0x1,%ebx
    1355: eb 0a                 jmp    1361 <phase_2+0x34>
    1357: e8 70 07 00 00        callq  1acc <explode_bomb>
    135c: eb f2                 jmp    1350 <phase_2+0x23>
    135e: 83 c3 01              add    $0x1,%ebx
    1361: 83 fb 05              cmp    $0x5,%ebx
    1364: 7f 19                 jg     137f <phase_2+0x52>
    1366: 48 63 d3              movslq %ebx,%rdx
    1369: 8d 43 ff              lea    -0x1(%rbx),%eax
    136c: 48 98                 cltq   
    136e: 8b 04 84              mov    (%rsp,%rax,4),%eax
    1371: 01 c0                 add    %eax,%eax
    1373: 39 04 94              cmp    %eax,(%rsp,%rdx,4)
    1376: 74 e6                 je     135e <phase_2+0x31>
    1378: e8 4f 07 00 00        callq  1acc <explode_bomb>
    137d: eb df                 jmp    135e <phase_2+0x31>
    137f: 48 8b 44 24 18        mov    0x18(%rsp),%rax
    1384: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
    138b: 00 00 
    138d: 75 06                 jne    1395 <phase_2+0x68>
    138f: 48 83 c4 20           add    $0x20,%rsp
    1393: 5b                    pop    %rbx
    1394: c3                    retq   
    1395: e8 46 fb ff ff        callq  ee0 <__stack_chk_fail@plt>
```

提示非常明显了，有个函数叫做read_six_numbers，所以输入肯定是六个数，阅读具体代码可知如果读入的不是六个数就会引爆炸弹。

正常输入下应该进入到$1350$这一行，注意这里ebx寄存器中放入了$$0x1$。ebx在小于等于5时执行。

$1369$将rbx-1放入eax作为偏移，之后$136e$将栈中对应元素放入eax，然后自己加自己。之后比较这个数与之后的那个数是否相等。到此我们就知道了这个数组要求后一个数是前一个数的两倍，所以构造一个等比数列就能满足要求了。

所以答案是 `1 2 4 8 16 32`.

## phase 3
```
000000000000139a <phase_3>:
    139a: 48 83 ec 18           sub    $0x18,%rsp
    139e: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
    13a5: 00 00 
    13a7: 48 89 44 24 08        mov    %rax,0x8(%rsp)
    13ac: 31 c0                 xor    %eax,%eax
    13ae: 48 8d 4c 24 04        lea    0x4(%rsp),%rcx
    13b3: 48 89 e2              mov    %rsp,%rdx
    13b6: 48 8d 35 eb 1c 00 00  lea    0x1ceb(%rip),%rsi        # 30a8 <array.3419+0x308>
    13bd: e8 ce fb ff ff        callq  f90 <__isoc99_sscanf@plt>
    13c2: 83 f8 01              cmp    $0x1,%eax
    13c5: 7e 1d                 jle    13e4 <phase_3+0x4a>
    13c7: 83 3c 24 07           cmpl   $0x7,(%rsp)
    13cb: 0f 87 99 00 00 00     ja     146a <phase_3+0xd0>
    13d1: 8b 04 24              mov    (%rsp),%eax
    13d4: 48 8d 15 a5 19 00 00  lea    0x19a5(%rip),%rdx        # 2d80 <_IO_stdin_used+0x200>
    13db: 48 63 04 82           movslq (%rdx,%rax,4),%rax
    13df: 48 01 d0              add    %rdx,%rax
    13e2: ff e0                 jmpq   *%rax
    13e4: e8 e3 06 00 00        callq  1acc <explode_bomb>
    13e9: eb dc                 jmp    13c7 <phase_3+0x2d>
    13eb: b8 00 00 00 00        mov    $0x0,%eax
    13f0: eb 05                 jmp    13f7 <phase_3+0x5d>
    13f2: b8 b3 02 00 00        mov    $0x2b3,%eax
    13f7: 2d e5 03 00 00        sub    $0x3e5,%eax
    13fc: 05 a4 02 00 00        add    $0x2a4,%eax
    1401: 2d a1 02 00 00        sub    $0x2a1,%eax
    1406: 05 a1 02 00 00        add    $0x2a1,%eax
    140b: 2d a1 02 00 00        sub    $0x2a1,%eax
    1410: 05 a1 02 00 00        add    $0x2a1,%eax
    1415: 2d a1 02 00 00        sub    $0x2a1,%eax
    141a: 83 3c 24 05           cmpl   $0x5,(%rsp)
    141e: 7f 06                 jg     1426 <phase_3+0x8c>
    1420: 39 44 24 04           cmp    %eax,0x4(%rsp)
    1424: 74 05                 je     142b <phase_3+0x91>
    1426: e8 a1 06 00 00        callq  1acc <explode_bomb>
    142b: 48 8b 44 24 08        mov    0x8(%rsp),%rax
    1430: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
    1437: 00 00 
    1439: 75 3b                 jne    1476 <phase_3+0xdc>
    143b: 48 83 c4 18           add    $0x18,%rsp
    143f: c3                    retq   
    1440: b8 00 00 00 00        mov    $0x0,%eax
    1445: eb b5                 jmp    13fc <phase_3+0x62> 
    1447: b8 00 00 00 00        mov    $0x0,%eax
    144c: eb b3                 jmp    1401 <phase_3+0x67>
    144e: b8 00 00 00 00        mov    $0x0,%eax
    1453: eb b1                 jmp    1406 <phase_3+0x6c>
    1455: b8 00 00 00 00        mov    $0x0,%eax        // 3 ?
    145a: eb af                 jmp    140b <phase_3+0x71>
    145c: b8 00 00 00 00        mov    $0x0,%eax
    1461: eb ad                 jmp    1410 <phase_3+0x76>
    1463: b8 00 00 00 00        mov    $0x0,%eax
    1468: eb ab                 jmp    1415 <phase_3+0x7b>
    146a: e8 5d 06 00 00        callq  1acc <explode_bomb>
    146f: b8 00 00 00 00        mov    $0x0,%eax
    1474: eb a4                 jmp    141a <phase_3+0x80>
    1476: e8 65 fa ff ff        callq  ee0 <__stack_chk_fail@plt>
```
看到这里应该有很多人都发现了前面四行几乎在每个phase都有出现，这部分的作用是先开一个栈内的空间，然后放入金丝雀值，在函数结束的时候检验金丝雀值是否被修改，如果修改了说明栈爆了，于是$stack_chk_fail$。这部分内容在attacklab中会有更加详细的应用，而且这个值也不能保证栈没有溢出，毕竟有很多方法可以绕过这玩意。

这里的读取函数是scanf，参数放在0x1ceb(%rip)，运行到这里时用$x/s$命令查看这个字符串，发现是"%d %d"，说明要输入的是两个整数。这两个数放在栈中，分别是(%rsp)和0x4(%rsp)。$13c7$表示如果7小于第一个输入的数，就会引爆炸弹，所以第一个数小于等于7。之后这个数又作为地址的偏移，决定了将要执行哪个位置的代码。理论上程序应该改执行$13eb$位置的代码，之后是对eax寄存器的一顿操作。如果5小于第一个输入的数就要引爆炸弹，这样就确定了第一个数只能是5或者6或者7。第二个数需要和eax中的数相同。到此这个炸弹就可以拆了。尝试第一个数为5或者6或者7，看看能否跳转到正常的对于eax的操作过程，然后单步执行到$1420$位置，查看eax的值（这样就不用人工计算了）。退出程序后重新开始一次，输入正确的值即可拆弹。

最终的答案是 `5 -673`.

## phase 4
```
000000000000147b <func4>:
    147b: 48 83 ec 08           sub    $0x8,%rsp
    147f: 89 d1                 mov    %edx,%ecx
    1481: 29 f1                 sub    %esi,%ecx
    1483: 89 c8                 mov    %ecx,%eax
    1485: c1 e8 1f              shr    $0x1f,%eax
    1488: 01 c8                 add    %ecx,%eax
    148a: d1 f8                 sar    %eax
    148c: 01 f0                 add    %esi,%eax
    148e: 39 f8                 cmp    %edi,%eax
    1490: 7f 0e                 jg     14a0 <func4+0x25>
    1492: 39 f8                 cmp    %edi,%eax
    1494: 7c 16                 jl     14ac <func4+0x31>
    1496: b8 00 00 00 00        mov    $0x0,%eax
    149b: 48 83 c4 08           add    $0x8,%rsp
    149f: c3                    retq   
    14a0: 8d 50 ff              lea    -0x1(%rax),%edx
    14a3: e8 d3 ff ff ff        callq  147b <func4>
    14a8: 01 c0                 add    %eax,%eax
    14aa: eb ef                 jmp    149b <func4+0x20>
    14ac: 8d 70 01              lea    0x1(%rax),%esi
    14af: e8 c7 ff ff ff        callq  147b <func4>
    14b4: 8d 44 00 01           lea    0x1(%rax,%rax,1),%eax
    14b8: eb e1                 jmp    149b <func4+0x20>

00000000000014ba <phase_4>:
    14ba: 48 83 ec 18           sub    $0x18,%rsp
    14be: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
    14c5: 00 00 
    14c7: 48 89 44 24 08        mov    %rax,0x8(%rsp)
    14cc: 31 c0                 xor    %eax,%eax
    14ce: 48 8d 4c 24 04        lea    0x4(%rsp),%rcx
    14d3: 48 89 e2              mov    %rsp,%rdx
    14d6: 48 8d 35 cb 1b 00 00  lea    0x1bcb(%rip),%rsi        # 30a8 <array.3419+0x308>
    14dd: e8 ae fa ff ff        callq  f90 <__isoc99_sscanf@plt>
    14e2: 83 f8 02              cmp    $0x2,%eax
    14e5: 75 0c                 jne    14f3 <phase_4+0x39>
    14e7: 8b 04 24              mov    (%rsp),%eax
    14ea: 85 c0                 test   %eax,%eax
    14ec: 78 05                 js     14f3 <phase_4+0x39>
    14ee: 83 f8 0e              cmp    $0xe,%eax
    14f1: 7e 05                 jle    14f8 <phase_4+0x3e>
    14f3: e8 d4 05 00 00        callq  1acc <explode_bomb>
    14f8: ba 0e 00 00 00        mov    $0xe,%edx
    14fd: be 00 00 00 00        mov    $0x0,%esi
    1502: 8b 3c 24              mov    (%rsp),%edi
    1505: e8 71 ff ff ff        callq  147b <func4>
    150a: 83 f8 05              cmp    $0x5,%eax
    150d: 75 07                 jne    1516 <phase_4+0x5c>
    150f: 83 7c 24 04 05        cmpl   $0x5,0x4(%rsp)
    1514: 74 05                 je     151b <phase_4+0x61>
    1516: e8 b1 05 00 00        callq  1acc <explode_bomb>
    151b: 48 8b 44 24 08        mov    0x8(%rsp),%rax
    1520: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
    1527: 00 00 
    1529: 75 05                 jne    1530 <phase_4+0x76>
    152b: 48 83 c4 18           add    $0x18,%rsp
    152f: c3                    retq   
    1530: e8 ab f9 ff ff        callq  ee0 <__stack_chk_fail@plt>
```
跟上一个phase一样，查看字符串发现输入应该是两个整数（记为x和y），分别放在(%rsp)和0x4(%rsp)中。

第一个数放到eax中，等于0引爆炸弹，大于0xe引爆炸弹。edx=0xe，esi=0，edi=x，然后调用函数$func4$。注意函数传参的顺序为rdi，rsi，rdx，rcx。函数的返回值不等于5就会引爆炸弹。第二个数不等于5就会引爆炸弹。

接下来看$func4$，这是一个递归调用的过程。可以转换成`C`来写一遍。
```C
#include <stdio.h>
int edi = 0;
int esi = 0;
int edx = 0;
int ecx = 0;    
int eax;
int func4()
{
    ecx = edx;
    ecx -= esi;
    eax = ecx;
    eax >>= 0x1f;
    eax += ecx;
    eax >>= 1;
    eax += esi;
    if (eax > edi)
    {
        edx = eax - 1;
        func4();
        eax += eax;
        return eax;
    }
    else if (eax < edi)
    {
        esi = eax + 1;
        func4();
        eax = eax + eax * 1 + 1;
        return eax;
    }
    else
    {
        eax = 0;
        return eax;
    }
}
int main()
{
    for (int i = 0; i < 0xe; ++i)
    {
        edi = i;
        esi = 0;
        edx = 0xe;
        printf("%d\t%d\n", i, func4());
    }
    return 0;
}
```

尝试第一个输入数的值使得func4返回值为5，得到i=10时成立。

答案是`10 5`.

## phase 5
```
0000000000001535 <phase_5>:
    1535: 53                    push   %rbx
    1536: 48 89 fb              mov    %rdi,%rbx
    1539: e8 5c 02 00 00        callq  179a <string_length>
    153e: 83 f8 06              cmp    $0x6,%eax
    1541: 75 0c                 jne    154f <phase_5+0x1a>
    1543: b9 00 00 00 00        mov    $0x0,%ecx
    1548: b8 00 00 00 00        mov    $0x0,%eax
    154d: eb 1e                 jmp    156d <phase_5+0x38>
    154f: e8 78 05 00 00        callq  1acc <explode_bomb>
    1554: eb ed                 jmp    1543 <phase_5+0xe>
    1556: 48 63 d0              movslq %eax,%rdx
    1559: 0f b6 14 13           movzbl (%rbx,%rdx,1),%edx
    155d: 83 e2 0f              and    $0xf,%edx
    1560: 48 8d 35 39 18 00 00  lea    0x1839(%rip),%rsi        # 2da0 <array.3419>
    1567: 03 0c 96              add    (%rsi,%rdx,4),%ecx
    156a: 83 c0 01              add    $0x1,%eax
    156d: 83 f8 05              cmp    $0x5,%eax
    1570: 7e e4                 jle    1556 <phase_5+0x21>
    1572: 83 f9 43              cmp    $0x43,%ecx
    1575: 74 05                 je     157c <phase_5+0x47>
    1577: e8 50 05 00 00        callq  1acc <explode_bomb>
    157c: 5b                    pop    %rbx
    157d: c3                    retq  
```
字符串的长度为6。

存储在`*(%rbx)`开始处。

eax=0，ecx=0。eax<=5时进循环

```C
rdx=eax
edx=*(rbx+rdx)
edx=edx & 0xf
rsi=0x1839(%rip)
ecx+=*(%rsi+4*%rdx)
eax+=1
```

说明输入的字符串作为偏移的参数，使得最终ecx需要等于0x43。中间的有关\%rip的数组需要运行起来才知道是什么，最后反正要把和凑成ecx=0x43。

对照数组找到合适的位置参数，然后转换成ASCII码对应的字符就可以，或者直接用特殊方法输入对应的ASCII码。答案并不唯一，输入 `SPEEEE` 即可.

## phase 6
```
000000000000157e <phase_6>:
    157e: 41 54                 push   %r12
    1580: 55                    push   %rbp
    1581: 53                    push   %rbx
    1582: 48 83 ec 60           sub    $0x60,%rsp
    1586: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
    158d: 00 00 
    158f: 48 89 44 24 58        mov    %rax,0x58(%rsp)
    1594: 31 c0                 xor    %eax,%eax
    1596: 48 89 e6              mov    %rsp,%rsi
    1599: e8 ae 05 00 00        callq  1b4c <read_six_numbers>
    159e: bd 00 00 00 00        mov    $0x0,%ebp
    15a3: eb 27                 jmp    15cc <phase_6+0x4e>
    15a5: e8 22 05 00 00        callq  1acc <explode_bomb>
    15aa: eb 33                 jmp    15df <phase_6+0x61>
    15ac: 83 c3 01              add    $0x1,%ebx
    15af: 83 fb 05              cmp    $0x5,%ebx
    15b2: 7f 15                 jg     15c9 <phase_6+0x4b>
    15b4: 48 63 c5              movslq %ebp,%rax
    15b7: 48 63 d3              movslq %ebx,%rdx
    15ba: 8b 3c 94              mov    (%rsp,%rdx,4),%edi
    15bd: 39 3c 84              cmp    %edi,(%rsp,%rax,4)
    15c0: 75 ea                 jne    15ac <phase_6+0x2e>
    15c2: e8 05 05 00 00        callq  1acc <explode_bomb>
    15c7: eb e3                 jmp    15ac <phase_6+0x2e>
    15c9: 44 89 e5              mov    %r12d,%ebp
    15cc: 83 fd 05              cmp    $0x5,%ebp
    15cf: 7f 17                 jg     15e8 <phase_6+0x6a>
    15d1: 48 63 c5              movslq %ebp,%rax
    15d4: 8b 04 84              mov    (%rsp,%rax,4),%eax
    15d7: 83 e8 01              sub    $0x1,%eax
    15da: 83 f8 05              cmp    $0x5,%eax
    15dd: 77 c6                 ja     15a5 <phase_6+0x27>
    15df: 44 8d 65 01           lea    0x1(%rbp),%r12d
    15e3: 44 89 e3              mov    %r12d,%ebx
    15e6: eb c7                 jmp    15af <phase_6+0x31>
    15e8: be 00 00 00 00        mov    $0x0,%esi
    15ed: eb 17                 jmp    1606 <phase_6+0x88>
    15ef: 48 8b 52 08           mov    0x8(%rdx),%rdx
    15f3: 83 c0 01              add    $0x1,%eax
    15f6: 48 63 ce              movslq %esi,%rcx
    15f9: 39 04 8c              cmp    %eax,(%rsp,%rcx,4)
    15fc: 7f f1                 jg     15ef <phase_6+0x71>
    15fe: 48 89 54 cc 20        mov    %rdx,0x20(%rsp,%rcx,8)
    1603: 83 c6 01              add    $0x1,%esi
    1606: 83 fe 05              cmp    $0x5,%esi
    1609: 7f 0e                 jg     1619 <phase_6+0x9b>
    160b: b8 01 00 00 00        mov    $0x1,%eax
    1610: 48 8d 15 f9 3a 20 00  lea    0x203af9(%rip),%rdx        # 205110 <node1>
    1617: eb dd                 jmp    15f6 <phase_6+0x78>
    1619: 48 8b 5c 24 20        mov    0x20(%rsp),%rbx
    161e: 48 89 d9              mov    %rbx,%rcx
    1621: b8 01 00 00 00        mov    $0x1,%eax
    1626: eb 12                 jmp    163a <phase_6+0xbc>
    1628: 48 63 d0              movslq %eax,%rdx
    162b: 48 8b 54 d4 20        mov    0x20(%rsp,%rdx,8),%rdx
    1630: 48 89 51 08           mov    %rdx,0x8(%rcx)
    1634: 83 c0 01              add    $0x1,%eax
    1637: 48 89 d1              mov    %rdx,%rcx
    163a: 83 f8 05              cmp    $0x5,%eax
    163d: 7e e9                 jle    1628 <phase_6+0xaa>
    163f: 48 c7 41 08 00 00 00  movq   $0x0,0x8(%rcx)
    1646: 00 
    1647: bd 00 00 00 00        mov    $0x0,%ebp
    164c: eb 07                 jmp    1655 <phase_6+0xd7>
    164e: 48 8b 5b 08           mov    0x8(%rbx),%rbx
    1652: 83 c5 01              add    $0x1,%ebp
    1655: 83 fd 04              cmp    $0x4,%ebp
    1658: 7f 11                 jg     166b <phase_6+0xed>
    165a: 48 8b 43 08           mov    0x8(%rbx),%rax
    165e: 8b 00                 mov    (%rax),%eax
    1660: 39 03                 cmp    %eax,(%rbx)
    1662: 7e ea                 jle    164e <phase_6+0xd0>
    1664: e8 63 04 00 00        callq  1acc <explode_bomb>
    1669: eb e3                 jmp    164e <phase_6+0xd0>
    166b: 48 8b 44 24 58        mov    0x58(%rsp),%rax
    1670: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
    1677: 00 00 
    1679: 75 09                 jne    1684 <phase_6+0x106>
    167b: 48 83 c4 60           add    $0x60,%rsp
    167f: 5b                    pop    %rbx
    1680: 5d                    pop    %rbp
    1681: 41 5c                 pop    %r12
    1683: c3                    retq   
    1684: e8 57 f8 ff ff        callq  ee0 <__stack_chk_fail@plt>
```

最后一个phase比较复杂，看汇编还不能运行获得数据分析实在难度太大而且枯燥的分析过程还不如自己写一遍来得清楚，所以就不一步一步看了。

但是总的来说最后一个phase都是一个排序，只要按照冒泡排序的思路去理解问题就不大。

比如我这个就是给了六个数，需要给出顺序排列时这些数的分别应当处在的位置。实在做不出来的话可以尝试在找出六个数之后暴力尝试从大到小，从小到大或者相应数所在的位置，总能猜出一个答案来。

答案是 `2 5 4 1 6 3`.

___

炸弹到此为止就拆完了。众所周知还有一个隐藏炸弹，但是不计分，所以看个人能力吧，我没做所以也就不讲了。

归根结底bomblab考察的是阅读汇编的能力，而这在之后的attacklab以及期中期末考试中都会用到，所以如果汇编水平欠缺的话抓紧补上吧。

最后还是提醒一句，不要过多关注奇技淫巧，还是要关注通用的解题方案，形成自己阅读汇编的模式。











