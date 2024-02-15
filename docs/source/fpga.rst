FPGA Implementation
=========

.. contents:: Table of Contents

Deployment
--------------

我们以 Ariane APU 为例，使用 Vivado 进行 FPGA 的部署。

.. attention::

   使用 GUI 界面虽然简单，但是效率很低，而且 Vivado 的 GUI 做得很差。
   我们推荐你使用 TCL（Tooling Command Language）操作 Vivado。
   请自行查看 CVA6 项目中完整的 Vivado 流程，我们只会解释部分重要的 TCL 片段。

在 <cva6> 路径下运行 ``make fpga``，该脚本会搜索所有的 RTL 源文件并将其添加到 ``<cva6>/corev_apu/fpga/scripts/add_sources.tcl``。

接着会在 ``<cva6>/corev_apu/fpga`` 中运行对应的 Makefile。
这个脚本首先会遍历 ``<cva6>/corev_apu/fpga/xilinx`` 目录中所有的 IP 文件夹，生成 IP 并综合。
然后运行 ``<cva6>/corev_apu/fpga/scripts`` 目录中的 ``prologue.tcl`` 和 ``run.tcl`` 综合源文件，最后布局布线生成 bitstream。

Source Files
^^^^^^^^^^^

将整个项目文件夹丢给 Vivado，让其自动寻找源文件是一种可取的方案。

CVA6 项目中，在 ``<cva6>`` 目录下运行 ``make fpga``，即可生成获取所有源文件的 TCL 脚本。
该文件为 ``<cva6>/corev_apu/fpga/src/scripts/add_sources.tcl``。

IP
^^^^^^^^^^^

Vivado 中提供了许多外设和总线的 IP（Intellectual Property），因此我们首先需要生成这些 IP。

.. attention::

   我们最终不会使用 IP，需要替换为自己实现的 RTL。

我们给出 CVA6 中是如何生成这些 IP 的。

1. 设置一些环境变量。

.. code-block::

   set partNumber $::env(XILINX_PART)
   set boardName  $::env(XILINX_BOARD)
   
   set ipName xlnx_axi_clock_converter

获取 FPGA 芯片的型号、板卡的名称和 IP 核心的名称。

2. 建一个新的项目。

.. code-block::
   
   create_project $ipName . -force -part $partNumber
   set_property board_part $boardName [current_project]
   create_ip -name axi_clock_converter -vendor xilinx.com -library ip -module_name $ipName
   set_property -dict [list CONFIG.ADDR_WIDTH {64} CONFIG.DATA_WIDTH {64} CONFIG.ID_WIDTH {5}] [get_ips $ipName]

项目的名称为 IP 核心的名称，项目的位置为当前目录，如果项目已经存在则强制覆盖，项目的 FPGA 芯片型号为前面从环境变量中获取的型号。
设置当前项目的板卡名称为前面从环境变量中获取的名称。

创建一个新的 IP 核心，核心的名称为 axi_clock_converter，供应商为 xilinx.com，库为 ip，模块的名称为前面设置的 IP 核心的名称。

设置 IP 核心的地址宽度为 64 位，数据宽度为 64 位，ID 宽度为 5 位。

3. IP 综合。

.. code-block::

   generate_target {instantiation_template} [get_files ./$ipName.srcs/sources_1/ip/$ipName/$ipName.xci]
   generate_target all [get_files  ./$ipName.srcs/sources_1/ip/$ipName/$ipName.xci]
   create_ip_run [get_files -of_objects [get_fileset sources_1] ./$ipName.srcs/sources_1/ip/$ipName/$ipName.xci]
   launch_run -jobs 8 ${ipName}_synth_1
   wait_on_run ${ipName}_synth_1

首先生成 IP 核心的实例化模板。
实例化模板是一个包含了如何实例化 IP 核心的代码的文件。
然后，生成所有目标。
在这里，所有目标可能包括了实例化模板、综合结果、实现结果等。

