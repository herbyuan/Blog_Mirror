---
title: ICS04 Archlab
date: 2023-02-07 21:20:54
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

Arch Lab的目的是加深对于Y86指令执行的理解。实验分城三个部分，Part A是完成三个程序的汇编代码，Part B要向已有的指令集中加入两条指令，Part C是优化代码，使其运行的尽可能快。

## Part A
在sim/misc文件夹中，我们要编写三个Y86指令汇编代码，实现examples.c中相应函数的功能。

### `sum.ys`

先来看一下c的程序：
```C
/* sum_list - Sum the elements of a linked list */
long sum_list(list_ptr ls)
{
    long val = 0;
    while (ls) {
  val += ls->val;
  ls = ls->next;
    }
    return val;
}
```

这是一个迭代求和的过程。实验材料并没有给出模板，但是可以参考教材图4-7给出的完整的程序文件的例子。这样的程序包括了数据和相应的所有指令。略微的区别在于书上是对一个数组进行求和，而lab要求对链表进行求和，所以要对取下一个数的方式做出修改，也就是先从当前节点拿到下一个节点的地址，然后得到下一个节点的数据。代码如下：

```
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main       # Execute main program
    halt            # Terminate program

# sample linked list of 3 elements
    .align 8
ele1:
    .quad 0x00a
    .quad ele2
ele2:
    .quad 0x0b0
    .quad ele3
ele3:
    .quad 0xc00
    .quad 0

main:
    irmovq ele1,%rdi
    call sum_list        # sum_list
    ret

# long sum_list(list_ptr ls)
# start in %rdi
sum_list:
    xorq %rax,%rax       # sum = 0
    pushq %r8
L1:
    andq %rdi,%rdi       # Set CC
    je L2
    mrmovq (%rdi),%r8
    addq %r8,%rax
    mrmovq 8(%rdi),%rdi
    jmp L1

L2:
    popq %r8
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```

### `rsum.ys`
先来看一下c的程序：
```C
/* rsum_list - Recursive version of sum_list */
long rsum_list(list_ptr ls)
{
    if (!ls)
  return 0;
    else {
  long val = ls->val;
  long rest = rsum_list(ls->next);
  return val + rest;
    }
}
```

这是一个递归调用函数来求和的过程。程序的初始化，数据设置，main都和上一问一样，仅需改变求和的函数实现。在函数调用时，应保存好caller saved寄存器中的数据，将传递的参数放在rdi等寄存器中，同时注意被调用的函数要保存好callee saved寄存器中的值。代码如下：
```
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main       # Execute main program
    halt            # Terminate program

# sample linked list of 3 elements
    .align 8
ele1:
    .quad 0x00a
    .quad ele2
ele2:
    .quad 0x0b0
    .quad ele3
ele3:
    .quad 0xc00
    .quad 0

main:
    irmovq ele1,%rdi
    call rsum_list        # sum_list
    ret

# long sum_list(list_ptr ls)
# ls in %rdi
rsum_list:
    andq %rdi,%rdi       # Set CC
    je L2
    pushq %r9
    mrmovq (%rdi), %r9
    mrmovq 8(%rdi), %rdi
    call rsum_list
    addq %r9,%rax
    popq %r9
    ret
L2:
    irmovq $0,%rax
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```

### `bubble.ys`
先来看一下c的程序：
```C
/* bubble_sort - Sort long numbers at data in ascending order */
void bubble_sort(long *data, long count)
{
     long *i, *last;
     for(last = data + count - 1; last > data; last--) {
         for(i = data; i < last; i++) {
             if(*(i + 1) < *i) {
                 long t = *(i + 1);
                 *(i + 1) = *i;
                 *i = t;
             }
         }
    }
}
```

需要用冒泡排序的方式对六个数升序排列，并且放在原先的地址空间上。也就是说，只是在两个数要换顺序的时候交换相应地址中的数值。由于没有递归过程，只要想明白循环和判断在汇编中的实现，此题并不困难。代码如下：

