FPGA Implementation
=========

.. contents:: Table of Contents

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

Naive Version
#####################

下面是一个名为 ``bootrom.S`` 的汇编语言文件，它包含了一个简单的 bootloader。

.. code-block::

   .section .text.start, "ax", @progbits
   .globl _start
   _start:
     li s0, 1
     slli s0, s0, 31
     csrr a0, mhartid
     la a1, _dtb
     jr s0
   
   .section .text.hang, "ax", @progbits
   .globl _hang
   _hang:
     csrr a0, mhartid
     la a1, _dtb
   1:
     wfi
     j 1b
   
   .section .rodata.dtb, "a", @progbits
   .globl _dtb
   .align 5, 0
   _dtb:
   .incbin "ariane.dtb"

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

   文本段（Text Segment）：包含程序的机器代码。

   数据段（Data Segment）：包含程序的全局变量和静态变量。

   堆（Heap）：用于动态内存分配，如 malloc、new 等操作。

   栈（Stack）：用于存放函数调用的局部变量和返回地址。

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

``la a1, _dtb`` 这行代码将 _dtb 标签的地址加载到 a1 寄存器。
_dtb 标签通常指向设备树二进制（DTB）文件的位置，这个文件描述了硬件的配置和布局。

``jr s0`` 这行代码跳转到 s0 寄存器指向的地址。在这个例子中，这个地址应该是 DRAM_BASE，也就是系统的主内存的基地址。

3. 定义 ``_hang`` 标签以及其对应的函数。

.. code-block::

   .section .text.hang, "ax", @progbits
   .globl _hang
   _hang:
     csrr a0, mhartid
     la a1, _dtb
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

4. 定义了 _dtb 标签，即设备树二进制文件（DTB）的位置。

.. code-block::

   .section .rodata.dtb, "a", @progbits
   .globl _dtb
   .align 5, 0
   _dtb:
   .incbin "ariane.dtb"

这个节用于存储只读数据，如常量和字符串字面量。
.rodata 的 "ro" 是 "read-only" 的缩写。
.rodata 节的内容在编译时就已经确定，且在程序运行时不会改变。
但与 .text 节不同的是，.rodata 节的内容不是用来执行的代码，而是用来读取的数据。

``.align 5, 0`` 这行代码将下一行的代码对齐到 2 的 5 次方（也就是 32）字节边界。如果当前的位置不是 32 字节边界，那么会插入 0 直到达到 32 字节边界。

.. attention::

   我们流片的 bootloader 不需要设备树。

为了能够成功解析 ``bootrom.S`` 中符号的地址，我们还需要自定义链接器脚本（linker script） ``test.ld``。

.. code-block::

   /*----------------------------------------------------------------------*/
   /* Setup                                                                */
   /*----------------------------------------------------------------------*/
   
   /* The OUTPUT_ARCH command specifies the machine architecture where the
      argument is one of the names used in the BFD library. More
      specifically one of the entires in bfd/cpu-mips.c */
   
   OUTPUT_ARCH( "riscv" )
   ENTRY(_start)
   
   /*----------------------------------------------------------------------*/
   /* Sections                                                             */
   /*----------------------------------------------------------------------*/
   
   SECTIONS
   {
   
     /* text: test code section */
     . = 0x80000000;
     .text.init : { *(.text.init) }
   
     . = ALIGN(0x1000);
     .tohost : { *(.tohost) }
   
     . = ALIGN(0x1000);
     .text : { *(.text) }
   
     /* data segment */
     .data : { *(.data) }
   
     .sdata : {
       __global_pointer$ = . + 0x800;
       *(.srodata.cst16) *(.srodata.cst8) *(.srodata.cst4) *(.srodata.cst2) *(.srodata*)
       *(.sdata .sdata.* .gnu.linkonce.s.*)
     }
   
     /* bss segment */
     .sbss : {
       *(.sbss .sbss.* .gnu.linkonce.sb.*)
       *(.scommon)
     }
     .bss : { *(.bss) }
   
     /* thread-local data segment */
     .tdata :
     {
       _tdata_begin = .;
       *(.tdata)
       _tdata_end = .;
     }
     .tbss :
     {
       *(.tbss)
       _tbss_end = .;
     }
   
     /* End of uninitalized data segement */
     _end = .;
   }

