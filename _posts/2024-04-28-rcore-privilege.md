---
title: 【笔记】rCore (RISC-V)：特权级
date: 2024-04-28 08:00:00 +0800
categories: [rCore, RISC-V]
tags: [rCore, RISC-V]
img_path: /assets/img/rcore
---


> src: <https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter2/1rv-privilege.html>

# Why

> 实现特权级机制的根本原因是应用程序运行的安全性不可充分信任。

从我的理解来看，权级的出现主要是为了 **处理程序的错误** 和 **限制程序的资源（指令、内存地址）**。

对于前者，一个程序出现错误时，这导致执行环境被破坏，影响其他程序的运行。

对于后者，如果所有程序都对机器拥有完全的资源控制权，这增加了执行环境被破坏的风险。

所以，需要给程序划分级别来保证低权级执行的安全性：

> 低特权级软件只能做高特权级软件允许它做的，且超出低特权级软件能力的功能必须寻求高特权级软件的帮助。


# What

## 特权级有 4 类

特权级 (privilege) 从低到高分为
* 用户/应用模式 (user mode) => 上层应用（用户态）
* 监管模式 (supervisor mode) => 操作系统层（内核态）
* 虚拟监督模式 (hypervisor) => 尚未成型，暂时忽略
* 机器模式 (machine mode) => 最高权级，拥有对机器的完全控制 => 监督模式执行环境 (SEE, Supervisor Execution Environment)

对于 M-S-U 权级中间的地带，SBI 和 ABI 是桥梁： M mode => SBI => S mode => ABI => U mode
* SBI (Supervisor Binary Interface)：监督模式二进制接口，SEE 和 S 模式的内核之间的接口
* ABI (Application Binary Interface)：应用程序二进制接口，内核和 U 模式的应用程序之间的接口，又名系统调用 (syscall)

这两种接口调用并不是普通的函数调用控制流，而是特殊的 **陷入异常控制流**，在该过程中会切换 CPU 特权级。
因此只有将接口下降到机器/汇编指令级才能够满足其跨高级语言的通用性和灵活性。

（说白了，就是跨权级调用会触发一种叫“陷入” (trap) 的异常机制，低权级向上一层高权级请求来处理，CPU 切换权级，
处理之后，回到低权级环境。这种切换的方式是指令级别的，与编程语言无关。）

![](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/PrivilegeStack.png)


## 特权级是按需实现的

RISC-V 架构中，只有 M 模式是必须实现的，剩下的特权级则可以根据应用的实际需求进行调整：

* 简单的嵌入式应用只需要实现 M 模式；
* 带有一定保护能力的嵌入式系统需要实现 M/U 模式；
* 复杂的多任务系统则需要实现 M/S/U 模式。

## 特权级需与硬件协同

处理器会设置不同级别的执行环境，并划分出不同级别专属的指令集，规定内核态的指令只能在内核的执行环境中执行，而且
处理器在执行指令前会对其权级进行检查，如果在用户态执行环境中执行这些内核态特权级指令，会产生异常。

## 异常控制流

* 常规控制流：顺序、循环、分支、函数调用
* 异常控制流 (ECF, Exception Control Flow)。其原因/形式： 
  * （陷入）用户态软件为获得内核态操作系统的服务功能而执行特殊指令
  * （故障）在执行某条指令期间产生了错误（如执行了用户态不允许执行的指令或者其他错误）并被 CPU 检测到

| Interrupt | Exception Code | Description                    | 中文           | 陷入指令 |
|:---------:+:--------------:+--------------------------------+----------------+----------|
|     0     |        0       | Instruction address misaligned | 指令地址不对齐 |          |
|     0     |        1       | Instruction access fault       | 指令访问故障   |          |
|     0     |        2       | Illegal instruction            | 非法指令       |          |
|     0     |        3       | Breakpoint                     | 断点           | ebreak   |
|     0     |        4       | Load address misaligned        | 读数地址不对齐 |          |
|     0     |        5       | Load access fault              | 读数访问故障   |          |
|     0     |        6       | Store/AMO address misaligned   | 存数地址不对齐 |          |
|     0     |        7       | Store/AMO access fault         | 存数访问故障   |          |
|     0     |        8       | Environment call from U-mode   | U 模式环境调用 | ecall    |
|     0     |        9       | Environment call from S-mode   | S 模式环境调用 | ecall    |
|     0     |       11       | Environment call from M-mode   | M 模式环境调用 | ecall    |
|     0     |       12       | Instruction page fault         | 指令页故障     |          |
|     0     |       13       | Load page fault                | 读数页故障     |          |
|     0     |       15       | Store/AMO page fault           | 存数页故障     |          |