```
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main       # Execute main program
    halt            # Terminate program

# sample linked list of 3 elements
    .align 8
Array:
    .quad 0xbca
    .quad 0xcba
    .quad 0xacb
    .quad 0xcab
    .quad 0xabc
    .quad 0xbac

main:
    irmovq Array,%rdi
    irmovq $6,%rsi
    call bubble_sort
    ret

# void bubble_sort(long *data, long count)
# data in %rdi, count in %rsi
bubble_sort:
    pushq %r8
    pushq %r9
    pushq %r10
    pushq %r11
    pushq %r12
    pushq %r13
    pushq %r14
    irmovq $8,%r8        # Constant 8
    rrmovq %rdi,%r9    # last in %r9
    addq %rsi,%rsi
    addq %rsi,%rsi
    addq %rsi,%rsi
    subq %r8,%rsi
    addq %rsi,%r9    # last = data + count - 1
L1:
    rrmovq %r9,%r10
    subq %rdi,%r10
    jle L2
    rrmovq %rdi,%r11   # i in %r11
L3:
    rrmovq %r9,%r12
    subq %r11,%r12
    jle L4
    mrmovq (%r11),%r12    # *i
    mrmovq 8(%r11),%r13    # *(i+1)
    rrmovq %r12,%r14
    subq %r13,%r14
    jle L5
    rrmovq %r13,%r14
    rmmovq %r12,8(%r11)
    rmmovq %r14,(%r11)
L5:
    addq %r8,%r11   # i++
    jmp L3
L4:
    subq %r8,%r9   # last--
    jmp L1
L2:
    popq %r14
    popq %r13
    popq %r12
    popq %r11
    popq %r10
    popq %r9
    popq %r8
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```

## Part B
在这一部分中，我们需要seq-full.hcl文件，修改增加两条Y86指令。这里我非常建议详细阅读seq-full.hcl的所有内容，这不论是对于完成lab还是期中考试都有非常大的帮助。