``OUTPUT_ARCH( "riscv" )`` 指定了输出的目标架构为 RISC-V。
``ENTRY(_start)`` 指定了程序的入口点为 _start。

``SECTIONS`` 是链接脚本的一个命令，它用于定义程序的内存布局。
在这个命令中，可以定义多个段（section），每个段都有一个名字和一个地址。

``. = 0x80000000;`` 将当前位置设置为 0x80000000。
``.text.init : { *(.text.init) }`` 定义了一个名为 .text.init 的段，它包含所有 .text.init 段的内容。
``*(.text.init)`` 表示将所有 .text.init 段的内容放在这里。

``. = ALIGN(0x1000);`` 将当前位置对齐到 0x1000 的边界。

.. note::

   更多有关 linker script 的信息，请你查阅 `The GNU linker <https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_node/ld_toc.html>`__ 。

C-compatiable Version
####################

如果我们想在 bare-metal 的 RISC-V CPU 上兼容 C 代码编译出来的二进制文件，那么所需要的 bootloader 更为复杂。

.. Tip::

   你可以参考 `这篇教程 <https://twilco.github.io/riscv-from-scratch/2019/04/27/riscv-from-scratch-2.html>`__ ，自己动手在 RISC-V CPU 上运行 C 代码。

C Runtime Initialzation
***********************

C 运行时文件 ``<cva6>/verif/bsp/crt0.S`` 提供 _start 函数，该函数是程序的入口点并执行以下任务：

- 初始化全局和堆栈指针。
- 将 ``vector_table`` 的地址存储在 ``mtvec``中，设置低两位为“0x1”以选择向量中断模式。
- 将 BSS 部分清零。
- 调用 C 构造函数的初始化并设置要调用的析构函数出口。它们在 C++ 中被广泛使用，但在 C 语言中并不常见。
- 将 ``argc`` 和 ``argv`` 归零（堆栈未初始化，因此将它们归零防止未初始化的值可能会导致程序的结果与预期的参考结果不匹配）。
- 调用 ``main`` 函数。
- 如果 ``main`` 函数返回，则调用 ``exit``。

.. Tip::

   在 RISC-V 架构中，``mtvec`` 是一个特殊的寄存器，它用于存储中断向量表的地址。
   当最低两位为 0x2 时，处理器会进入向量中断模式。
   中断向量表是一个包含了处理各种中断的函数地址的表，当发生中断时，处理器会根据 ``mtvec`` 寄存器中的地址找到中断向量表，然后跳转到相应的函数去处理中断。
   这与直接模式不同，在直接模式下，所有的中断都会被送到同一个处理函数。

.. Tip::

   "BSS" 是 "Block Started by Symbol" 的缩写，它是程序内存布局中的一个部分，用于存储程序中未初始化的全局变量和静态变量。
   在程序开始执行之前，BSS 段中的所有变量都需要被设置为零。
   这样做的目的是确保所有的未初始化的全局变量和静态变量都有一个确定的初始值（即零），这可以避免程序在运行时遇到未定义的行为。

接下来是 ``crt0.S`` 的分段解析。

1. 定义程序入口点。

.. code-block::

   /* Make sure the vector table gets linked into the binary.  */
   .global vector_table
   
   /* Entry point for bare metal programs */
   .section .text.start
   .global _start
   .type _start, @function

``.type _start, @function`` 这条指令将 ``_start`` 符号的类型设置为函数。
这对于调试和反汇编工具来说是有用的，它们可以通过这个信息更好地理解 ``_start`` 符号的用途。

2. 初始化全局和栈指针。

.. code-block::

   _start:
   /* initialize global pointer */
   .option push
   .option norelax
   1:	auipc gp, %pcrel_hi(__global_pointer$)
       addi  gp, gp, %pcrel_lo(1b)
   .option pop
   
   /* initialize stack pointer */
       la sp, __stack_end

   /* initialize stack pointer */
   	la sp, __stack_end

