---
title: 【笔记】rCore (RISC-V)：程序的内存布局
date: 2024-04-25 20:00:00 +0800
categories: [rCore, RISC-V]
tags: [rCore, RISC-V]
img_path: /assets/img/rcore
---

# 程序的内存布局

一个程序加载到内存之后，这部分内存可以划分为两大块：代码区域和数据区域。

可以用段 (section) 来描述一段连续的内存空间，用 `.name` 来命名一个段，在汇编中，用 
`.section .name` 声明一个段的开始。

![](https://rcore-os.cn/rCore-Tutorial-Book-v3/_images/MemoryLayout.png)

* 代码 (code memory)：`.text` 用来存放代码（也就是指令)
* 数据 (data memory)：
  * `.rodata`：已初始化的、只读的全局数据，通常是一些常数、常量字符串
    * 提供函数访问的全局不变量
  * `.data`：已初始化的、可修改的全局数据
    * 提供函数访问的全局变量
    * gp (x3) 寄存器可以保存 .data 和 .bss 中间的地址，然后利用偏移量来访问这两块区域
    * .rodata 和 .data 段会被称为数据段（Data Segment），因为它们在程序中运行时已经初始化
  * `.bss`：未初始化的全局数据，通常由程序的加载者进行零初始化（即将这块区域逐字节清零）
    * 提供函数访问的全局变量
    * 它是程序在运行时的静态内存分配部分：bss 是 Block Started by Symbol 的缩写，因为在程序开始运行时被初始化为零，所以
      .bss 段在程序的可执行文件中并不占用实际的空间，但是在加载到内存时会分配相应的空间
  * 堆 (heap)：程序运行时动态分配的数据，如 C/C++ 中的 malloc/new 分配到的数据本体就放在堆区域，它向高地址增长
    * 数据的本体存放在堆上时，函数可以通过指向该数据的指针来访问该数据
      * 因为数据的本体（实际)大小可能只有在，运行时才可知，而指针的大小是编译时可知
      * 指针可以放在全局变量的数据段 (.data 和 .bss)，也可以放在栈帧里面
  * 栈 (stack)：存储了需要保存和恢复的函数上下文、每个函数作用域内的局部变量，向低地址增长
    * 函数上下文相关的寄存器内的数据被复制到栈上
    * 函数内的局部变量也会复制到栈上
    * 通过当前栈指针加上一个偏移量来访问栈帧内的数据

# Qemu 启动流程（内存布局向）

在 Qemu 模拟的 virt 硬件平台上，物理内存的起始物理地址为 0x80000000 ，物理内存的默认大小为
128MiB（可用地址范围为 `[0x80000000,0x88000000)`），它可以通过 -m 选项进行配置。

启动流程则可以分为三个阶段（以下地址为 Qemu 模拟的物理内存地址)：
1. 第一个阶段由固化在 Qemu 内的一小段汇编程序（固件)负责
  * Qemu 实际执行的第一条指令位于物理地址 0x1000 （PC = 0x1000）
  * Qemu 执行一段指令，然后跳到 0x80000000，计算机控制权移交给下一阶段
2. 第二个阶段由 bootloader 负责
    * bootloader 负责对计算机进行一些初始化工作，然后跳转到下一阶段软件的入口，把计算机控制权移交给下一阶段
    * 下阶段的入口地址可以是一个预先约定好的固定的值，也有可能是在 bootloader 运行期间才动态获取到的值
        * RustSBI 约定为固定的 0x80200000，需保证 0x80000000 正好为 bootloader 的第一条指令
        * 因此，bootloader `rustsbi-qemu.bin` 文件的内容必须加载到 `0x80000000` 开始的位置上
3. 第三个阶段则由内核镜像负责
  * 将内核镜像预先加载到 0x80200000 开头的区域上，从而获取到计算器的控制权

手动调整内存布局，让内核的首个指令从 0x80200000 开始：

```nasm
; 内核的内存布局 entry.asm
    .section .text.entry  ; 内核的第一个地址
    .globl _start         ; 入口点
_start:
    la sp, boot_stack_top ; 调整栈指针指向栈顶
    call rust_main        ; 运行内核

    .section .bss.stack   ; 分配程序的栈
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16      ; 分配栈的大小，这里的 4096 与链接脚本的 4K 对齐呼应
    .globl boot_stack_top ; 该程序的栈顶
boot_stack_top:
```

```nasm
OUTPUT_ARCH(riscv)
ENTRY(_start) ; 入口点
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;

    ...

    .text : {
        *(.text.entry)   ; 确保内核的指令位于其他指令之前
        *(.text .text.*)
    }

    ...

    .bss : {
        *(.bss.stack)    ; 手动控制栈的位置
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    ...
}
```

`sbss` 是一个符号，用来表示 .bss 中，除 .bss.stack 之外的 .bss 段的起点。

当然上面的脚本在这里并不完整，它有一个对应的 `ebss` 开表示 .bss 的结束。

这里排除掉 .bss.stack 区域的原因是，内核代码会将 sbss 到 ebss 的区域进行零初始化，但不对
.bss.stack 区域进行初始化。栈上的数据总是被覆盖使用的，无需使用前初始化。
