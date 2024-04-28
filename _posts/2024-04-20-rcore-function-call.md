---
title: 【笔记】rCore (RISC-V)：函数调用与调用栈
date: 2024-04-20 16:00:00 +0800
categories: [rCore, RISC-V]
tags: [rCore, RISC-V]
img_path: /assets/img/
---

# 函数调用是一种控制流结构

程序中构造的 **控制流** (Control Flow) 具有多种结构。在汇编级别，它们核心都是利用跳转一类的指令：

| 控制流结构                   | 汇编指令             | 说明                                 |
|------------------------------+----------------------+--------------------------------------|
| 分支结构 (如 if/switch 语句) | B 型指令 (条件分支)  | beq, bne, blt, bltu, bge, bgeu       |
| 循环结构 (如 for/while 语句) | 不支持 loop 循环指令 | 需组合使用寄存器、分支指令和跳转指令 |
| 函数调用 (Function Call)     | J 型 (无条件跳转)    | jal 和 jalr                          |

其他控制流都只需要跳转到一个 *编译期固定下来* 的地址，而函数调用的返回跳转是跳转到一个 
*运行时确定* （确切地说是在函数调用发生的时候）的地址。

B 型和 J 型的核心在于控制 pc 寄存器，打破原有的执行顺序，让 pc 指向新的地址，从这个新地址上执行指令。

（原有的执行顺序指 `pc <- pc + 4`，这里和下面都假设这位于指令长度为 4 字节）

# jal 指令

jal 指令表示 jump and link （跳转并链接），指令的调用形式为 `jal rd, offset`。

jal 被描述成 link and jump 更为准确，因为它做了两件事：
* 链接（或者说保存）原本下个指令的地址到 rd：`rd <- pc + 4`
* 修改 pc 的值来从转到新的地址：`pc <- pc + offset`

jal 通常具有两种功能：
1. 实现函数调用：
  * 通常保存到返回地址寄存器 (ra)，也就是 `jal ra, offset`
    * pc 在当前地址 + offset 的那个新地址上执行一些指令之后，可以通过 ra 来知道函数调用前的下一条指令
      * 当 pc 处于新的地址上时，如果出现函数调用，则会修改 ra，为了避免指令的地址丢失，
        需要使用调用栈来保存那些跳转前的上下文和下一条指令
2. 实现无条件跳转：
  * 将目标寄存器从 ra 换成零寄存器（x0），因为写入 x0 不改变其值
  * 其实也就是 `jal x0, offset`
    * `x0 <- pc + 4`：丢弃原本要执行的下个指令
    * `pc <- pc + offset`：修改 pc，让程序从新的地址开始执行指令
      * 因为“跳转”就是打破原有执行顺序，也就是修改 pc
      * “无条件”这个词表明直接跳转到当前地址 + offset 的那个新地址，即不需要 link 的功能

jal 实际上并没有限制 rd 必须为 ra 或者 x0：rd 可以为 32 个寄存器中的任何一个。它的核心功能只有两点：
* 保存跳转前的下个指令的地址到一个寄存器
* 修改 pc 的值，从新的地址来执行那个新地址上的指令

```text
jal 指令格式 (31 <- 0)
imm[20|10:1|11|19:12] rd 1101111 
```

> 和分支指令类似，jal 将 20 位立即数乘以 2，符号扩展后与 pc 相加，从而得到跳转目标地址。


# jalr 指令

其调用形式为 `jal rd, offset(rs)`，它做了两件事：
* 链接（或者说保存）原本下个指令的地址到 rd：`rd <- pc + 4`
* 修改 pc 的值来转到新的地址：`pc <- pc + offset(rs)`
  * 新的地址为 rs 寄存器中的值 + offset

功能:
1. 调用那些地址需要动态计算的过程
2. 将 ra 和 x0 分别作为源寄存器和目标寄存器，实现从过程中返回
3. 将 x0 作为目标寄存器，则能实现需要计算跳转地址的 switch 和 case 语句
4. `ret` 伪指令：`jalr x0, 0(x1)`，丢弃原本下个指令的地址，直接跳转到 ra 内的地址，来达到从一个函数调用中进行返回

```text
jalr 指令格式 (31 <- 0)
imm[11:0] rs1 000 rd 1100111
```

# jal 和 jalr 总结

```text
                 ┌───────┐ target = pc + imm
         jal rd, │imm    │─────┐           
              │  └───────┘     ▼           
rd <- pc+4 ◄──┤                pc <- target 
              │  ┌───────┐     ▲           
        jalr rd, │imm(rs)├─────┘           
                 └───────┘ target = rs + imm
```

注意：指令的立即数总是有范围的，这意味着如果一个函数跳转的距离太大，单纯的一步
jal(r) 无法做到跳到那个遥远的函数入口。所以函数跳转实际上：

* `jal(r)` 用于短距离跳转
* `call` 伪指令 (即 `auipc rd, offsetHi` 和 `jalr rd, offsetLo(rd)`) ：中距离跳转
* 长距离跳转：分步跳转、跳转表等（实质上是采用更多指令来跳转）

# 函数调用栈

`jal ra, offset` 之类的指令把原来下一步的执行流地址赋给了 ra，然后在新的地址上执行指令。

