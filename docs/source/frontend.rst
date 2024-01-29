Frontend
=====

.. Tips::
   
   数字综合的规模很大，因此基本上依赖于脚本。
   如果想要深入理解脚本做了什么，请善用 ``man`` 指令。

从习惯上来说，我们一般把生成网表前的步骤称为前端（frontend）。
我们数字前端所使用的 EDA 工具是 Cadence Genus。
它的输入是 RTL 文件，输出是网表，进行的操作称为综合（synthesis）。
在使用 Genus 综合前，首先需要把实例文件夹 ``build_adder`` 复制至你的工作目录下，之后我们称之为 ``<work-dir>``。
以下是 ``<work-dir>`` 的目录结构：

- ``rtl``: 存放要被综合的 RTL，并需要将 rtl 名字添加到 rtl/srcs.tcl 文件中。
- ``sram``: 存放 sram compiler 产生的各种 SRAM view。
- ``tb``: RTL 仿真的 testbench。
- ``logic_syn``: Genus script。
- ``scripts``: 流程的所有脚本，后面会具体介绍。



Clock
--------------

:code:`core_config.tcl` 中 :code:`set rm_clock_pin io_clock` 的 :code:`io_clock` 为全局时钟信号名称，需要与RTL中的时钟信号名称对应。


.. note::

   This section is under development.