创建一个 IP 核心的运行。
在这里，运行是一个包含了如何综合和实现 IP 核心的流程的对象。
启动 IP 核心的综合。在这里，``-jobs 8`` 参数表示使用 8 个并行任务来执行综合。
最后等待综合完成，确保在继续执行后续的脚本之前，综合已经成功完成。

4. 重复步骤 1 ~ 3，直到所有的 IP 都已经生成。

Design Constraint
^^^^^^^^^^^^^^

1. FPGA 设计项目的创建和一些参数的设置。

.. code-block::

   set project ariane
   create_project $project . -force -part $::env(XILINX_PART)
   set_property board_part $::env(XILINX_BOARD) [current_project]
   # set number of threads to 8 (maximum, unfortunately)
   set_param general.maxThreads 8
   set_msg_config -id {[Synth 8-5858]} -new_severity "info"
   set_msg_config -id {[Synth 8-4480]} -limit 1000

设置变量 project，其值为 ariane。
这个变量将被用作项目的名称。

创建一个新的项目，项目的名称为 project 变量的值，即 ariane。
项目的位置是当前目录（.）。
-force 选项表示如果项目已经存在，则覆盖它。
-part $::env(XILINX_PART) 选项表示项目的 FPGA 芯片型号为环境变量 XILINX_PART 的值。

设置了当前项目的板卡型号为环境变量 XILINX_BOARD 的值、Vivado 的最大线程数为 8。
改变消息 Synth 8-5858 的严重性级别为 "info"，Synth 8-4480 的最大显示次数为 1000。

2. IP 的读取、包含目录的设置以及顶层设计的设置。

``read_ip {...}``：读取了一系列 IP。
这些 IP 核的文件路径被包含在大括号 {} 中，每个路径都被双引号 "" 包围。
这些 IP 包括 DDR3 内存接口、AXI 时钟转换器、AXI 数据宽度转换器、AXI GPIO、AXI Quad SPI 和时钟生成器等。

``set_property include_dirs {...} [current_fileset]``：这个命令设置了当前文件集的包含目录。
这些目录包含了设计所需的头文件。
这些目录的路径被包含在大括号 {} 中，每个路径都被双引号 "" 包围。

``source scripts/add_sources.tcl``：这个命令执行了一个 Tcl 脚本 add_sources.tcl。
这个脚本可能包含了一些添加源文件的命令。

``set_property top ${project}_xilinx [current_fileset]``：这个命令设置了当前文件集的顶层设计。
顶层设计的名称为 ${project}_xilinx，其中 ${project} 是一个变量，其值应该在之前的代码中被设置。

3. 向设计项目中添加约束文件。

``add_files -fileset constrs_1 -norecurse constraints/$project.xdc``：这个命令向名为 constrs_1 的文件集中添加了一个约束文件。
约束文件的路径为 constraints/$project.xdc，其中 $project 是一个变量，其值应该在之前的代码中被设置。
-norecurse 选项表示不递归地添加目录中的文件，也就是说，只添加指定的文件，不添加该文件所在目录下的其他文件。

.. attention::

   在约束文件中加入 ``set_property CLOCK_DEDICATED_ROUTE FALSE [get_nets tck_IBUF]``，否则 Vivado 会报错。


Bitstream
^^^^^^^^^^^^

.. code-block::

   add_files -fileset constrs_1 -norecurse constraints/$project.xdc
   synth_design -rtl -name rtl_1
   set_property STEPS.SYNTH_DESIGN.ARGS.RETIMING true [get_runs synth_1]
   launch_runs synth_1
   wait_on_run synth_1
   open_run synth_1


启动名为 rtl_1 的 RTL 级别的综合。
设置 synth_1 综合步骤的参数，使得综合过程中进行重时序操作。重时序可以优化设计的时序性能。
最终启动名为 synth_1 的综合流程，并打开 synth_1 的综合流程的结果。
这个结果包括了综合报告、网表文件等。