``.option norelax`` 这条指令关闭了汇编器的 relax 功能。
在 RISC-V 汇编语言中，relax 功能可以自动优化一些指令序列，使得它们更加紧凑和高效。
但在这段代码中，我们需要关闭 relax 功能，以确保 ``auipc`` 和 ``addi`` 两条指令不会被优化掉。

``.option push`` 和 ``.option pop`` 这两条指令用于保存和恢复汇编器的选项。
在这段代码中，它们用于保存和恢复 norelax 选项的状态。

.. note::

   在 RISC-V 汇编语言中，.option 是一个指令，用于设置汇编器的选项。
   这些选项可以影响汇编器的行为。

``auipc gp, %pcrel_hi(__global_pointer$)`` 和 ``addi gp, gp, %pcrel_lo(1b)`` 这两条指令用于初始化全局指针 gp。
``auipc`` 指令将 ``__global_pointer$`` 的高 20 位加到程序计数器 pc 上，然后将结果存储到 gp 中。
``addi`` 指令将 ``__global_pointer$`` 的低 12 位加到 gp 上，然后将结果存储到 gp 中。
这样，gp 就被设置为了 __global_pointer$ 的地址。

.. note::

   ``__global_pointer$`` 是一个特殊的符号，它通常用于优化全局变量和静态变量的访问。
   在 RISC-V 指令集中，大部分指令只能处理较小的立即数（即常数）。
   如果你需要访问一个全局变量或静态变量，而它的地址超出了这个范围，那么你需要使用多条指令来计算这个地址。
   这会使得代码变得复杂，并可能降低性能。

   为了解决这个问题，RISC-V 引入了全局指针（Global Pointer，简称 GP）。
   GP 是一个寄存器，它的值通常设置为靠近全局变量和静态变量的一个地址。
   这样，你就可以使用一条指令，通过 GP 加上一个小的偏移量来访问这些变量。

   ``__global_pointer$`` 就是 GP 的值。
   在链接时，链接器会计算出一个合适的值，然后将这个值赋给 ``__global_pointer$``。
   在程序开始执行时，启动代码会将 ``__global_pointer$`` 的值加载到 GP 寄存器中。

.. note::

   ``%pcrel_hi`` 是一个伪指令，用于获取一个符号相对于当前指令的高 20 位地址。
   RISC-V 指令集中的 ``auipc`` 指令可以将一个 20 位的立即数（即常数）加到程序计数器（PC）上，然后将结果存储到一个寄存器中。
   但是，这个立即数必须是硬编码在指令中的，你不能直接使用一个符号的地址作为这个立即数。

   为了解决这个问题，RISC-V 汇编语言提供了 ``%pcrel_hi`` 伪指令。
   你可以在 ``auipc`` 指令中使用 ``%pcrel_hi(symbol)``。
   汇编器会自动计算出 symbol 相对于当前指令的高 20 位地址，然后将这个地址作为 ``auipc`` 指令的立即数。

   同样，``%pcrel_lo`` 是一个伪指令，用于获取一个符号相对于前一条 ``auipc`` 指令的低 12 位地址。

   RISC-V 指令集中的 `auipc` 指令可以将一个 20 位的立即数（即常数）加到程序计数器（PC）上，然后将结果存储到一个寄存器中。
   然后，你可以使用 `addi` 指令，将一个 12 位的立即数加到这个寄存器上，从而得到一个完整的 32 位地址。

``la sp, __stack_end`` 用于初始化栈指针 sp。``la`` 是 "load address" 的缩写，它将 ``__stack_end`` 的地址加载到 sp 中。
这样，sp 就被设置为了栈的顶部。

.. note::

   栈顶（Stack Top）和栈底（Stack Bottom）是描述栈结构的两个术语。

   栈顶：这是栈中最后一个放入的元素所在的位置。
   新元素总是被放在栈顶，也总是从栈顶被取出。
   在大多数系统中，栈顶的地址是动态变化的，因为新的元素被压入栈或从栈中弹出时，栈顶的位置会相应地移动。

   栈底：这是栈中第一个放入的元素所在的位置。
   在大多数系统中，栈底的地址在程序运行期间是固定不变的。

3. 设置中断向量表的地址。

.. code-block::

   /* set vector table address */
   la a0, __vector_start
   ori a0, a0, 1 /*vector mode = vectored */
   csrw mtvec, a0

