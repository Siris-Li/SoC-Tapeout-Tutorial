FPGA Implementation
=========

.. contents:: Table of Contents

Deployment
--------------

FPGA 原型验证的流程和 ASIC 的流程基本一致：

- 读取源文件。
- 逻辑综合（synthesis）。
- 添加管脚约束（constraint）。
- 布局布线（implementation）。
- 生成比特流文件（bitstream）。

不同阶段都可以进行对应的仿真：
- 综合前仿真（behavioral simulation）。
- 综合后仿真（post-synthesis behavioral simulation）。
- 布局布线后仿真（post-implementation behavioral simulation）。
- 比特流验证。

.. attention::

   只有最后的比特流验证是真实的物理实现结果验证，其他都属于软件仿真。

.. Tip::

   Vivado 综合时会进行优化，将一些信号重命名，这会导致仿真时无法找到对应的信号。
   为了解决这个问题，可以在源代码中实例化的模块或者定义的信号上添加 ``(* DONT_TOUCH = TRUE *)`` 属性，这样 Vivado 就不会对其进行优化。

Boot
----------------

Bare-metal
^^^^^^^^^^^^^^^^^

"Bare-metal"（裸机） 是一个术语，通常用于描述在嵌入式系统或计算机上运行的软件，该软件直接在硬件上运行，没有操作系统或其他软件层介入。
Bare-metal 软件是针对特定硬件平台编写的，它与硬件之间的交互是直接的，没有中间层，与之相对应的是操作系统。
Bare-metal 的一些重要特点和概念如下：

- 无操作系统：它直接管理硬件资源，包括处理器、内存、外设等，而不使用操作系统提供的抽象和服务。
- 硬件控制：Bare-metal 软件具有对硬件的细粒度控制。它可以直接操作寄存器、配置外设、设置时钟和中断等，以满足特定应用程序的需求。
- 性能和效率：由于没有操作系统的开销，Bare-metal 软件通常能够实现更高的性能和更低的延迟。这对于一些实时性要求高的应用程序非常重要。
- 嵌入式系统：Bare-metal 常用于嵌入式系统，如微控制器、嵌入式处理器等。这些系统通常需要小型、高效、快速响应的软件，因此 Bare-metal 非常适用。

Bare-metal 软件可用于各种应用，包括嵌入式控制、传感器数据采集、嵌入式网络设备、实时控制系统等。

Device Tree
^^^^^^^^^^^^^

`设备树 <https://devicetree-specification.readthedocs.io/en/stable/>`__ （Device Tree）是一种数据结构，用于描述硬件设备的组成和配置信息，特别是在嵌入式系统中。
设备树主要用于操作系统，以便在启动时了解硬件的配置和布局，从而能够正确地初始化和管理硬件设备。
在裸机环境中，CPU 通常不需要设备树。
这是因为，硬件的配置通常会直接编码到程序中，由程序直接管理，不需要设备树来描述硬件的配置。

.. attention::

   我们流片的 bootloader 不需要设备树。

Bootloader
^^^^^^^^^^^^^^^

引导加载程序（Bootloader）是计算机启动时运行的一段小程序。
它的主要任务是加载操作系统内核到内存，并将控制权交给内核。
当 CPU 上电启动时，CPU 会从一个固定的地址（通常是 ROM 或者固定的 RAM 地址）开始执行代码，这段代码就是引导加载程序。
引导加载程序通常只包含最基本的硬件初始化和内核加载功能。
在RISC-V处理器架构中，通常存在多个引导加载程序（Bootloader）阶段，包括零阶段引导加载程序（Zero Stage Bootloader）和一阶段引导加载程序（First Stage Bootloader）。

Zero Stage Bootloader
########################

零阶段引导加载程序通常是在处理器复位后直接运行的一小段代码。
它通常位于芯片内部的 BootROM 中，因为它需要非常快速地执行。
零阶段引导加载程序的主要任务是进行基本的硬件初始化和设置，以准备进一步的引导加载过程。
它可能会初始化内存控制器、设置栈指针、配置中断等，以便后续的引导加载程序能够正常运行。

