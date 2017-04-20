..  BSD LICENSE
    Copyright(c) 2010-2014 Intel Corporation. All rights reserved.
    All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted provided that the following conditions
    are met:

    * Redistributions of source code must retain the above copyright
    notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above copyright
    notice, this list of conditions and the following disclaimer in
    the documentation and/or other materials provided with the
    distribution.
    * Neither the name of Intel Corporation nor the names of its
    contributors may be used to endorse or promote products derived
    from this software without specific prior written permission.

    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
    "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
    LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
    A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
    LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
    DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
    THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
    (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

**第一部分：架构概述**

概述
====

本章节给出了DPDK架构的一个全局的描述。

DPDK的主要目标就是要为数据面快速报文处理应用提供一个简洁但是完整的框架。
用户可以通过代码来理解其中使用的一些技术，并用来构建自己的应用原型或是添加自己的协议栈。
用户也可以替换DPDK提供的原生的选项。

通过创建环境抽象层EAL，DPDK框架为每个特殊的环境创建了运行库。
这个环境抽象层是对底层架构的抽象，通过make和配置文件，在Linux用户空间编译完成。
一旦EAL库编译完成，用户可以通过链接这些库来构建自己的app。
除开环境抽象层，还有一些其他库，包括哈希算法、最长前缀匹配、环形缓冲器。
DPDK提供了一些app用例用来指导如何使用这些特性来创建自己的应用程序。

DPDK实现了run-to-complete报文处理模型，数据面处理程序在调用之前必须预先分配好所有的资源，并作为执行单元运行与逻辑核心上。
这种模型并不支持调度，且所有的设备通过轮询方式访问。
不使用中断方式的主要原因就是中断处理增加了性能开销。

作为RTC模型的扩展，通过使用ring在不同core之间传递报文和消息，也可以实现报文处理的流水线模型（pipeline）。
流水线模型允许操作分阶段执行，在多核代码执行中可能更高效。


开发环境
--------

DPDK项目创建要求Linux环境及相关的工具链，例如一个或多个编译工具、汇编程序、make工具、编辑器及DPDK组建和库用到的库。

当制定环境和架构的库编译出来，这些库就可以用于创建我们自己的数据面处理程序。

创建Linux用户空间app时，需要用到glibc库。
对于DPDP app，必须使用两个全局的环境变量（RTE_SDK & RTE_TARGET），这两个变量必须在编译app之间配置好:

.. code-block:: console

    export RTE_SDK=/home/user/DPDK
    export RTE_TARGET=x86_64-native-linuxapp-gcc

也可以参阅 *DPDK入门指南* 来获取更多搭建开发环境的信息。

环境适配层EAL
-------------

环境适配层(Environment Abstraction Layer)提供了通用的接口来隐藏了环境细节，使得上层app和库无需考虑这些细节。
EAL提供的服务有：

*   DPDK的加载和启动

*   支持多线程和多进程执行方式

*   CPU亲和性设置

*   系统内存分配和释放

*   原子操作

*   定时器引用

*   PCI总线访问

*   跟踪和调试功能

*   CPU特性编号

*   中断处理

*   警告操作

*   内存管理

EAL的更完整的描述请参阅 :ref:`Environment Abstraction Layer <Environment_Abstraction_Layer>`.

核心组件
--------

*核心组件* 指一系列的库，用于为高性能包处理程序提供所有必须的元素。核心组件及其之间的关系如下图所示:

.. _figure_architecture-overview:

.. figure:: img/architecture-overview.*

   Core Components Architecture


环形缓冲区管理(librte_ring)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ring数据结构提供了一个无锁的多生产者，多消费者的FIFO表处理接口。
他比无锁队列优异的地方在于它容易部署，适合大量的操作，而且更快。
Ring库在 :ref:`Memory Pool Manager (librte_mempool) <Mempool_Library>` 中使用到，
而且ring还用于不同核之间或是逻辑核上处理单元之间的通信。
Ring缓存机制及其使用可以参考 :ref:`Ring Library <Ring_Library>`。

内存池管理(librte_mempool)
~~~~~~~~~~~~~~~~~~~~~~~~~~

内存池管理的主要职责就是在内存中分配指定数目对象的POOL。
每个POOL以名称来唯一标识，并且使用一个ring来存储空闲的对象节点。
它还提供了一些其他的服务如对象节点的每核备份缓存及自动对齐以保证元素能均衡的处于每核内存通道上。
内存池分配器具体行为参考 :ref:`Mempool Library <Mempool_Library>`。

网络报文缓冲区管理(librte_mbuf)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

报文缓存管理器提供了创建、释放报文缓存的能力，DPDK应用程序中可能使用这些报文缓存来存储消息。
而消息通常在程序开始时通过DPDK的MEMPOOL库创建并存储。
BUFF库提供了报文申请释放的API，通常消息buff用于缓存普通消息，报文buff用于缓存网络报文。
报文缓存管理参考 :ref:`Mbuf Library <Mbuf_Library>`。

定时器管理(librte_timer)
~~~~~~~~~~~~~~~~~~~~~~~~

这个库位DPDK执行单元提供了定时服务，为函数异步执行提供支持。
定时器可以设置周期调用或只调用一次。
使用EAL提供的接口获取高精度时钟，并且能在每个核上根据需要初始化。
具体参考 :ref:`Timer Library <Timer_Library>`。

以太网轮询驱动架构
------------------

DPDK的PMD驱动支持1G、10G、40G。
同时DPDK提供了虚拟的以太网控制器，被设计成非异步，基于中断的模式。
详细内容参考 :ref:`Poll Mode Driver <Poll_Mode_Driver>`。

报文转发算法支持
----------------

DPDK提供了哈希（librte_hash）、最长前缀匹配的（librte_lpm）算法库用于支持包转发。
详细内容查看 :ref:`Hash Library <Hash_Library>` 和  :ref:`LPM Library <LPM_Library>` 。

网络协议库(librte_net)
----------------------

这个库提供了IP协议的一些定义，以及一些常用的宏。
这些定义都基于FreeBSD IP协议栈的代码，并且包含相关的协议号，IP相关宏定义，IPV4和IPV6头部结构等等。