.. code-block::

   # set for RuntimeOptimized implementation
   set_property "steps.place_design.args.directive" "RuntimeOptimized" [get_runs impl_1]
   set_property "steps.route_design.args.directive" "RuntimeOptimized" [get_runs impl_1]

设置名为 impl_1 的实现流程中布局布线设计步骤的指令为 "RuntimeOptimized"。
"RuntimeOptimized" 指令会优化设计的运行时间。

.. code-block::

   launch_runs impl_1
   wait_on_run impl_1
   launch_runs impl_1 -to_step write_bitstream
   wait_on_run impl_1
   open_run impl_1

启动名为 `impl_1` 的实现流程，但只执行到 "write_bitstream" 步骤。
"write_bitstream" 步骤是实现流程的最后一个步骤，它生成了一个比特流文件，这个文件可以被下载到 FPGA 芯片上。
打开名为 `impl_1` 的实现流程的结果。
这个命令可以让用户查看实现流程的结果，包括布局布线的结果和比特流文件（.bit）。

.. Tip::

   .bit 文件是一个二进制文件，用于直接配置FPGA的硬件。
   当你设计并综合一个FPGA项目时，最终会生成一个.bit文件。
   这个文件包含了用于配置FPGA的所有必要信息，如查找表（LUTs）、寄存器等的配置数据。
   通常，这个文件是通过JTAG或其他直接编程接口传输到FPGA的。
   一旦FPGA断电，这个配置就会丢失。

.. hint::

   如果你想要 FPGA 每次启动时都能自动加载所需的配置，那你需要将 .bit 文件转换成 .mcs 文件（Memory Configuration Stream）。
   这是一个用于非易失性存储器编程的文件，比如用于配置PROM（Programmable Read-Only Memory）或者闪存。

Report
^^^^^^^^^^^^^^^^

.. code-block::

   check_timing -verbose                                                   -file reports/$project.check_timing.rpt
   report_timing -max_paths 100 -nworst 100 -delay_type max -sort_by slack -file reports/$project.timing_WORST_100.rpt
   report_timing -nworst 1 -delay_type max -sort_by group                  -file reports/$project.timing.rpt
   report_utilization -hierarchical                                        -file reports/$project.utilization.rpt
   report_cdc                                                              -file reports/$project.cdc.rpt
   report_clock_interaction                                                -file reports/$project.clock_interaction.rpt

生成 FPGA 设计的各种报告，包括时序报告、资源利用率报告、CDC 报告和时钟交互报告。

.. code-block::

   # output Verilog netlist + SDC for timing simulation
   write_verilog -force -mode funcsim work-fpga/${project}_funcsim.v
   write_verilog -force -mode timesim work-fpga/${project}_timesim.v
   write_sdf     -force work-fpga/${project}_timesim.sdf

生成 Verilog 网表和 SDF 文件，用于功能仿真和时序仿真。
这是 FPGA 设计流程的一部分，通过这个步骤，可以对设计进行仿真，验证设计的功能和时序。

Adjustment
^^^^^^^^^^^^^^^^^^^

为了实现 FPGA 的移植，我们需要修改部分脚本和源文件。

- ``<cva6>/Makefile``：``XILINX_PART`` ``XILINX_BOARD`` 修改。
- ``<cva6>/corev_apu/fpga/Makefile``：只保留 ips 中的 xlnx_clk_gen.xci、xlnx_axi_dwidth_converter_dm_master.xci 和 xlnx_axi_dwidth_converter_dm_slave.xci。
- ``<cva6>/corev_apu/fpga/scripts/run.tcl``：注释掉 read_ip 中不需要的 ``.xci``。
可以选择在 ``launch_runs`` 后添加选项 ``-jobs <cpu_core_nums>``。另外，如果需要挂接 SRAM，你需要注释掉如下几行代码：

