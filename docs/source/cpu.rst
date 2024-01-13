CPU Verification
================

.. contents:: Table of Contents


Importance
------------

CPU 作为中央处理器，在SoC中的重要性是显而易见的。
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

	RV32MFDA 是 RISC-V 的标准扩展，它们与 RV32I 统称为 RV32G（G 代表 general）。
	如果你注意过 ``RISC-V gcc`` 中的 ``-march`` 选项，你可能会发现它形如 ``rv32imac_zba_zbb_zbs_zbc_zicsr_zifencei``。
	这是因为，除了标准扩展之外，还有其他可选扩展，例如 ``zb*`` 指的是 “B” 标准扩展（Bit-manipulation）。

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
CSR 记录了 CPU 当前的状态信息，因此对于仿真或者流片后验证都十分重要。

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

.. note::

	在程序员视角下，32个GPR有不同于 x0 ~ x31 的名称，这被称为 ABI （Application Binary Interface）。


以下是一段汇编

.. code-block:: assembly

	.text 			# 指示符：进入代码节
	.align 2 		# 指示符：将代码按 2^2 字节对齐
	.globl main 		# 指示符：声明全局符号 main
	main: 			# main 的开始符号
	addi sp,sp,-16 		# 分配栈帧
	sw ra,12(sp) 		# 保存返回地址
	lui a0,%hi(string1) 	# 计算 string1
	addi a0,a0,%lo(string1) # 的地址
	lui a1,%hi(string2) 	# 计算 string2
	addi a1,a1,%lo(string2) # 的地址
	call printf 		# 调用 printf 函数
	lw ra,12(sp) 		# 恢复返回地址
	addi sp,sp,16 		# 释放栈帧
	li a0,0 		# 装入返回值 0
	ret 			# 返回
	.section .rodata 	# 指示符：进入只读数据节
	.balign 4 		# 指示符：将数据按 4 字节对齐
	string1: 		# 第一个字符串符号
	.string "Hello, %s!\n" 	# 指示符：以空字符结尾的字符串
	string2: 		# 第二个字符串符号
	.string "world" 	# 指示符：以空字符结尾的字符串



以英文句号开头的命令称为汇编器指示符（assembler directives）。
这些命令作用于汇编器，而非由其翻译的代码，具体用于通知汇编器在何处放置代码和数据、指定程序中使用的代码和数据常量等。

.. note::

	汇编器生成的文件为 ELF（Executable and Linkable Format，可执行可链接格式）[TIS Committee 1995] 标准格式目标文件。

Linker
^^^^^^^^^^^^^^^

链接器允许分别编译和汇编各文件，故只改动一个文件时无需重新编译所有源代码。
链接器把新目标代码和已有机器语言模块（如函数库）“拼接” 起来，即编辑目标文件中所有 “跳转并链接（``jal``）” 指令的链接目标。
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

`Spike <https://github.com/riscv-software-src/riscv-isa-sim>`__ 是一个开源的 RISC-V ISA 仿真器。
它通过软件来模拟 CPU 指令的行为，属于行为级的仿真，速度较快。
我们通常认为 ISS 运行的结果是正确的。

RTL Simulator
#####################

`Verilator <https://www.veripool.org/verilator>`__ 是一个开源的 Verilog/SystemVerilog 仿真器。
它将 RTL 编译为 C++ 或 SystemC 后再运行仿真，结果是周期精准的，但速度较慢。

Environment
##################

`RISCV-DV <https://github.com/chipsalliance/riscv-dv>`__ 是一个随机的指令生成器，它可以给待测试的模块提供验证环境。

Methodology
^^^^^^^^^^^^^^^^

Differential Testing
##################

进行 DiffTest 需要提供一个和 DUT（Design Under Test，测试对象）功能相同但实现方式不同的 REF（Reference，参考实现），然后让它们接受相同的有定义的输入，观测它们的行为是否相同。
在 CPU 验证中 DUT 为 RTL 仿真的结果，REF 为 ISS 仿真的结果。

Regression Testing
################

