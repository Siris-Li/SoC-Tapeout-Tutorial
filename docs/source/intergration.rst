SoC Intergration
=======

.. contents:: Table of Contents


Overview
--------------

简要来说，计算机由存储器、运算器、控制器、输入设备和输出设备五部分组成， 其中运算器和控制器合称为中央处理器 (Central Processing Processor)，也就是CPU。 
冯·诺依曼结构以运算器为中心，输入输出设备与存储器之间的数据传送都需要经过运算器。
SoC是一种单片计算机系统解决方案，在单个芯片上集成了处理器、内存控制器、GPU以及硬盘、USB、网络等IO接口，用户在搭建计算机系统时只需要使用单个主要芯片即可。 
目前SoC主要应用于移动处理器和工业控制领域，相比于多片结构，单片SoC的集成度更高，功耗控制方法更加灵活，有利于系统的小型化和低功耗设计。 
不过，由于全系统都在一个芯片上实现，系统的扩展性没有多片结构好，升级的开销也更大。


Components
--------------

Bus
^^^^^^^^^^^

总线是计算机硬件系统中的一种通信系统，它允许各种设备共享资源。
总线可以连接各种设备，如处理器、内存、硬盘等，使它们能够进行数据交换。

在总线系统中，主机和从机是两种基本的设备类型。

- 主机（master）：主机通常是控制总线通信的设备，它发起数据传输请求。
在计算机系统中，CPU通常被视为主机，因为它控制着其他所有设备的操作。
- 从机（slave）：从机是接收主机请求并响应的设备。它们不会主动发起数据传输，而是等待主机的请求。
在计算机系统中，如硬盘、内存、输入/输出设备等通常被视为从机。

在总线通信中，主机和从机的角色是相对的。在某些情况下，一个设备可能同时充当主机和从机。
例如，在DMA（直接内存访问）操作中，硬盘驱动器可能充当主机，而内存则充当从机。

AMBA（Advanced Microcontroller Bus Architecture）是由ARM公司提出的一种开放标准，用于在嵌入式微控制器中连接和管理系统中的功能模块。
AMBA总线架构主要包括以下几种类型：

- AMBA APB（Advanced Peripheral Bus）：高级外设总线，用于连接低功耗和低性能的外设，如UART、I2C、SPI等。
- AMBA AHB（Advanced High-performance Bus）：高性能总线，用于连接高性能模块，如CPU核、DSP核、RAM接口等。
- AMBA AXI（Advanced eXtensible Interface）：高级可扩展接口，是一种高性能、低延迟的总线接口，用于设计高速互连的系统。
- AMBA ACE（Advanced Coherency Extensions）：高级一致性扩展，用于支持多处理器系统的一致性通信。

有关 AXI 协议的详细介绍，可以参考 `AXI intro <https://siris-limx.github.io/useful-utility-tutorial/AXI/>`__ 。

CLINT
^^^^^^^^^^^^^^^

全称为 Core-local Interrupt Controller，是 RISC-V 架构中用于处理本地中断的一个组件。
CLINT 主要处理两种类型的中断：软件中断（Software Interrupts）和定时器中断（Timer Interrupts）。

CLINT 中有三个寄存器被作为 slave 映射到总线地址上，分别为 ``msip``、``mtimecmp`` 和 ``mtime``。

- ``msip``：Machine mode software interrupt（IPI，inter-process interrupt）。一个由软件设置来触发中断的寄存器，可以用来置位 ``mip`` 寄存器中的 ``MSIP`` 位。
- ``mtimecmp``：Machine mode timer compare register。机器模式下的定时器比较寄存器，当 ``mtime`` 寄存器的值大于或等于 ``mtimecmp`` 寄存器的值时，就会产生一个定时器中断。这个中断会一直保持，直到通过写入 ``mtimecmp`` 寄存器来清除它。只有当中断被启用并且 ``mie`` 寄存器中的 ``MTIE`` 位被设置时，这个中断才会被接收。
- ``mtime``：Timer register。这是一个 64 位的计数器，通常用于生成定时器中断。

.. attention::

   CLINT 对 CPU 发起的中断不会经过总线。

.. note::

   ``mip`` 和 ``mie`` 都是 CSR，前者包含待处理中断的信息，后者包含中断使能位。

PLIC
^^^^^^^^^^^^^^^^

全称为 Platform-Level Interrupt Controller，它将全局中断源（通常是 I/O 设备）连接到中断目标（通常是 CPU）。

Debug Module
^^^^^^^^^^^^^^^

RISC-V 官方有 debug 的 `设计说明文档 <https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf>`__ ，类似于 ISA，是一种规范。

.. figure:: ../img/debugsys_schematic.svg
   :align: center

调试系统与多个组件交互，接下来我们将对此进行描述。
调试模块通过核心接口（Core Interface）与被调试的 hart（hardware thread，对于没有超线程支持的 CPU 来说，指的就是一个 CPU 核） 进行交互，通过其总线主机（Bus Host）和系统总线进行交互，并通过调试模块接口 (DMI) 与调试传输模块（DTM）进行交互。

JTAG
***************

与我们直接交互的软件为调试器（例如 GDB），它运行在调试主机上。
调试器与调试转换器（例如 OpenOCD）通信，调试转换器与调试传输硬件（例如 USB-JTAG 适配器）通信。
调试传输硬件通过 JTAG 信号连接到测试平台（待测试的SoC）的调试传输模块 (DTM)。
DTM 使用调试模块接口 (DMI) 提供对调试模块 (DM) 的访问。

外部调试器通过专用总线（调试模块接口 (DMI)）与调试模块的寄存器交互。
这些寄存器称为“调试模块寄存器”（Debug Module Registers）。


Core Interface
********************

调试模块发出调试请求（debug request）让 CPU 进入调试模式。
CPU 接收到调试请求后，会跳转到 Debug ROM 中的暂停地址（Halt Address），将 ``pc`` 保存在 ``dpc`` 中，更新 ``dcsr``。
CPU 要从调试模式返回，需要使用 ``DRET`` 指令，这条指令一般会位于 Debug ROM 中。

Bus Interface
********************

调试模块作为 master 连接到系统总线，可以写入 SRAM，或验证其内容。

调试存储器（Debug Memory）包含 Program Buffer、Debug ROM 和 一些 CSR。
它作为 slave 被映射到总线的地址上。


Intergration
--------------

Memory Map
^^^^^^^^^^^^

CPU 通过地址访问外围设备，因此需要给每个设备分配地址空间和属性，这个过程称为地址映射（memory mapping）。


每个映射的地址可以有不同的属性（attribute），这些属性定义了对该地址空间的访问特性和行为。

- EX (Executable)：指示相应的内存区域可以用于存放可执行代码。当 CPU 尝试执行存储在具有此属性的地址区域中的指令时，如果该区域被标记为可执行（EX），这个操作是合法的。
如果一个区域不是可执行的，尝试在这个区域执行代码将触发异常，这是一种安全特性，用于防止例如缓冲区溢出攻击。
- NI (Non-Idempotent)：非幂等属性意味着对同一个地址的多次读取或写入操作可能会产生不同的结果。
这个属性通常用于映射到特殊硬件设备的内存区域，例如 I/O 设备。
在这些区域，每次访问可能都会改变设备的状态或触发某种操作，因此相同的操作不会总是产生相同的结果。
例如，从一个设备状态寄存器读取可能会清除该寄存器，或者向控制寄存器写入可能会触发硬件操作。
- C (Cached)：这个属性指示相应的内存区域可以被缓存。
请注意，如果你需要写入某一段地址，那么这段地址必须是可缓存的。


.. note::

   This section is under development.