这个表显示了异常中的所有的陷入和故障类型。陷入是通过 ebreak 或 ecall 主动触发的，但故障并不是。

调用 ecall 指令时，会根据 CPU 所处的权级来触发相应的异常码，比如当处于 U 模式，则触发 8。

对这些异常进行分类：
- 访问故障异常：在物理内存地址不支持访问类型时发生，如尝试写入 ROM。
- 断点异常：在执行 ebreak 指令，或者地址或数据与调试触发器（debug trigger）匹配时发生。
- 环境调用异常：在执行 ecall 指令时发生。
- 非法指令异常：在对无效操作码进行译码时发生。
- 不对齐地址异常：在有效地址不能被访问位宽整除时发生，如地址为 0x12 的 amoadd.w。

此外，有一列中断标识符，涉及中断 (1) 与异常 (0) 的区别：

|      | 发生 | 原因                                                                               |
|------+------+------------------------------------------------------------------------------------|
| 中断 | 异步 | 来自软件、时钟和外部。可以发生在任何时刻。<br>与指令流异步的外部事件，如点击鼠标。 |
| 异常 | 同步 | 指令执行的一种结果，如非法指令、内存访问错误等                                     |

对所有 RISC-V 系统来说，一个共性问题是如何处理异常和屏蔽中断。见 [中断]。

[中断]: ../rcore-os-multiprograms/#中断

RISC-V 的异常是精确的：在异常点之前的所有指令都执行完毕，而异常点之后的指令则都未开始执行。

## CSR

CSR = control and status register （控制与状态寄存器），它不同于 32 个通用寄存器，比如
* CSR 划分了权级，低权级无法读写高权级的 CSR
* 在级别范围内，可以对 CSR 进行读写。CSR 的读写权限与寄存器的编号有关。
  * CSR 的编号是 12 位的，而不是像通用寄存器那样只有 5 位。所以 CSR 可高达 4096 个。
  * CSR 的读写性由最高的 2 位 `[11:8]` 的确定，11 开头的 CSR 只允许读，其余允许读写
  * `[9:8]` 位标识了最低读写权限，`00` 为 U 开始，`01` 为 S 开始等等
* 有些 CSR 的写入者是机器
* CSR 对位域的读写具有如下规范（即读写行为）
  * WPRI (Write Preserve, Read Ignore)：写入任何值都不会改变这个位的值，忽略读取这个位的值；意味着程序写入这个位是，自己去保存这个位的值
  * WLRL (Write Legal, Read Legal)：只写入合法值，只读取合法值；写入非法值是可以但不必抛出异常
  * WARL (Write Any, Read Legal)：写入任何值，只读取出合法值；写入非法值不会抛出异常
* CSR 读写指令是原子性的，要么全部成功，要么全部失败，是不会被打断地完成多个读写操作的指令
* 读写 CSR 需要使用专门的指令，也有一些伪指令，以下罗列一部分

| CSR 指令 | 语法                 | 功能                                                                                           | 名称             | 伪指令             |
|----------+----------------------+------------------------------------------------------------------------------------------------+------------------+--------------------|
| csrrw    | `csrrw rd, csr, rs1` | `t = CSRs[csr]; CSRs[csr] = x[rs1]; x[rd] = t`<br>CSR 数据读到 rd，然后把 rs1 的数据写入 CSR   | 读后写           |                    |
| csrr     | `csrr rd, csr`       | `x[rd] = CSRs[csr]` CSR 数据读到 rd                                                            | 读               | csrrs rd, csr, x0  |
| csrw     | `csrw csr, rs1`      | `CSRs[csr] = x[rs1]` 把 rs1 的数据写入 CSR                                                     | 写               | csrrw x0, csr, rs1 |
| sret     | `sret`               | `pc <- sepc`；特权模式=sstatus.SPP;<br>sstatus.SIE=sstatus.SPIE; sstatus.SPIE=1；sstatus.SPP=0 | 监管模式异常返回 |                    |

