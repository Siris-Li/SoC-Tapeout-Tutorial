C Runtime Initialzation
***********************

C 运行时文件 ``<cva6>/verif/bsp/crt0.S`` 提供 _start 函数，该函数是程序的入口点并执行以下任务：

- 初始化全局和堆栈指针。
- 将 ``vector_table`` 的地址存储在 ``mtvec`` 中，设置低两位为“0x1”以选择向量中断模式。
- 将 BSS 部分清零。
- 调用 C 构造函数的初始化并设置要调用的析构函数出口。它们在 C++ 中被广泛使用，但在 C 语言中并不常见。
- 将 ``argc`` 和 ``argv`` 归零（堆栈未初始化，因此将它们归零防止未初始化的值可能会导致程序的结果与预期的参考结果不匹配）。
- 调用 ``main`` 函数。
- 如果 ``main`` 函数返回，则调用 ``exit``。

.. Tip::

   在 RISC-V 架构中，``mtvec`` 是一个特殊的寄存器，它用于存储中断向量表的地址。
   当最低两位为 0x1 时，处理器会进入向量中断模式（vectored）。
   中断向量表是一个包含了处理各种中断的函数地址的表，当发生中断时，处理器会根据 ``mtvec`` 寄存器中的地址找到中断向量表，然后跳转到相应的函数去处理中断。
   这与直接模式（direct）不同，在直接模式下，所有的中断都会被送到同一个处理函数。

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
这样，gp 就被设置为了 ``__global_pointer$`` 的地址。

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

   RISC-V 指令集中的 ``auipc`` 指令可以将一个 20 位的立即数（即常数）加到程序计数器（PC）上，然后将结果存储到一个寄存器中。
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

.. code-block::

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

请参考 CVA6 的 `实现方式 <https://github.com/openhwgroup/cva6/tree/master/verif/bsp>`__ 。


System Calls
********************

请参考 CVA6 的 `实现方式 <https://github.com/openhwgroup/cva6/tree/master/verif/bsp>`__ 。


Linker Script
*******************

请参考 CVA6 的 `实现方式 <https://github.com/openhwgroup/cva6/tree/master/verif/bsp>`__ 。