First Stage Bootloader
######################

一阶段引导加载程序位于零阶段引导加载程序之后运行。
它通常位于可写的存储介质（如Flash存储器）中，而不是芯片内部的BootROM。
一阶段引导加载程序的主要任务是从存储介质中加载更复杂的引导加载程序，如二阶段引导加载程序（Second Stage Bootloader）或操作系统内核，到内存中并开始执行。
它可能还会进行更高级的硬件初始化，如初始化外部设备、加载驱动程序等。
这两个阶段的引导加载程序通常是为了实现引导过程的分层和模块化。
零阶段引导加载程序是最基本的初始化步骤，它保证了处理器在运行任何复杂引导加载程序之前处于一个合适的状态。
一阶段引导加载程序进一步构建在此基础上，负责加载更多的软件组件，最终启动操作系统或主应用程序。

Bootcode Generation
^^^^^^^^^^^^^^^^^^^^^^^^^

Assembly
#############

下面是一个名为 ``bootrom.S`` 的汇编语言文件，它包含了一个简单的 bootloader。

.. code-block::

   .section .text.start, "ax", @progbits
   .globl _start
   _start:
     li s0, 1
     slli s0, s0, 31
     csrr a0, mhartid
     jr s0
   
   .section .text.hang, "ax", @progbits
   .globl _hang
   _hang:
     csrr a0, mhartid
   1:
     wfi
     j 1b

接下来我们分段详细解释这个汇编代码的行为。

1. 定义 ``_start`` 标签，这是引导加载程序的入口点。

.. code-block::

   .section .text.start, "ax", @progbits
   .globl _start

- ``.section``：定义了一个新的节。
- ``.text``：这个节通常用于存储程序的代码，也就是 CPU 执行的指令。.text 节的内容在编译时就已经确定，且在程序运行时不会改变。因此，.text 节通常被设置为只读和可执行。
- ``.start``：这个节的名字。
- ``ax``：表示这个节是可分配的（a）并且可以包含代码（x）。
- ``@progbits``：表示这个节包含了程序的实际代码或数据，而不是其他一些信息，如未初始化的数据或调试信息。
- ``.globl _start``：这行代码声明了一个全局符号 _start。在链接过程中，全局符号可以被其他的对象文件引用。在大多数系统中，_start 是程序的入口点，也就是程序开始执行的地方。这通常是操作系统或引导加载程序在加载程序后首先调用的函数。

.. Hint::

   在链接器脚本或汇编语言中，“可分配”（allocatable）是一个属性，用来描述一个节（section）是否需要在程序的内存映像中分配空间。
   如果一个节被标记为“可分配”，那么在链接过程中，链接器会为这个节分配内存空间。
   在加载程序时，加载器会将这个节的内容加载到内存中。
   例如，包含程序代码或初始化的全局变量的节通常都是“可分配”的，因为这些代码和数据需要被加载到内存中，以便 CPU 可以执行或访问它们。
   相反，包含调试信息或符号表的节通常不是“可分配”的，因为这些信息只在链接或调试时需要，而在程序运行时并不需要加载到内存中。

.. Hint::

   内存映像（Memory Image）是一个术语，通常用来描述程序在内存中的布局和组织。
   当一个程序被加载到内存中执行时，它的代码、数据和其他资源会被放置在内存的特定位置。这些代码、数据和资源在内存中的布局就构成了这个程序的内存映像。
   内存映像通常包括以下几个部分：

   - 文本段（Text Segment）：包含程序的机器代码。
   - 数据段（Data Segment）：包含程序的全局变量和静态变量。
   - 堆（Heap）：用于动态内存分配，如 malloc、new 等操作。
   - 栈（Stack）：用于存放函数调用的局部变量和返回地址。

2. 定义 ``_start`` 函数。

.. code-block::

   _start:
     li s0, 1
     slli s0, s0, 31
     csrr a0, mhartid
     la a1, _dtb
     jr s0

``li s0, 1`` 这行代码将立即数 1 加载到寄存器 s0 中。
然后，``slli s0, s0, 31`` 这行代码将 s0 寄存器中的值左移 31 位。
这两行代码的组合效果等同于将 DRAM_BASE（0x8000_0000）加载到 s0 寄存器。