.. code-block::

   # launch_runs -jobs 24 impl_1 -to_step write_bitstream
   # wait_on_run impl_1
   # open_run impl_1

并替换成如下的代码：

.. code-block::

   open_run impl_1
   set_property SEVERITY {Warning} [get_drc_checks LUTLP-1]
   set_property IS_ENABLED 0 [get_drc_checks {CSCL-1}]
   write_bitstream -force work-fpga/${project}.bit

否则，Vivado 会报 combinational loop 的错。

- ``<cva6>/corev_apu/fpga/src/ariane_xilinx.sv``：根据需求，注释掉不需要的部分。
- ``<cva6>/corev_apu/fpga/src/ariane_peripherals_xilinx.sv``：根据需求，注释掉不需要的部分。

.. Hint::

   建议将时钟信号引出，约束到 led 上，以便观察时钟信号是否存在。

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

CVA6 Example
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

RISC-V 官方推荐的调试平台即为 OpenOCD，因此我们也采用 OpenOCD 作为我们 SoC 的调试工具。
安装方法如下：

.. code-block::

   $ git clone https://github.com/riscv/riscv-openocd
   $ sudo apt-get install libftdi-dev libusb-1.0-0 libusb-1.0-0-dev autoconf automake texinfo pkg-config
   $ cd riscv-openocd
   $ ./bootstrap
   $ ./configure --enable-ftdi
   $ make -j<number of your cpus>
   $ sudo make install

如果你安装成功，执行如下指令，你会看到类似的输出：

.. code-block::

   $ which openocd
   /usr/local/bin/openocd
   $ openocd -v
   Open On-Chip Debugger 0.12.0+dev-03598-g78a719fad (2024-01-20-05:43)
   Licensed under GNU GPL v2
   For bug reports, read
           http://openocd.org/doc/doxygen/bugs.html

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

Debug Module
##############

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

Flow
^^^^^^^^^^^^

1. 烧录 bitstream 到 FPGA 上。

在 Vivado GUI 中，打开 hardware manager，将生成的 bitstream 通过 jtag 接口烧录至 FPGA 中。

2. 连接 PC 和 FPGA。

JTAG Adapter 的 USB 端接入 PC，另一端接到实例化 SoC 中 JTAG 对应的约束管脚。

3. 在 PC 中启动 OpenOCD。

.. code-block::

   $ cd <cva6>/corev_apu/fpga
   $ sudo openocd -f ariane.cfg

``ariane.cfg`` 中定义了如何通过 JTAG 接口对一个 RISC-V 设备进行调试。

.. code-block::

   adapter speed  100
   adapter driver ftdi

设置适配器的速度为 100 kHz，并指定其驱动为 FTDI。

.. code-block::

   ftdi vid_pid 0x0403 0x6014

   # Channel 1 is taken by Xilinx JTAG
   ftdi channel 0

指定 FTDI 芯片的 VID 和 PID，这两个参数用于在 USB 设备中唯一标识一个设备。
并指定使用 FTDI 芯片的哪个通道进行 JTAG 调试。

.. code-block::

   ftdi layout_init 0x0018 0x001b
   ftdi layout_signal nTRST -ndata 0x0010

设置 JTAG 的引脚布局。
``ftdi layout_init`` 设置初始的引脚状态，``ftdi layout_signal`` 设置 nTRST 信号的引脚。

.. code-block::

   set _CHIPNAME riscv
   jtag newtap $_CHIPNAME cpu -irlen 5
   
   set _TARGETNAME $_CHIPNAME.cpu
   target create $_TARGETNAME riscv -chain-position $_TARGETNAME -coreid 0

创建一个新的 JTAG TAP，并创建一个目标设备。
这里的目标设备是一个 RISC-V 架构的 CPU。

.. code-block::

   gdb_report_data_abort enable
   gdb_report_register_access_error enable
   
   riscv set_reset_timeout_sec 120
   riscv set_command_timeout_sec 120

