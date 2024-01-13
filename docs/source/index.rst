Welcome to SoC Tapeout Babysitting Tutorial!
===================================

本教程旨在给没有流片经验的宝宝们提供全面的文档参考。



Preface
--------


在学习和实验的过程中，你会遇到大量的问题。
你的能力是跟你独立解决问题的投入成正比的，大佬告诉你答案，展示的是大佬的能力，并不是你的能力。
因此，拥有信息的检索和获取的能力是十分必要的。

ChatGPT
^^^^^^^^^^^^

点对点解决问题，十分迅速。


Offical Manual
^^^^^^^^^^^

官方手册包含了查找对象的所有信息，关于查找对象的一切问题都可以在官方手册中找到答案。
通常官方手册的内容十分详细，在短时间内通读一遍基本上不太可能，因此你需要懂得“如何使用目录来定位你所关心的问题”。
如果你希望寻找一些用于快速入门的例子，你应该使用搜索引擎。

.. Tip::

	对于 Linux 命令行中如 gcc，verilator 的参数，可以使用 ``man`` 指令快速查看官方文档，结合 ChatGPT 使用可以显著提高效率。


Google
^^^^^^^^

英文维基百科比中文维基百科和百度百科包含更丰富的内容。
stackoverflow是一个程序设计领域的问答网站, 里面除了技术性的问题（What is ":-!!" in C code?）之外, 也有一些学术性（Is there a regular expression to detect a valid regular expression?) 和一些有趣的问题（What is the “-->” operator in C++?）。


Zhihu & BiliBili
^^^^^^^^^^

不可否认，这两个平台上存在优质的内容，适合快速入门或者闲暇时随意翻看。


Prerequisites
--------------

我们假设你有如下的先验知识：

- Linux 基础知识
- 五级流水线的 CPU 架构
- RTL 编写能力
- C、Python 等编程语言的阅读能力

.. Attention::

	最重要的是，一定要知道你做的每一个步骤的意义，流片的机会很珍贵，请务必重视！


Contents
--------

.. toctree::
	:maxdepth: 2
	:numbered:

	cpu
	intergration
	fpga
	frontend
	backend
	layout
	appendix