为了保证加入的新功能没有影响到已有功能的实现, 还需要重新运行测试用例，这个过程称为回归测试。
RISC-V 有多种回归测试的用例：

- `RISC-V Compliance <https://github.com/lowRISC/riscv-compliance>`__

- `RISC-V Tests <https://github.com/riscv-software-src/riscv-tests>`__

- `RISC-V Architecure Tests <https://github.com/riscv-non-isa/riscv-arch-test>`__

.. note::

	通过测试并不意味着设计符合 RISC-V 架构。这些只是基本的测试，检查规范的重要方面，而不关注细节。

CVA6 Example
----------------

`CVA6 <https://github.com/openhwgroup/cva6>`__ 是一个经过流片验证的开源 RISC-V CPU。
我们以该 CPU 为例，介绍如何仿真开源的 CPU。

.. note::

	如没有特别说明，默认运行环境为 Linux。
	Linux 下很多操作都是在终端（terminal）中进行，终端中运行的是 shell，Ubuntu 默认的 shell 为 bash。
	命令行操作有一定的学习成本，但请你一定坚持。
	我们会尽可能解释接下来的命令行操作，但绝大部分基础的内容仍需要你自行学习。


Setup
^^^^^^^^^^^^

1. 克隆仓库。

.. code-block:: shell

	$ git clone https://github.com/openhwgroup/cva6.git
	$ cd cva6
	$ git submodule update --init --recursive

.. note::

	我们使用 ``<cva6>`` 代指该项目的根目录。
	例如你的 ``cva6`` 项目位于 ``/home/user/cva6``，则 ``<cva6> == /home/user/cva6``。

.. Important::

	Git 是最流行的代码版本管理工具，著名的 Github 就是依托于 Git 建立的。
	学习如何使用 Git 是基本功，任何开源项目都会用到它。
	因此，在继续下一步之前，强烈建议理解该步骤中 ``git`` 的行为。

2. 安装 GCC 工具链。

.. code-block:: shell

	$ cd util/gcc-toolchain-builder
	$ export RISCV=<your desire RISC-V toolchain directory>
	$ sudo apt-get install autoconf automake autotools-dev curl git libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool bc zlib1g-dev
	$ sh get-toolchain.sh
	$ sh build-toolchain.sh $RISCV

你需要将 ``<your desire RISC-V toolchain directory>`` 换成一个真实的目录，它可以没有被创建，例如 ``/home/user/cva6/riscv-toolchain``。

.. note::

	实际上 ``<cva6>/util/gcc-toolchain-builder>`` 中有 ``README.md``，你可以自行根据其内容安装 GCC 工具链，我们也推荐你这么做，因为99%开源项目并没有本教程这样的保姆式文档。


.. Important::

	``export`` 指令是非常常见的 shell 指令，它为 shell 创建了环境变量（environmnet variable）。
	如果你不确定你是否真的创建了该变量，可以在 shell 中输入 ``echo $RISCV``，输出应该和你所设置的值一致。
	强烈建议你去了解常见的环境变量以及其作用，例如 ``PATH``，这对理解 shell 来说很重要。

3. 安装必要的包。

.. code-block:: shell

	$ sudo apt-get install help2man device-tree-compiler

4. 安装 Python 的环境依赖。

.. code-block:: shell

	$ cd <cva6>
	$ pip3 install -r verif/sim/dv/requirements.txt

.. Important::

	我们非常建议你安装 `miniconda` 用来管理 Python 的环境。
	Python 不同版本之间并不兼容，因此最好每个项目都有一个独立的 Python 环境。

5. 安装 Spike 和 Verilator。

.. code-block:: shell

	$ export DV_SIMULATORS=veri-testharness,spike
	$ bash verif/regress/smoke-tests.sh

在运行这条指令之前，请先查看该脚本的内容，试图理解这个脚本的行为。
请参考 `CVA6 Repo Issue 1757 <https://github.com/openhwgroup/cva6/issues/1757>`__，理解并修改对应的脚本。
如果你安装成功，你会在 ``<cva6>/tools`` 路径下发现 Spike 和 Verilator 的文件夹。
在此之后，你应该会发现 ``<cva6>/verif/regress/smoke-tests.sh`` 会报出 Error，这是因为环境变量设置的原因，你可以查看 shell 中的输出文本来定位具体是哪个环境变量。