`sret` 指令做的事情比较多，这里结合 `sstatus` CSR 中具有一些位信息来解释一下：
* `pc <- CSRs[spec]` 把 sret 之后的执行权转交给 `spec` CSR 中的地址，接下来的程序流应该从那个地址开始
* `特权模式=sstatus.SPP` 将机器权级切换成 SPP 这个位域表示的权级，所以如果要保证权限回到 U，应保证在执行 sret 之前 SPP 为 0
  * 虽然 SPP 表示进入 S 模式之前的那个模式 (0 为 U，1 为 S)，但是陷入到 S 之后，SPP 会被
  * 当然，这带来一个问题，直接硬编码成权级回到 U 模式，而不是 SPP 表示的模式不就能解决这里的安全问题了吗。我猜是出于设计的一致性：
    mret 的权级切换也是以 SPP 为准，意味着可以从 M 回到 S 或者 U，只不过 M 直接到 U 可能带来安全风险。
  * 不管怎样 sstatus.SPP 都会在 sret 设置完权限之后，变成 0
* 然后是两个位域，SIE 和 SPIE，它们与中断有关（异常的中断位为 0，所以不属于中断）
  * SIE=0 关闭所有中断，SIE=1 开启所有中断。SIE 只在 S 模式下生效；此外还可以通过 sie CSR 关闭单独一些中断来源。
  * SPIE 表示陷入到 S 模式之前，S 级别的中断是否开启了。
  * 当陷入到 S 模式的时候，SPIE 被设置成 SIE，SIE 被设置成 0
  * 当 sret 发生的时候 SIE 被设置成 SPIE，SPIE 被设置成 1

# How

```text
   ┌──────┐ mret  ┌──────┐ sret  ┌──────┐
   │      │──────►│      │──────►│      │
   │M mode│       │S mode│       │U mode│
   │      │◄──────│      │◄──────│      │
   └──────┘ ecall └──────┘ ecall └──────┘

其他特权级指令：wfi、sfence.vma、访问特定模式 CSR 的指令
```

对于 U 陷入到 S 模式这种权级切换的情况，需要知道 S 级别的 CSR：

| CSR      | 与 Trap 相关的功能                                                                                         |
|----------+------------------------------------------------------------------------------------------------------------|
| sstatus  | 具有固定的位信息，其中 `SPP` 位告知了 trap 发生前处于 S (SPP=1) 还是 U (SPP=0) 权级                        |
| sepc     | 当 trap 是异常时，记录 trap 发生之前执行的最后一条指令的地址；sret 时赋给 pc                               |
| scause   | 具有固定的位信息，给出了 trap 的原因                                                                       |
| stval    | 与 sccause 有关，作为 trap 的额外信息，比如对于 access/page fault，这为虚拟地址                            |
| stvec    | 处理 trap 程序/函数/指令的起始地址；<br>`[1:0]` 位表示模式，剩余位为虚拟地址（4 字节对齐，即最低两位必须为 0） |
| sscratch | 供 S 用的临时寄存器，通常存储指针，来引导 S/U 环境切换，比如交换内核和用户态的栈                           |

```text

   从 U 模式切换到 S 模式（中断/异常/ecall）
                        ▼                                                         
┌─────────────────────────────────────────────────────────────────────────────────
│  硬件与 CSR                                                                     
│sstatus.SPP=0 （因为从 U 模式来的）                                              
│sepc 指向 trap 处理之后的下条指令的地址 （用于 sret）                            
│scause 和 stval 被修改成此次 trap 的原因和附加信息                               
│pc <- stvec （程序跳转到处理中断/陷入/异常的指令入口，在系统初始化中设置好 stvec）
└───────────────────────┬─────────────────────────────────────────────────────────
                        ▼                                                         
┌─────────────────────────────────────────────────────────────────────────────────
│  内核的事情                                                              
│交换用户栈和内核栈，sp 指向内核栈                                       
│建栈：保存应用程序的 trap 上下文（记录应用程序的寄存器、一些 CSR 的状态)        
│根据 scause 和 stval 处理 trap                                                  
│退栈：恢复应用程序的寄存器、恢复或者设置一些 CSR 的状态                         
│换栈：sp 指向用户栈                                                             
└───────────────────────┬─────────────────────────────────────────────────────────
                        ▼
sret：回到 U 模式，执行应用程序
```
