CPU Verification
================

.. contents:: Table of Contents


Importance
------------

CPU 作为中央处理器，在SoC中的重要性是显而易见的.
虽然对于学术流片来说，CPU 大概率只是作为一个控制器来使用，但在工程实现中，我们不能忽视 CPU 的验证。
理想情况下，我们编写的程序会运行在 CPU 上，通过 CPU 去控制加速器等模块，因此我们与加速器的交互是通过 CPU 来完成的，必须保证 CPU 功能的正确性。

相比于加速器来说，CPU 的设计比较复杂，我们不太可能从头自己写一个 CPU，大概率是 Github 上开源的 RISC-V CPU 或者 ARM Cortex-M0/M4。
M0/M4 有完整成熟的集成开发工具链，RISC-V CPU 相比而言生态相对比较简陋，各个部分需要我们自己拼装验证。
考虑到目前学术界流片的 SoC 所使用的 CPU 基本上都是 RISC-V CPU，因此接下来我们会补充一些关于 RISC-V CPU 的必要知识并简单介绍 RISC-V CPU 的验证流程。
以流片为目标，我们至少需要了解 CPU 基本的架构以及和外界交互的机制。



RISC-V Privileged Instruction Set Architecture
--------------

RISC-V Instruction Set
^^^^^^^^^^^^^^^^^

RISC-V指令集的一个特点是模块化，其核心是一个名为 RV32I 的基础 ISA，可运行完整的软件栈。
RV32I 已冻结，永不改变，这为编译器开发者、操作系统开发者和汇编语言程序员提供了稳定的指令目标。
模块化特性源于可选的标准扩展，硬件可根据应用程序的需求决定是否包含它们。
这种模块化特性能设计出面积小、能耗低的 RISC-V 处理器，这对于嵌入式应用至关重要。
RISC-V 编译器得知当前硬件包含哪些扩展后，便可为该硬件生成最优代码。
一般约定将扩展对应的字母加到指令集名称之后，以指示包含哪些扩展。
例如，RV32IMFDA 在必选基础指令集（RV32I）上添加了乘法（RV32M），单精度浮点（RV32F）, 双精度浮点（RV32D）和原子指令（RV32A）扩展。

.. note::

    RV32IMFDA 是 RISC-V 的标准扩展，它们与 RV32I 统称为 RV32G（G 代表 general）。

Privilege
^^^^^^^^^^^^^^^^^

处理器的代码可以运行在不同等级的模式下，一般而言软件代码会默认运行在用户模式（*user mode*, U mode）下，该模式具有最低的权限。
除此以外，还有监管模式（*supervisor mode*, S mode）和机器模式（*machine mode*, M mode）。
操作系统一般运行在 S 模式下，而 M 模式则具有最高的特权，最重要的特性是拦截和处理异常（不寻常的运行时事件）。
在 M 模式下运行的代码能完全访问内存、I/O 和底层系统功能，这对启动和配置系统是必不可少的。
因此，M 模式是唯一一个所有标准 RISC-V 处理器都必须实现的特权模式。

.. note::

    实际上简单的 RISC-V 微控制器仅支持 M 模式，我们流片的 CPU 就属于这一类。


Control Status Register
^^^^^^^^^^^^^^^^^

体系结构的课程中一定会学习到寄存器堆（Register File），这些寄存器也被称为 GPR（General Purpose Register）。
实际上还有另一个“寄存器堆”——控制状态寄存器（CSR），它们被用来实现特权架构所带来的新特性，例如 :code:`mcause` 用于记录异常和中断的原因。
除了处理特权架构，还有一些 CSR 用于标识处理器特性或测量性能，例如 :code:`mcycle` 用于记录运行周期数。


Assembly
------------------

了解了 CPU 的基本架构之后，我们需要知道软件代码如何翻译成 CPU 可运行的指令，这个过程被称为编译（compiling）。
将 C 程序翻译成计算机可运行的机器语言程序需要四个经典步骤：

:code:`foo.c` --compiler--> :code:`foo.s` --assembler--> :code:`foo.o` --linker--> :code:`a.out` --loader--> CPU

.. note::

    这些步骤是概念上的，实际上会合并某些步骤来加速翻译过程。

Compiler & Assembler
^^^^^^^^^^^^^^^

编译器负责将高级语言转换成汇编，汇编器负责将汇编转换成机器码。
汇编器的作用不仅是用处理器可理解的指令生成目标代码，还支持一些对汇编语言程序员或编译器开发者有用的操作。
这类操作是常规指令的巧妙特例，称为伪指令。
最经典的例子为 :code:`nop`，它在RISC-V中由 :code:`addi x0, x0, 0` 实现。

