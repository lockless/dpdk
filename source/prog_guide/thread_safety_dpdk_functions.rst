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

DPDK功能的线程安全
===================

DPDK由几个库组成。这些库中的某些功能可以同时被多个线程安全地调用，而另一部分则不能。 本节介绍开发人员在构建自己的应用程序时考虑这些问题。

DPDK的运行时环境通常是每个逻辑核上的单个线程。但是，在某些情况下，它不仅是多线程的，而且是多进程的。通常，最好避免在在线程和/或进程之间共享数据结构。如果不可能，则执行块必须以线程安全的方式访问数据。可以使用诸如原子操作或锁的机制，这将允许执行块串行操作。但是，这可能会对应用程序的性能产生影响。

快速路径API
--------------

在数据面中运行的应用程序对性能敏感，但这些库中的某些函数可能不会多线程并发调用。PMD中的Hash，LPM和mempool库以及RX / TX都是这样的例子。

通过设计，Hash和LPM库线程不安全，不能并行调用，以保持性能。然而，如果需要，开发人员可以在这些库之上添加封装层以提供线程安全性。在所有情况下都不需要锁，并且在哈希和LPM库中，可以在多个线程中并行执行值的查找。但是，当访问单个哈希表或LPM表时，添加，删除或修改值不能不使用锁在多个线程中完成。锁的另一个替代方法是创建这些表的多个实例，允许每个线程自己的副本。

PMD的RX和TX是DPDK应用程序中最关键的方面，建议不要使用锁，因为它会影响性能。但是请注意，当每个线程在不同的NIC队列上执行I/O时，这些功能可以安全地从多个线程使用。如果多个线程在同一个NIC端口上使用相同的硬件队列，则需要锁定或某种其他形式的互斥。

Ring库的实现基于无锁缓冲算法，保持其原有的线程安全设计。此外，它可以为多个或单个消费者/生产者入队/出队操作提供高性能。mempool库基于DPDK无锁ring库，因此也是多线程安全的。

非性能敏感API
---------------

在第25.1节描述的性能敏感区域之外，DPDK为大多数其他库提供线程安全的API。例如，malloc和memzone功能可以安全地用于多线程和多进程环境中。

PMD的设置和配置不是性能敏感的，但也不是线程安全的。在多线程环境中PMD设置和配置期间的多次读/写可能会被破坏。由于这不是性能敏感的，开发人员可以选择添加自己的层，以提供线程安全的设置和配置。预计在大多数应用中，网络端口的初始配置将由启动时的单个线程完成。

库初始化
---------

建议DPDK库在应用程序启动时在主线程中初始化，而不是随后在转发线程中初始化。但是，DPDK会执行检查，以确保库仅被初始化一次。如果尝试多次初始化，则返回错误。
在多进程情况下，共享内存的配置信息只能由primary process初始化。此后，primary process和secondary process都可以分配/释放最终依赖于rte_malloc或memzone的任何内存对象。

中断线程
----------

DPDK在轮询模式下几乎完全用于Linux用户空间。对于诸如接收PMD链路状态改变通知的某些不经常的操作，可以在主DPDK处理线程外部的附加线程中调用回调。这些函数回调应避免操作也由普通DPDK线程管理的DPDK对象，如果需要这样做，应用程序就可以为这些对象提供适当的锁定或互斥限制。
