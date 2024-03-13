Frontend
=====

.. Tip::

   数字综合的规模很大，因此基本上依赖于脚本。
   如果想要深入理解脚本做了什么，请善用 ``man`` 指令。

从习惯上来说，我们一般把生成网表前的步骤称为前端（frontend）。
我们数字前端所使用的 EDA 工具是 Cadence Genus。
它的输入是 RTL 文件，输出是网表，进行的操作称为综合（synthesis）。
在使用 Genus 综合前，首先需要把示例文件夹 ``build_adder`` 复制至你的工作目录下，之后我们称之为 ``<work-dir>``。
以下是 ``<work-dir>`` 的目录结构：

- ``rtl``: 存放要被综合的 RTL，并需要将 rtl 名字添加到 rtl/srcs.tcl 文件中。
- ``sram``: 存放 sram compiler 产生的各种 SRAM view。
- ``tb``: RTL 仿真的 testbench。
- ``logic_syn``: Genus script。
- ``scripts``: 流程的所有脚本，后面会具体介绍。

TLDR
---------------------

复制模板文件夹 ``build_adder`` 至你的工作目录 ``<work-dir>``。
修改以下文件中的对应内容：

1. ``scripts/design_inputs_macro.tcl``

- PDK 和标准库的路径：``set rm_foundry_kit_dirs``。
- 综合的时钟周期：``set rm_clock_period``。
- SRAM 的路径：``set sram_insts`。

2. ``scripts/core_config.tcl``

- 顶层模块名称：``set rm_core_top``。
- 时钟信号名称：``set rm_clock_pin``。


3. ``rtl/srcs.tcl``

- 添加需要综合的 RTL 文件。

.. attention::

   你可以使用 ``scripts/grep_syn_src.sh`` 脚本来自动添加 RTL 文件，该脚本接收一个命令行参数用于指定搜索路径，如果没有指定，则默认搜索 ``rtl/`` 文件夹。

.. attention::

   将所需要的SRAM lib、Verilog 文件生成并放置于 ``sram/my_sram/`` 文件夹中，并确保 sram 的名称与文件夹的一致性。请注意，``rtl/srcs.tcl`` 中请不要包括 sram 的 Verilog 文件，这样综合工具才会去寻找 lib 文件。

4. ``scripts/tech.tcl``

不能走数字综合的模块，例如 CIM，需要在综合的时候读取 lib 文件获取时序信息，因此需要在 ``tech.tcl`` 中添加对应 lib 文件的路径。
具体添加的方法为，找到 lib 对应的 corner、voltage、temperature，然后添加到对应的 ``[ss|tt|ff]_*v_*c_libs`` 中。

``rtl/srcs.tcl`` 中的 CIM Verilog 文件请只定义端口，不要有任何逻辑，这样综合工具才会去寻找 lib 文件，同时 lib、lef 文件中的 library 名称也必须与模块名称相同。

以上内容修改完成后，在你的工作目录 ``<work-dir>`` 运行 ``make genus_syn`` 即可进行综合。


弃用
-------------------

``make genus_syn`` 会调用 ``scripts/genus_synthesis.tcl`` 脚本，进行综合。

``genus_synthesis.tcl`` 首先会调用 ``scripts/core_config.tcl, srcipts/tech.tcl, logic_syn/scripts/project_vars.tcl`` 进行 Genus 环境的设置。我们需要修改 ``core_config.tcl`` 中的顶层模块名称（``set rm_core_top``）、时钟信号名称（``set rm_clock_pin``）rst??。
``tech.tcl`` 会 ``source scripts/design_inputs_macro.tcl``，这之中定义了工艺库的路径和名称、SRAM 的路径。需要修改 ``design_inputs_macro.tcl`` 中的标准工艺库名称（````）、SRAM 的名称（````）、综合时的频率（``set rm_clock_period``）。
``project_vars.tcl`` 中定义了工程的名称、工程路径、工程的一些设置。需要修改 ``project_vars.tcl`` 中的工程名称、工程路径。

Clock
--------------

:code:`core_config.tcl` 中 :code:`set rm_clock_pin io_clock` 的 :code:`io_clock` 为全局时钟信号名称，需要与RTL中的时钟信号名称对应。


.. note::

   This section is under development.