设置一些 GDB 的参数，以及 RISC-V 的超时时间。

.. code-block::
   # prefer to use sba for system bus access
   riscv set_mem_access progbuf sysbus abstract
   
   # Try enabling address translation (only works for newer versions)
   if { [catch {riscv set_enable_virtual on} ] } {
       echo "Warning: This version of OpenOCD does not support address translation. To debug on virtual addresses, please update to the latest version." }

设置 RISC-V 的内存访问方式，优先使用 system bus access，尝试启用地址转换功能。

.. code-block::

   init
   halt
   echo "Ready for Remote Connections"

执行 ``init`` 和 ``halt`` 指令，初始化 JTAG 调试器并暂停目标设备的运行。

如果你能成功启动 OpenOCD，终端中会输出如下信息：

.. code-block::

   Open On-Chip Debugger 0.12.0+dev-03598-g78a719fad (2024-01-20-05:43)
   Licensed under GNU GPL v2
   For bug reports, read
           http://openocd.org/doc/doxygen/bugs.html
   Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
   Info : clock speed 100 kHz
   Info : JTAG tap: riscv.cpu tap/device found: 0x00000001 (mfg: 0x000 (<invalid>), part: 0x0000, ver: 0x0)
   Info : [riscv.cpu] datacount=2 progbufsize=8
   Info : [riscv.cpu] Examined RISC-V core
   Info : [riscv.cpu]  XLEN=64, misa=0x800000000014112d
   [riscv.cpu] Target successfully examined.
   Info : [riscv.cpu] Examination succeed
   Info : starting gdb server for riscv.cpu on 3333
   Info : Listening on port 3333 for gdb connections
   Ready for Remote Connections
   Info : Listening on port 6666 for tcl connections
   Info : Listening on port 4444 for telnet connections

4. 使用 gdb 连接 OpenOCD。

.. code-block::

   $ <riscv-gcc-toolchain>/bin/riscv-none-elf-gdb /path/to/elf
   GNU gdb (GDB) 14.0.50.20230114-git
   Copyright (C) 2022 Free Software Foundation, Inc.
   License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
   This is free software: you are free to change and redistribute it.
   There is NO WARRANTY, to the extent permitted by law.
   Type "show copying" and "show warranty" for details.
   This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv-none-elf".
   Type "show configuration" for configuration details.
   For bug reporting instructions, please see:
   <https://www.gnu.org/software/gdb/bugs/>.
   Find the GDB manual and other documentation resources online at:
       <http://www.gnu.org/software/gdb/documentation/>.
   
   For help, type "help".
   Type "apropos word" to search for commands related to "word".
   (gdb) target remote: 3333
   (gdb)

接着，你就可以通过 GDB 调试程序和访问内存了。
一些常用的 GDB 指令如下：

- ``x/10w 0x12345``：以字（4 字节）为单位，查看地址 0x12345 开始的 10 个字的内容。
- ``x/i``：一种特殊的格式，用于将内存中的内容解释为机器指令。i 代表 "instruction"，即指令。例如，`x/i $pc` 这条命令会显示程序计数器（PC）当前指向的机器指令。
- ``info registers``：列出所有寄存器的值。
- ``set {int}0x54321 = 0xabcdf``：将地址 0x54321 处的 4 个字节的内容设置为 16 进制的 abcdf。
- ``stepi``：执行 pc 地址对应的指令。

.. Hint::

   ROM（只读存储器）是一种只能读取不能写入的存储器。
   如果你试图在 GDB 中使用 ``set`` 命令写入 ROM 地址的数据，GDB 可能不会显示错误，但实际上数据并没有被写入 ROM。
   当你使用 ``x`` 命令读取该地址时，GDB 可能会显示你之前尝试写入的数据，但这只是 GDB 内部状态的一部分，不代表实际的硬件状态。
   在真实的硬件中，ROM 的内容在写入后就不能更改。


.. note::

   This section is under development.