但新地址上的指令依然可以修改 ra 以及任意通用寄存器，这造成之前寄存器内的数据被覆盖。

所以，当发生嵌套的函数调用时，需要一个数据结构保存，在跳转前的、某些寄存器内的值。

这些需要保持不变的寄存器的集合就被称为“函数调用上下文” (Function Call Context)。

换句话说，在调用发生前，某些寄存器内的值被复制（保存）到栈上，当调用返回时（即退出前)，从栈上恢复数据到相应的寄存器内。

## 寄存器与调用规范

在函数调用中，寄存器可以分为
* 临时寄存器 vs 保存寄存器
  * 临时寄存器：其值在函数调用前后保持不变
  * 保存寄存器：其值在函数调用前后可能改变
    * 这还可以细分为：被调用者保存(Callee-Saved) 寄存器 vs 调用者保存(Caller-Saved) 寄存器
      * 被调者所保存的：ra、sp 和保存寄存器 (s 开头的寄存器, 其中包括 s0 = fp)
      * 调用者保存：函数参数 (a 开头) 和临时寄存器 (t 开头)

| 寄存器  | ABI 名称 | 描述                      | 调用前后是否一致 |   保存者 |
|---------+----------+---------------------------+:----------------:+---------:|
| x0      | zero     | 硬连接线为 0              |         -        |        - |
| x1      | ra       | 返回地址                  |        否        | 被调用者 |
| x2      | sp       | 栈指针                    |        是        | 被调用者 |
| x3      | gp       | 全局指针                  |         -        |        - |
| x4      | tp       | 线程指针                  |         -        |        - |
| x5      | t0       | 临时寄存器/备用链接寄存器 |        否        |   调用者 |
| x6-x7   | t1-t2    | 临时寄存器                |        否        |   调用者 |
| x8      | s0/fp    | 保存寄存器/帧指针         |        是        | 被调用者 |
| x9      | s1       | 保存寄存器                |        是        | 被调用者 |
| x10-x11 | a0-a1    | 函数参数/返回值           |        否        |   调用者 |
| x12-x17 | a2-a7    | 函数参数                  |        否        |   调用者 |
| x18-x27 | s2-s11   | 保存寄存器                |        是        | 被调用者 |
| x28-x31 | t3-t6    | 临时寄存器                |        否        |   调用者 |

其中，对于带数字的别名寄存器：

|           寄存器组          | 含义           | 功能                                                |
|:---------------------------:+----------------+-----------------------------------------------------|
|     a0-a7 <br>(x10-x17)     | 函数参数寄存器 | 用于传递输入参数。 a0-a1 还可以作为函数返回值寄存器 |
| t0~t6 <br>(x5-x7, x28-x31 ) | 临时寄存器     | 在被调函数中可以随意使用无需保存                    |
| s0-s11 <br>(x8-x9, x18-x27) | 保存寄存器     | 临时寄存器，但被调函数保存后才能在被调函数中使用    |

* ra 寄存器：被调用者函数可能也会调用函数，在调用之前就需要修改 ra 使得这次调用能正确返回。因此，每个函数都需要在开头保存
  ra 到自己的栈帧中，并在结尾使用 ret 返回之前将其恢复。
* a0 还是 a1：当返回值的大小小于等于位宽时，只需使用 a0，超过位宽则使用 a0 + a1
* x0、gp、tp 在一个程序运行期间都不会变化，因此不必放在函数调用上下文中
* 以上划分基于 RISC-V 架构上的 C 语言调用规范。调用规范规定
  * 函数的输入参数和返回值如何传递
  * 函数调用上下文中调用者/被调用者保存寄存器的划分
  * 其他的在函数调用流程中对于寄存器的使用方法


## 栈帧

栈帧 (Stack Frame) 是当前执行函数用于存储局部变量和函数返回信息的内存结构，是 `[新 sp, 旧 sp)` 
或者说 `[fp, sp)` 区间的物理内存。

在 RISC-V 架构中，栈是从高地址向低地址增长的。所以开辟一个 SF 只需将栈指针向低位址移动一个栈大小
(`addi sp, sp, -framesize`)，释放则只需将栈指针向高位址移动一个栈大小 (`addi sp, sp, framesize`)。

SF 的大小并不是固定的，因为它取决于局部变量和寄存器的使用情况。

对于栈顶那层 SF 来说，sp 指向栈顶，fp 则指向该 SF 的开头。

![](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/StackFrame.png)

```nasm
; 一个汇编代码控制 SF 的示例
; https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter1/5support-func-call.html

; 开场
; 为当前函数分配 64 字节的栈帧
addi sp, sp, -64
; 将 ra 和 fp 压栈保存
sd ra, 56(sp)
sd s0, 48(sp)
; 更新 fp 为当前函数栈帧顶端地址
addi s0, sp, 64

; 函数执行
; 中间如果再调用了其他函数会修改 ra

; 结尾
; 恢复 ra 和 fp
ld ra, 56(sp)
ld s0, 48(sp)
; 退栈
addi sp, sp, 64
; 返回，使用 ret 指令或其他等价的实现方式
ret
```