首先，给出两条指令的描述，可以参考书本的示例。按照相应流水线阶段的过程写出相应的步骤并不困难，需要注意的一个是加法指令一定不要忘记设置状态码CC，另一个是ALU单元的两个操作数的顺序。
```
#              iaddq V, rB               
# Fetch    icode:ifun <- M1[PC]
#          rA:rB <- M1[PC+1]
#          valC <- M8[PC+2]
#          valP <- PC+10
# Decode   valB <- R[rB]
# Excute   valE <- valC + valB
#          Set CC
# Memory
# Write    R[rB] <- valE
# PC       PC <- valP      
#
#              jm V, rB               
# Fetch    icode:ifun <- M1[PC]
#          rA:rB <- M1[PC+1]
#          valC <- M8[PC+2]
#          valP <- PC+10
# Decode   valB <- R[rB]
# Excute   valE <- valC + valB
# memory   valM <- M8[valE]
# Write    
# PC       PC <- valM
```
然后在HCL文件相应的位置添加上两种指令相应的操作即可。
```
#/* $begin seq-all-hcl */
####################################################################
#  HCL Description of Control for Single Cycle Y86-64 Processor SEQ   #
#  Copyright (C) Randal E. Bryant, David R. O'Hallaron, 2010       #
####################################################################

## Your task is to implement the iaddq instruction
## The file contains a declaration of the icodes
## for iaddq (IIADDQ)
## Your job is to add the rest of the logic to make it work

####################################################################
#    C Include's.  Don't alter these                               #
####################################################################

quote '#include <stdio.h>'
quote '#include "isa.h"'
quote '#include "sim.h"'
quote 'int sim_main(int argc, char *argv[]);'
quote 'word_t gen_pc(){return 0;}'
quote 'int main(int argc, char *argv[])'
quote '  {plusmode=0;return sim_main(argc,argv);}'

####################################################################
#    Declarations.  Do not change/remove/delete any of these       #
####################################################################

##### Symbolic representation of Y86-64 Instruction Codes #############
wordsig INOP  'I_NOP'
wordsig IHALT 'I_HALT'
wordsig IRRMOVQ 'I_RRMOVQ'
wordsig IIRMOVQ 'I_IRMOVQ'
wordsig IRMMOVQ 'I_RMMOVQ'
wordsig IMRMOVQ 'I_MRMOVQ'
wordsig IOPQ  'I_ALU'
wordsig IJXX  'I_JMP'
wordsig ICALL 'I_CALL'
wordsig IRET  'I_RET'
wordsig IPUSHQ  'I_PUSHQ'
wordsig IPOPQ 'I_POPQ'
# Instruction code for iaddq instruction
wordsig IIADDQ  'I_IADDQ'
#Instruction code for jm instruction
wordsig IJM     'I_JM'

##### Symbolic represenations of Y86-64 function codes                  #####
wordsig FNONE    'F_NONE'        # Default function code

##### Symbolic representation of Y86-64 Registers referenced explicitly #####
wordsig RRSP     'REG_RSP'      # Stack Pointer
wordsig RNONE    'REG_NONE'     # Special value indicating "no register"

##### ALU Functions referenced explicitly                            #####
wordsig ALUADD  'A_ADD'   # ALU should add its arguments

##### Possible instruction status values                             #####
wordsig SAOK  'STAT_AOK'  # Normal execution
wordsig SADR  'STAT_ADR'  # Invalid memory address
wordsig SINS  'STAT_INS'  # Invalid instruction
wordsig SHLT  'STAT_HLT'  # Halt instruction encountered

##### Signals that can be referenced by control logic ####################

##### Fetch stage inputs    #####
wordsig pc 'pc'       # Program counter
##### Fetch stage computations    #####
wordsig imem_icode 'imem_icode'   # icode field from instruction memory
wordsig imem_ifun  'imem_ifun'    # ifun field from instruction memory
wordsig icode   'icode'   # Instruction control code
wordsig ifun    'ifun'    # Instruction function
wordsig rA    'ra'      # rA field from instruction
wordsig rB    'rb'      # rB field from instruction
wordsig valC    'valc'    # Constant from instruction
wordsig valP    'valp'    # Address of following instruction
boolsig imem_error 'imem_error'   # Error signal from instruction memory
boolsig instr_valid 'instr_valid' # Is fetched instruction valid?

##### Decode stage computations   #####
wordsig valA  'vala'      # Value from register A port
wordsig valB  'valb'      # Value from register B port

##### Execute stage computations  #####
wordsig valE  'vale'      # Value computed by ALU
boolsig Cnd 'cond'      # Branch test

##### Memory stage computations   #####
wordsig valM  'valm'      # Value read from memory
boolsig dmem_error 'dmem_error'   # Error signal from data memory


####################################################################
#    Control Signal Definitions.                                   #
####################################################################

################ Fetch Stage     ###################################

# Determine instruction code
word icode = [
  imem_error: INOP;
  1: imem_icode;    # Default: get from instruction memory
];

# Determine instruction function
word ifun = [
  imem_error: FNONE;
  1: imem_ifun;   # Default: get from instruction memory
];

bool instr_valid = icode in 
  { INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
         IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ, IJM };

# Does fetched instruction require a regid byte?
bool need_regids =
  icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
         IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ, IJM };

# Does fetched instruction require a constant word?
bool need_valC =
  icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ, IJM };

################ Decode Stage    ###################################

## What register should be used as the A source?
word srcA = [
  icode in { IRRMOVQ, IRMMOVQ, IOPQ, IPUSHQ  } : rA;
  icode in { IPOPQ, IRET } : RRSP;
  1 : RNONE; # Don't need register
];

## What register should be used as the B source?
word srcB = [
  icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ, IJM  } : rB;
  icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
  1 : RNONE;  # Don't need register
];

## What register should be used as the E destination?
word dstE = [
  icode in { IRRMOVQ } && Cnd : rB;
  icode in { IIRMOVQ, IOPQ, IIADDQ } : rB;
  icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
  1 : RNONE;  # Don't write any register
];

## What register should be used as the M destination?
word dstM = [
  icode in { IMRMOVQ, IPOPQ } : rA;
  1 : RNONE;  # Don't write any register
];

################ Execute Stage   ###################################

## Select input A to ALU
word aluA = [
  icode in { IRRMOVQ, IOPQ } : valA;
  icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ, IJM } : valC;
  icode in { ICALL, IPUSHQ } : -8;
  icode in { IRET, IPOPQ } : 8;
  # Other instructions don't need ALU
];

## Select input B to ALU
word aluB = [
  icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
          IPUSHQ, IRET, IPOPQ, IIADDQ, IJM } : valB;
  icode in { IRRMOVQ, IIRMOVQ } : 0;
  # Other instructions don't need ALU
];

## Set the ALU function
word alufun = [
  icode == IOPQ : ifun;
  1 : ALUADD;
];

## Should the condition codes be updated?
bool set_cc = icode in { IOPQ, IIADDQ };

################ Memory Stage    ###################################

## Set read control signal
bool mem_read = icode in { IMRMOVQ, IPOPQ, IRET, IJM };

## Set write control signal
bool mem_write = icode in { IRMMOVQ, IPUSHQ, ICALL };

## Select memory address
word mem_addr = [
  icode in { IRMMOVQ, IPUSHQ, ICALL, IMRMOVQ, IJM } : valE;
  icode in { IPOPQ, IRET } : valA;
  # Other instructions don't need address
];

## Select memory input data
word mem_data = [
  # Value from register
  icode in { IRMMOVQ, IPUSHQ } : valA;
  # Return PC
  icode == ICALL : valP;
  # Default: Don't write anything
];

## Determine instruction status
word Stat = [
  imem_error || dmem_error : SADR;
  !instr_valid: SINS;
  icode == IHALT : SHLT;
  1 : SAOK;
];

################ Program Counter Update ############################

## What address should instruction be fetched at

word new_pc = [
  # Call.  Use instruction constant
  icode == ICALL : valC;
  # Taken branch.  Use instruction constant
  icode == IJXX && Cnd : valC;
  # Completion of RET instruction.  Use value from stack
  icode == IRET : valM;
  # Use value from memory
  icode == IJM : valM;
  # Default: Use incremented PC
  1 : valP;
];
#/* $end seq-all-hcl */
```

