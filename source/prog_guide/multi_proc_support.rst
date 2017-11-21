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

.. _Multi-process_Support:

多进程支持
============

在DPDK中，多进程支持旨在允许一组DPDK进程以简单的透明方式协同工作，以执行数据包处理或其他工作负载。为了支持此功能，已经对核心的DPDK环境抽象层（EAL）进行了一些增加。


EAL已被修改为允许不同类型的DPDK进程产生，每个DPDK进程在应用程序使用的hugepage内存上具有不同的权限。现在可以指定两种类型的进程：

*   primary processes, 可以初始化，拥有共享内存的完全权限

*   secondary processes, 不能初始化共享内存，但可以附加到预初始化的共享内存并在其中创建对象。

独立DPDK进程是primary processes，而secondary processes只能与主进程一起运行，或者主进程已经为其配置了hugepage共享内存。

为了支持这两种进程类型以及稍后描述的其他多进程设置，EAL还提供了两个附加的命令行参数：

*   ``--proc-type:`` 用于将给定的进程实例指定为primary processes或secondary processes DPDK实例。

*   ``--file-prefix:`` 以允许不希望协作具有不同存储器区域的进程。

DPDK提供了许多示例应用程序，演示如何可以一起使用多个DPDK进程。这些用例在《DPDK Sample Application用户指南》中的“多进程示例应用”一章中有更详尽的记录。

内存共享
----------

使用DPDK的多进程应用程序工作的关键要素是确保内存资源在构成多进程应用程序的进程之间正确共享。一旦存在可以通过多个进程访问的共享存储器块，则诸如进程间通信（IPC）的问题就变得简单得多。

在独立进程或者primary processes启动时，DPDK向内存映射文件中记录其使用的内存配置的详细信息，包括正在使用的hugepages，映射的虚拟地址，存在的内存通道数等。当secondary processes启动时，这些文件被读取，并且EAL在secondary processes中重新创建相同的内存配置，以便所有内存区域在进程之间共享，并且所有指向该内存的指针都是有效的，并且指向相同的对象。

.. note::

    有关Linux内核地址空间布局随机化（ASLR）如何影响内存共享的详细信息参考 `多进程限制`_ 。

.. _figure_multi_process_memory:

.. figure:: img/multi_process_memory.*

   Memory Sharing in the DPDK Multi-process Sample Application


EAL还支持自动检测模式（由EAL -proc-type = auto标志设置），如果主实例已经在运行，则DPDK进程作为辅助实例启动。

部署模式
----------

对称/对等进程
~~~~~~~~~~~~~~~~

DPDK多进程支持可用于创建一组对等进程，每个进程执行相同的工作负载。该模型相当于具有多个线程，每个线程都运行相同的主循环功能，如大多数提供的DPDK示例应用程序中所完成的一样。 在此模型中，应使用–proc-type = primary EAL标志生成第一个生成的进程，而所有后续实例都应使用–proc-type = secondary标志生成。

simple_mp和symmetric_mp示例应用程序演示了此模型的用法。它们在《DPDK Sample Application用户指南》中“多进程示例应用”一章中有描述。

非对称/非对等进程
~~~~~~~~~~~~~~~~~~~~

可用于多进程应用程序的替代部署模型是具有单个primary process实例，充当负载均衡器或distributor，在作为secondary processes运行的worker或客户机线程之间分发接收到的数据包。在这种情况下，广泛使用rte_ring对象，它们位于共享的hugepage内存中。

client_server_mp示例应用程序显示此模型用法。在《DPDK Sample Application用户指南》中“多进程示例应用”一章中有描述。

运行多个独立的DPDK应用程序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

除了涉及多个DPDK进程的上述情况之外，可以并行运行多个DPDK进程，这些进程都可以独立工作。使用EAL的–file-prefix参数提供对此使用场景的支持。

默认情况下，EAL使用rtemap_X文件名在每个hugetlbfs文件系统上创建hugepage文件，其中X的范围为0到最大的hugepages -1。同样，当以root身份运行（或以非root用户身份运行时为$ HOME / .rte_config），如果文件系统和设备权限为空，则会在每个进程中使用/var/run/.rte_config文件名创建共享配置文件）。以上每个文件名的部分可以使用file-prefix参数进行配置。

除了指定file-prefix参数外，并行运行的任何DPDK应用程序都必须明确限制其内存使用。这通过将-m标志传递给每个进程来指定每个进程可以使用多少hugepage内存（以兆字节为单位）（或通过–socket-mem来指定每个进程可以使用每个套接字的多少hugepage内存）。

.. note::

    在单台机器上并行运行的独立DPDK实例无法共享任何网络端口。一个进程使用的任何网络端口都应该在其他进程中列入黑名单。

运行多个独立的DPDK应用程序组
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以同样的方式，可以在单个系统上并行运行独立的DPDK应用程序，这也可以简单地扩展到并行运行DPDK应用程序的多进程组。在这种情况下，secondary processes必须使用与其共享内存连接的primary process相同的–file-prefix参数。

.. note::

    并行运行的多个独立DPDK进程的所有限制和问题也适用于此使用场景。

多进程限制
------------

运行DPDK多进程应用程序时存在一些限制。其中一些记录如下：

*   多进程功能要求在所有应用程序中都存在完全相同的hugepage内存映射。Linux安全功能，地址空间布局随机化（ASLR）可能会干扰此映射，因此可能需要禁用此功能才能可靠地运行多进程应用程序。

.. warning::

    禁用地址空间布局随机化（ASLR）可能具有安全隐患，因此建议仅在绝对必要时才被禁用，并且只有在了解了此更改的含义时。

* 作为单个应用程序运行并使用共享内存的所有DPDK进程必须具有不同的coremask /corelist参数。任何相同的逻辑内核不可能拥有primary和secondary实例或两个secondary实例。尝试这样做可能会导致内存池缓存的损坏等问题。

* 中断的传递，如Ethernet设备链路状态中断，在secondary process中不起作用。所有中断仅在primary process内触发。在多个进程中需要中断通知的任何应用程序都应提供自己的机制，将中断信息从primary process转移到需要该信息的任何secondary process。


*  不支持使用基于不同编译二进制文件运行的多个进程之间的函数指针，因为在一个进程中给定函数的位置可能与其中一个进程的位置不同。这样可以防止librte_hash库在多线程实例中正常运行，因为它在内部使用了一个指向散列函数的指针。

要解决此问题，建议多进程应用程序通过直接从代码中调用散列函数，然后使用rte_hash_add_with_hash()/rte_hash_lookup_with_hash()函数来执行哈希计算，而不是内部执行散列的函数，例如rte_hash_add()/rte_hash_lookup()。

* 根据所使用的硬件和所使用的DPDK进程的数量，可能无法在每个DPDK实例中都使用HPET定时器。可用于Linux 用户空间的HPET comparators的最小数量可能只有一个，这意味着只有第一个primary DPDK进程实例可以打开和mmap/dev/hpet。如果所需DPDK进程的数量超过可用的HPETcomparators数量，则必须使用TSC（此版本中的默认计时器）而不是HPET。