``ori a0, a0, 1`` 将 a0 寄存器的值与 1 进行或运算，然后将结果存储回 a0 中。
这条指令的目的是设置中断向量模式为 vectored。

``srw mtvec, a0`` 将 a0 寄存器的值写入 mtvec 控制状态寄存器。

4. 将 BSS 部分清零。

.. code-block::

   /* clear the bss segment */
   la a0, _edata
   la a2, _end
   sub a2, a2, a0
   li a1, 0
   call memset

``la a0, _edata`` 将 _edata 的地址加载到寄存器 a0 中。_edata 是一个符号，通常在链接脚本中定义，表示已初始化数据段（即 .data 段）的结束地址，也就是 BSS 段的开始地址。

``la a2, _end`` 这条指令将 _end 的地址加载到寄存器 a2 中。_end 是一个符号，通常在链接脚本中定义，表示 BSS 段的结束地址。

``sub a2, a2, a0`` 这条指令将 a0 寄存器的值从 a2 寄存器的值中减去，然后将结果存储回 a2 中。这样，a2 寄存器中就存储了 BSS 段的大小。

``li a1, 0`` 这条指令将 0 加载到寄存器 a1 中。这是因为我们要将 BSS 段的内容清零。

``call memset`` 这条指令调用 ``memset`` 函数，将 BSS 段的内容清零。
在这个调用中，a0 寄存器中的值作为第一个参数，表示要清零的内存区域的开始地址；a1 寄存器中的值作为第二个参数，表示要设置的值；a2 寄存器中的值作为第三个参数，表示要清零的内存区域的大小。

5. 调用 C 构造函数的初始化并设置要调用的析构函数出口。

``la a0, __libc_fini_array`` 将 ``__libc_fini_array`` 的地址加载到寄存器 a0 中。``__libc_fini_array`` 是一个函数，通常在 C 库中定义，它会调用所有全局和静态对象的析构函数。

``call atexit`` 这条指令调用 ``atexit`` 函数，将 ``__libc_fini_array`` 函数注册为一个退出处理函数。
``atexit`` 是一个标准的 C 库函数，它可以注册一个函数，这个函数会在 main 函数返回或 exit 函数被调用时执行。
在这个调用中，a0 寄存器中的值作为参数，表示要注册的函数的地址。

``call __libc_init_array`` 这条指令调用 ``__libc_init_array`` 函数。
``__libc_init_array`` 是一个函数，通常在 C 库中定义，它会调用所有全局和静态对象的构造函数。

.. attention::

   这段代码的作用是在程序开始执行前调用所有全局和静态对象的构造函数，以及在程序结束时调用所有全局和静态对象的析构函数。
   这是 C++ 程序的一部分初始化和清理过程，但在 C 程序中通常不需要这个过程。

6. 将 ``argc`` 和 ``argv`` 归零，调用 ``main``，``exit``。

.. code-block::

   // Initialize these variables to 0. Cannot use argc or argv
   // since the stack is not initialized
   	li a0, 0
   	li a1, 0
   	li a2, 0
   
   	call main
   	tail exit

``li a0, 0``、``li a1, 0`` 和 ``li a2, 0`` 将寄存器 a0、a1 和 a2 的值设置为 0。

``call main`` 这条指令调用 main 函数。
main 函数是 C 和 C++ 程序的入口点，程序的执行从这里开始。

``tail exit`` 调用 exit 函数并结束当前的函数。
exit 函数是一个标准的 C 库函数，它会结束程序的执行，并将 main 函数的返回值作为程序的退出状态返回给操作系统。
在这个调用中，因为 main 函数的返回值会被存储在 a0 寄存器中，所以 exit 函数会将 a0 寄存器中的值作为程序的退出状态。

.. Important::

   在 bare-metal（无操作系统）环境中，exit 函数的行为需要由你自己定义。
   在这种环境中，没有操作系统来接管程序结束后的清理工作，所以你需要自己决定 exit 函数应该做什么。
   一种常见的做法是让 exit 函数进入一个无限循环。
   这样，当 exit 函数被调用时，程序会停止执行任何有意义的操作，但 CPU 仍然在运行。
   