## Part C

这部分是要修改流水线，尽量提高程序的性能。

修改pipe-full.hcl，增加iaddq指令，可以让一些需要两步的命令变成一步。如果修改预测的分支或者让CC在有些时候另作他用可以让程序运行步骤更少，但实践表明不这么操作也能拿到满分。

修改ncopy.ys，采用循环展开，想办法避免load-use冒险。

如此操作可以达到CMU原版lab的满分，但是好像甚至不能达到PKU的及格分，所以需要用一些奇技淫巧来提高性能。最好的方式应当是用树的结构，网络上也有相应的文章讲述如何使用，但是需要略作修改来适配测试。然而我过于菜，没能理解其中的奥妙以及修改的方向，总是越来越垃圾最终放弃。以下是我给出的一种朴素的思路，依然可以获得满分。由于没有什么技巧性以及当时写代码时十分崩溃，我不想做任何解释。

```
#/* $begin ncopy-ys */
##################################################################
# ncopy.ys - Copy a src block of len words to dst.
# Return the number of positive words (>0) contained in src.
#
# name: He Zhuoyuan
# id: 1800013351@pku.edu.cn
#
# Describe how and why you modified the baseline code.
# 1. 增加iaddq指令，使得很多两步指令可以合并为一布
# 2. 一开始的%rax就是0，不需要赋值
# 3. n * 1循环展开，减少循环的次数，减少浪费CPE的跳转指令，同时减少load/use造成的bubble
# 4. 跳转指令是默认taken的，所以要把概率高的下一步语句写在跳转的目标位置
# 5. 处理最后的小于n个元素时，尽量平衡各个情况所需的CPE(参考了三叉树的实现方式)
# 6. 想办法减少len较小的时候的判断用的额外开销，因为此时必要开销仅仅会分摊到很少的几个数，因此对总平均CPE影响很大
# 7. 为了减少代码长度，在处理最后小于n个数的时候想办法复用代码(通过不改变条件码来决定是否跳转，并且此步骤本来应该是bubble，不增加CPE)
##################################################################
# Do not modify this portion
# Function prologue.
# %rdi = src, %rsi = dst, %rdx = len
ncopy:

##################################################################
# You can modify this portion
    # Loop header
L0:
    iaddq $-10,%rdx 
    jl LASTN         # 处理最后小于n个数

L1:         # 对10个数两个一组处理，避免load/use气泡
    mrmovq (%rdi), %r8
    mrmovq 8(%rdi),%r9
    rmmovq %r8,(%rsi)
    andq %r8, %r8
    jle L2
    iaddq $1,%rax    # if (val > 0) count++
L2: 
    rmmovq %r9,8(%rsi)
    andq %r9,%r9
    jle L3
    iaddq $1,%rax
L3:
    mrmovq 16(%rdi), %r8
    mrmovq 24(%rdi), %r9
    rmmovq %r8,16(%rsi)
    andq %r8, %r8
    jle L4
    iaddq $1, %rax      # if (val > 0) count++
L4:
    rmmovq %r9,24(%rsi)
    andq %r9,%r9
    jle L5
    iaddq $1,%rax
L5:
    mrmovq 32(%rdi), %r8
    mrmovq 40(%rdi), %r9
    rmmovq %r8,32(%rsi)
    andq %r8, %r8 
    jle L6
    iaddq $1, %rax      # if (val > 0) count++
L6:
    rmmovq %r9,40(%rsi)
    andq %r9,%r9
    jle L7
    iaddq $1,%rax 
L7:
    mrmovq 48(%rdi), %r8
    mrmovq 56(%rdi), %r9
    rmmovq %r8,48(%rsi)
    andq %r8, %r8  
    jle L8
    iaddq $1, %rax      # if (val > 0) count++
L8:
    rmmovq %r9,56(%rsi)
    andq %r9,%r9
    jle L9
    iaddq $1,%rax 
L9:
    mrmovq 64(%rdi), %r8
    mrmovq 72(%rdi), %r9
    rmmovq %r8,64(%rsi)
    andq %r8, %r8   
    jle L10
    iaddq $1, %rax        # if (val > 0) count++
L10:
    rmmovq %r9,72(%rsi)
    andq %r9,%r9
    jle L11
    iaddq $1,%rax          # if (val > 0) count++

L11:                     # next loop preparation
    iaddq $80,%rdi
    iaddq $80,%rsi
    iaddq $-10,%rdx
    jge L1

LASTN:
    mrmovq (%rdi), %r10
    iaddq $7,%rdx        # 判断剩余的len与3(10-7)的大小关系
    jl  LESS3     # <3
    jg  MORE3        # >3
    je  EQUAL3     # =3

LESS3:
    iaddq   $2,%rdx      # 10-7-2=1
    je  LAST1_2
    iaddq   $-1,%rdx    # len == 2
    je  LAST2
    ret         # len == 0 
MORE3:
    iaddq   $-3,%rdx    # 10-7+3=6 
    jg  MORE6      #  len > 6
    je  EQUAL6     # len == 6
    iaddq   $1,%rdx 
    jl LAST4     # len == 4
    je  LAST5     # len == 5    
MORE6:
    iaddq   $-2,%rdx
    jl  LAST7
    mrmovq 64(%rdi), %r9   # read src[8] from src
    je  LAST8    # len=8

LAST9:
    rmmovq %r9, 64(%rsi)
    andq %r9, %r9     # set cc

LAST8:
    mrmovq 56(%rdi), %r8   # read src[7] from src
    jle LAST8_2    
    iaddq $1, %rax        # if(rsi[8]>0) count++
LAST8_2:    
    rmmovq %r8, 56(%rsi)
    andq %r8, %r8     # set cc

LAST7:
    mrmovq 48(%rdi), %r8   # read src[6] from src
    jle LAST7_2   
    iaddq $1, %rax        # if(rsi[7]>0) count++
LAST7_2:        
    rmmovq %r8, 48(%rsi)
    andq %r8, %r8     # set cc

EQUAL6:
    mrmovq 40(%rdi), %r8   # read src[5] from src
    jle LAST6_2
    iaddq $1, %rax        # if(rsi[6]>0) count++
LAST6_2:        
    rmmovq %r8, 40(%rsi)
    andq %r8, %r8     # set cc

LAST5:
    mrmovq 32(%rdi), %r8   # read src[4] from src
    jle LAST5_2   
    iaddq $1, %rax        # if(rsi[5]>0) count++
LAST5_2:
    rmmovq %r8, 32(%rsi)
    andq %r8, %r8     # set cc

LAST4:
    mrmovq 24(%rdi), %r8   # read src[3] from src
    jle LAST4_2 
    iaddq $1, %rax        # if(rsi[4]>0) count++
LAST4_2:
    rmmovq %r8, 24(%rsi)
    andq %r8, %r8     # set cc

EQUAL3:
    mrmovq 16(%rdi), %r8   # read src[2] from src
    jle EQUAL3_2   
    iaddq $1, %rax        # if(rsi[3]>0) count++
EQUAL3_2:
    rmmovq %r8, 16(%rsi)
    andq %r8, %r8     # set cc

LAST2:
    mrmovq 8(%rdi), %r8    # read src[1] from src
    jle LAST2_2   
    iaddq $1, %rax        # if(rsi[2]>0) count++
LAST2_2:
    rmmovq %r8, 8(%rsi)
    andq %r8, %r8 

LAST1:
    # mrmovq (%rdi), %r8 # read src[0] from src
    jle LAST1_2        # 上一步操作(如果是跳转到这里执行直接跳转，不然判断上一个记录到dst的数是不是正数)
    iaddq $1, %rax        # if(rsi[1]>0) count++
LAST1_2:
    rmmovq %r10, (%rsi)
    andq %r10, %r10
    jle Done 
    iaddq $1, %rax        # if(rsi[0]>0) count++

##################################################################
# Do not modify the following section of code
# Function epilogue.
Done:
    ret
##################################################################
# Keep the following label at the end of your function
End:
#/* $end ncopy-ys */
```

___


Arch Lab到此就结束了，个人觉得这部分最重要的还是汇编的实现以及流水线的理解，但是对于Part C这样对特殊情况极致性能的追求我是十分厌恶的，毕竟为了寻求这种不通用的方法占用了大量本可以用来理解本质内容（比如说流水线的Bubble是什么冒险怎么解决Forward怎么传）的时间，对于我这种孤身一人奋战的同学来说更是极大的不划算。所以我依然建议对于lab的态度就是理解其中的内涵，而不是创造其中的奥妙。

最后，由于本人在挣扎lab时看完了所有网上能找到的资料，所以参考文献依然是全网2020.10之前的所有相关文章，如有雷同，是我抄袭。