如果你并不想 Debug，那么请在运行这条指令之前先运行 ``source verif/sim/setup-env.sh``。

.. Hint::

	如果你发现有时候运行 ``<cva6>/verif/regress/smoke-tests.sh`` 会报环境变量没有设置的问题，你可以研究一下 ``bash script.sh``，``sh script.sh``，``./script.sh`` 和 ``source script.sh`` 之间的联系和区别。
	然后再研究 ``export VAR=xx`` 和 ``VAR=xx`` 的区别。
	理解了上述两个区别之后，你就能明白为什么有时候环境变量丢失了。

6. 运行回归测试。

.. code-block:: shell
	
	$ export DV_SIMULATORS=veri-testharness,spike
	$ bash verif/regress/dv-riscv-arch-test.sh

你应该会发现 ``<cva6>/verif/regress/smoke-tests.sh`` 不仅安装了仿真器，还安装了许多测试用例。
在 ``<cva6>/verif/regress`` 目录下，有很多回归测试的脚本，这些都可以运行。
我们建议你在运行回归测试之前，先了解脚本跑了什么指令，这对之后自定义测试用例有很大帮助。

Standalone Simulation
^^^^^^^^^^^^^^^^

如果你看过回归测试的脚本，很容易就发现 CVA6 Core 的回归测试是通过多次调用 ``<cva6>/verif/sim/cva6.py`` 来完成的。
我们自己写的 C 代码也需要通过 ``<cva6>/verif/sim/cva6.py`` 来进行 DiffTest。
CVA6 支持很多的仿真器，因此我们需要指定比较的两个仿真器。
一般而言，我们使用 Spike 和 Verilator，指定方式为添加环境变量：``export DV_SIMULATORS=veri-testharness,spike``。


.. Hint::

	如果你想知道 ``<cva6>/verif/sim/cva6.py`` 到底运行了什么，你可以在运行该文件时试着添加 ``--debug <your debug log output directory>``。

你可以在任意路径下创建你自定义的 C 代码，例如 ``<custom path>/test.c``。
接下来，你只需要进入 ``cva6.py`` 所在的路径并运行该文件即可。

.. code-block:: shell

	$ cd <cva6>/verif/sim
	$ python cva6.py --target cv32a60x --iss=$DV_SIMULATORS --iss_yaml=cva6.yaml --c_tests <custom path>/test.c --linker=../tests/custom/common/test.ld --gcc_opts="-static -mcmodel=medany -fvisibility=hidden -nostdlib -nostartfiles -g ../tests/custom/common/syscalls.c ../tests/custom/common/crt.S -lgcc -I../tests/custom/env -I../tests/custom/common"

这个 python 文件会进行如下5件事情：

1. 你之前安装的 riscv-none-elf-gcc 会将 ``test.c`` 编译成一个对象文件（``test.o``），它包含了源代码编译后的机器代码，但还没有被链接成可以执行的程序。如果你想查看你所写的 C 程序对应的汇编代码，你可以通过 ``riscv-none-elf-objdump -d test.o`` 生成该对象文件的反汇编文件（disassembly）。

2. riscv-none-elf-objcopy 会把 ``test.o`` 转换为一个二进制文件 ``test.bin``，这个二进制文件可以被直接加载到内存中执行。

3. 调用 Verilator 和仿真环境，加载二进制文件，记录仿真过程，输出到 ``<verilator output path>/test.csv``。

4. 调用 Spike 和仿真环境，加载二进制文件，记录仿真过程，输出到 ``<spike output path>/test.csv``。

5. 将 Verilator 和 Spike 生成的 CSV 文件进行比较，输出测试结果。

.. Important::

	本小节中各种文件的路径请根据 shell 中的输出来寻找。
	同时，我们强烈推荐你了解仿真过程中 GCC，Verialtor，Spike 所接受参数的意义。