7. 定义全局函数 ``_init`` 和 ``_fini``，并且设置大小。

.. code-block::

   .size  _start, .-_start
   
   .global _init
   .type   _init, @function
   .global _fini
   .type   _fini, @function
   _init:
   _fini:
    /* These don't have to do anything since we use init_array/fini_array. Prevent
       missing symbol error */
   	ret
   .size  _init, .-_init
   .size _fini, .-_fini

在 ``_init`` 和 ``_fini`` 两个函数的定义中，只有一条 ``ret`` 指令，这意味着这两个函数什么也不做，直接返回。

``.size _init, .-_init`` 和 ``.size _fini, .-_fini`` 设置了 ``_init`` 和 ``_fini`` 函数的大小。
在这里，``.`` 表示当前位置，``.-_init`` 和 ``.-_fini`` 分别表示从 ``_init`` 和 ``_fini`` 的开始位置到当前位置的距离，也就是 ``_init`` 和 ``_fini`` 函数的大小。

.. hint::

   这段代码的注释说明，由于我们使用了 ``init_array/fini_array``，所以 ``_init`` 和 ``_fini`` 函数不需要做任何事情。
   这两个函数的存在只是为了防止链接时出现缺少符号的错误。
   
下面是完整的 ``crt0.S`` 文件的内容。

.. code-section::

   /* Make sure the vector table gets linked into the binary.  */
   .global vector_table
   
   /* Entry point for bare metal programs */
   .section .text.start
   .global _start
   .type _start, @function
   
   _start:
   /* initialize global pointer */
   .option push
   .option norelax
   1:	auipc gp, %pcrel_hi(__global_pointer$)
   	addi  gp, gp, %pcrel_lo(1b)
   .option pop
   
   /* initialize stack pointer */
   	la sp, __stack_end
   
   /* set vector table address */
   	la a0, __vector_start
   	ori a0, a0, 1 /*vector mode = vectored */
   	csrw mtvec, a0
   
   /* clear the bss segment */
   	la a0, _edata
   	la a2, _end
   	sub a2, a2, a0
   	li a1, 0
   	call memset
   
   /* new-style constructors and destructors */
   	la a0, __libc_fini_array
   	call atexit
   	call __libc_init_array
   
   /* call main */
   //	lw a0, 0(sp)                    /* a0 = argc */
   //	addi a1, sp, __SIZEOF_POINTER__ /* a1 = argv */
   //	li a2, 0                        /* a2 = envp = NULL */
   // Initialize these variables to 0. Cannot use argc or argv
   // since the stack is not initialized
   	li a0, 0
   	li a1, 0
   	li a2, 0
   
   	call main
   	tail exit
   
   .size  _start, .-_start
   
   .global _init
   .type   _init, @function
   .global _fini
   .type   _fini, @function
   _init:
   _fini:
    /* These don't have to do anything since we use init_array/fini_array. Prevent
       missing symbol error */
   	ret
   .size  _init, .-_init
   .size _fini, .-_fini

Interrupt and Exception Handling
***************************




System Calls
********************



Linker Script
*******************





`RISC-V Assembly Programmer's Manual <https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md>`__





   Bootloader在RISC-V CPU上运行Bare-metal软件时充当了初始化和准备阶段的角色。它负责确保硬件适当地配置和Bare-metal软件正确加载到内存中。一旦这些任务完成，Bootloader将控制权转交给Bare-metal软件，使其能够在已准备好的硬件环境中执行。
   
   二进制文件（.bin）通常是一个可执行文件，它包含了用于直接在硬件或操作系统上执行的机器代码。这些文件通常由编译器从源代码生成，然后可以直接被加载和执行。
   
   镜像文件（.img）通常是一个存储设备或文件系统的完整二进制复制。它包含了存储设备的所有内容，包括文件系统、文件、目录和元数据。镜像文件通常用于备份、恢复或在不同的设备或系统之间复制数据。在嵌入式系统开发中，镜像文件通常包含了完整的固件，包括引导加载程序、内核、应用程序和文件系统。
   
   
   
   C标准库中有一个exit函数，它会在程序执行完之后自动调用。




.. note::

   This section is under development.