``csrr a0, mhartid`` 这行代码将 mhartid 控制和状态寄存器（CSR）的值读取到 a0 寄存器。
mhartid 寄存器包含了当前硬件线程的 ID。

``jr s0`` 这行代码跳转到 s0 寄存器指向的地址。在这个例子中，这个地址应该是 DRAM_BASE，也就是系统的主内存的基地址。

3. 定义 ``_hang`` 标签以及其对应的函数。

.. code-block::

   .section .text.hang, "ax", @progbits
   .globl _hang
   _hang:
     csrr a0, mhartid
   1:
     wfi
     j 1b

``wfi`` 这行代码执行了等待中断（Wait For Interrupt）指令。
这个指令会使处理器进入低功耗模式，直到接收到一个中断。

``j 1b`` 这行代码跳转到前面定义的 1 标签。
1b 是一个汇编标签，1 是标签的名字，b 表示向后查找。
在这个特定的情况下，``j 1b`` 使程序进入一个无限循环，直到接收到一个中断或者复位信号。

.. Hint::

   "向后跳转"和"向前跳转"是相对于当前执行位置的。
   "向后跳转"意味着跳转到之前的代码位置，"向前跳转"意味着跳转到后面的代码位置。

_hang 代码段通常只在出现错误或特殊情况时才会执行。
例如，如果在尝试跳转到主内存执行程序时发生错误，或者在特定的硬件事件（如电源管理事件）发生时，程序可能会跳转到 _hang 代码段。

.. Warning::

   TODO：``wfi`` 详细解释和执行机制。

   粗看下来，大致上是 CPU 识别到 ``wfi`` 指令之后，会控制 commit stage 不提交指令。
   此时，尽管 frontend 一直在取指令，但是并没有新的指令被提交，即 CPU 被暂停（halt）。

Linker Script
##################

为了能够成功解析 ``bootrom.S`` 中符号的地址，我们还需要自定义链接器脚本（linker script） ``linker.ld``。

.. code-block::

   SECTIONS
   {
       ROM_BASE = 0x10000; /* ... but actually position independent */
   
       . = ROM_BASE;
       .text.start : { *(.text.start) }
       . = ROM_BASE + 0x40;
       .text.hang : { *(.text.hang) }
   }

``SECTIONS`` 是链接脚本的一个命令，它用于定义程序的内存布局。
在这个命令中，可以定义多个段（section），每个段都有一个名字和一个地址。

``ROM_BASE = 0x10000`` 定义了一个名为 ROM_BASE 的符号，其值为 0x10000。这个符号通常用来表示程序的起始地址。

然后，``.`` 符号被设置为 ROM_BASE 的值。
在链接脚本中，``.`` 符号表示当前的地址计数器，也就是下一个将被分配的字节的地址。

接下来，定义了一个名为 .text.start 的段，这个段包含所有 .text.start 输入段的内容。
输入段通常来自于编译器生成的目标文件。
这个段被放置在当前的地址（即 ROM_BASE）。

然后，地址计数器增加 0x40，也就是说，下一个将被分配的字节的地址现在是 ROM_BASE + 0x40。

最后定义了一个名为 .text.hang 的段，这个段包含所有 .text.hang 输入段的内容。这个段被放置在当前的地址（即 ROM_BASE + 0x40）。

.. note::

   更多有关 linker script 的信息，请你查阅 `The GNU linker <https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_node/ld_toc.html>`__ 。

.. Hint::

   汇编文件和链接器脚本均参考 ``<cva6>/corev_apu/bootrom`` 中的文件。

Compile
######################

编译所用的指令如下：

.. code-block::

   riscv-none-elf-gcc -Tlinker.ld -Os -ggdb -march=rv64im -mabi=lp64 -Wall -mcmodel=medany -mexplicit-relocs bootrom.S -nostdlib -static -Wl,--no-gc-sections -o bootrom.elf