以下是一段汇编::

	.text # 指示符：进入代码节
	.align 2 # 指示符：将代码按 2^2 字节对齐
	.globl main # 指示符：声明全局符号 main
	main: # main 的开始符号
	addi sp,sp,-16 # 分配栈帧
	sw ra,12(sp) # 保存返回地址
	lui a0,%hi(string1) # 计算 string1
	addi a0,a0,%lo(string1) # 的地址
	lui a1,%hi(string2) # 计算 string2
	addi a1,a1,%lo(string2) # 的地址
	call printf # 调用 printf 函数
	lw ra,12(sp) # 恢复返回地址
	addi sp,sp,16 # 释放栈帧
	li a0,0 # 装入返回值 0
	ret # 返回
	.section .rodata # 指示符：进入只读数据节
	.balign 4 # 指示符：将数据按 4 字节对齐
	string1: # 第一个字符串符号
	.string "Hello, %s!\n" # 指示符：以空字符结尾的字符串
	string2: # 第二个字符串符号
	.string "world" # 指示符：以空字符结尾的字符串

.. note::

	在程序员视角下，32个GPR有不同于 x0 ~ x31 的名称，这被称为 ABI （Application Binary Interface）。


以英文句号开头的命令称为汇编器指示符（assembler directives）。
这些命令作用于汇编器，而非由其翻译的代码，具体用于通知汇编器在何处放置代码和数据、指定程序中使用的代码和数据常量等。
其中上述汇编用到的指示符有::

	• .text——进入代码节。
	• .align 2——后续代码按 2^2 字节对齐。
	• .globl main——声明全局符号 “main”。
	• .section .rodata——进入只读数据节
	• .balign 4——数据节按 4 字节对齐。
	• .string "Hello, %s!\n"——创建以空字符结尾的字符串。
	• .string "world"——创建以空字符结尾的字符串。

.. note::

	汇编器生成的文件为 ELF（Executable and Linkable Format，可执行可链接格式）[TIS Committee 1995] 标准格式目标文件。

Linker
^^^^^^^^^^^^^^^

链接器允许分别编译和汇编各文件，故只改动一个文件时无需重新编译所有源代码。
链接器把新目标代码和已有机器语言模块（如函数库）“拼接” 起来，即编辑目标文件中所有 “跳转并链接” 指令的链接目标。
例如上述汇编有两个数据符号（``string1`` 和 ``string2``）和两个代码符号（``main`` 和 ``printf``）待确定。

根据链接的形式，可以将链接结果分为静态（static linking）和动态（dynamic linking）两种。
前者在程序运行前链接并加载所有库的代码，后者首次调用所需外部函数时才会将其加载并链接到程序中。


Loader
^^^^^^^^^^^^^^

运行一个程序时，加载器会将其加载到内存中，并跳转到它的起始地址。

.. note::

	如今的 “加载器” 就是操作系统。


Verification
------------------

Open-Sourced Tools
^^^^^^^^^^^^^^^^^^^

Instruction Set Simulator
######################

Spike <https://github.com/riscv-software-src/riscv-isa-sim> 是一个开源的 RISC-V ISA 仿真器。
它通过软件来模拟 CPU 指令的行为，属于行为级的仿真，速度较快。
我们通常认为 ISS 运行的结果是正确的。

RTL Simulator
#####################

Verilator <https://www.veripool.org/verilator> 是一个开源的 Verilog/SystemVerilog 仿真器。
它将 RTL 编译为 C++ 或 SystemC 后再运行仿真，结果是周期精准的，但速度较慢。

Environment
##################

RISCV-DV <https://github.com/chipsalliance/riscv-dv> 是一个随机的指令生成器，它可以给待测试的模块提供验证环境。

Methodology
^^^^^^^^^^^^^^^^

Differential Testing
##################

进行 DiffTest 需要提供一个和 DUT (Design Under Test, 测试对象) 功能相同但实现方式不同的 REF (Reference, 参考实现), 然后让它们接受相同的有定义的输入, 观测它们的行为是否相同。
DUT 为 RTL 仿真的结果，REF 为 ISS 仿真的结果。

Regression Testing
################

为了保证加入的新功能没有影响到已有功能的实现, 还需要重新运行测试用例，这个过程称为回归测试。

RISC-V 有多种回归测试的用例：

- RISC-V Compliance <https://github.com/lowRISC/riscv-compliance>。

- RISC-V Tests <https://github.com/riscv-software-src/riscv-tests>。

- RISC-V Architecure Tests <https://github.com/riscv-non-isa/riscv-arch-test>。

.. note::

	通过测试并不意味着设计符合 RISC-V 架构。这些只是基本的测试，检查规范的重要方面，而不关注细节。

CVA6 Example
----------------

CVA6 <https://github.com/openhwgroup/cva6> 是一个经过流片验证的开源 RISC-V CPU。
我们以该 CPU 为例，介绍如何仿真开源的 CPU。

.. note::

	Under active development!