- ``-Tlinker.ld``：使用 linker.ld 文件作为链接脚本。链接脚本用于控制如何将各个代码和数据段映射到目标内存。
- ``-Os``：进行优化，以使生成的代码尽可能小。
- ``-ggdb``：生成可以被 GDB 调试器使用的调试信息。
- ``-march=rv64im``：指定目标架构为 RISC-V，具有 64 位地址空间和整数乘法和除法指令。
- ``-mabi=lp64``：指定目标 ABI（应用二进制接口）为 LP64，这意味着 long 和指针类型都是 64 位的。
- ``-Wall``：生成所有的警告信息。
- ``-mcmodel=medany``：指定代码模型为 medany，这意味着代码可以被加载到任何地址。
- ``-mexplicit-relocs``：生成显式的重定位信息。
- ``-nostdlib``：不链接标准库。
- ``-static``：生成静态链接的可执行文件。
- ``-Wl,--no-gc-sections``：在链接时不丢弃未使用的代码和数据段。

Boot Flow
######################

使用上述 bootloader，并将 CVA6 的启动地址指向 BootRom 的基地址，上电之后 CPU 便会顺序执行 bootloader 中的指令。
当运行到跳转指令时，CPU 会跳转至 SRAM 的基地址。
SRAM 中的数据在上电时被初始化为0，因此 CPU 识别到其为非法指令（illegal instruction），会抛出异常（exception），同时更新 ``mtval`` ``mepc`` ``mcause`` 等 CSR。
此时 CPU 会根据 ``mtvec`` 中的数据跳转至异常处理程序的基地址。
CVA6 指定了该地址为 ``boot_addr + 0x40``，在我们的 bootloader 中，异常处理程序被设置为了 ``wfi``（wait for interrupt）指令。

.. Hint::

   - ``mtval``：异常处理程序的基地址。
   - ``mepc``：异常指令的地址。
   - ``mcause``：CPU 异常的原因。

Debug
----------------

Tools
^^^^^^^^^^^^

GDB
#############

GDB 是 GNU 调试器（GNU Debugger）的缩写，是一个功能强大且广泛使用的开源调试工具。
GDB旨在帮助开发人员诊断和修复程序中的错误，在程序运行时提供功能丰富的调试和分析功能。

.. attention::

   我们需要使用 RISC-V 的 GDB，它的可执行文件全名为 ``riscv-none-elf-gdb``，应该位于 ``<riscv-gcc-toolchain>/bin`` 下。

.. Tip::

   如果你想查阅有关 OpenOCD 的使用方法，请参考 `官方文档 <https://www.eecs.umich.edu/courses/eecs373/readings/Debugger.pdf>`__ 。

OpenOCD
##############

OpenOCD（Open On-Chip Debugger）是一个开源项目，旨在提供针对嵌入式系统的调试、仿真和编程解决方案。
它可以与多种调试适配器和芯片配合使用，支持多种处理器架构和调试协议。

.. Tip::

   如果你想查阅有关 OpenOCD 的使用方法，请参考 `官方文档 <https://openocd.org/doc/pdf/openocd.pdf>`__ 。

JTAG Adapter
#################

OpenOCD 可以看作调试主机（Debug Host）所运行的一个软件，它一般通过主机的 USB 接口发送信号。
我们所实现的 SoC 对外的调试接口是 JTAG（joint Test Action Group，是一种用于测试集成电路的标准接口和协议）。
二者之间需要 JTAG Adapter 用于信号的格式转换。

我们所使用的 JTAG Adapter 中最关键的芯片称为 `FTDI <https://ftdichip.com/wp-content/uploads/2020/07/DS_FT232H.pdf>`__ （Future Technology Devices International），它负责输出 JTAG 信号。
连接到 PC 后，``lsusb`` 的输出中会有如下一条：

.. code-block::

   Bus <bus id> Device <device id>: ID 0403:6014 Future Technology Devices International, Ltd FT232H Single HS USB-UART/FIFO IC


Example
----------------

请参考 `CVA6 FPGA Verification <https://edge-training-soc.readthedocs.io/zh-cn/latest/fpga.html>`__ 。


.. note::

   This section is under development.
