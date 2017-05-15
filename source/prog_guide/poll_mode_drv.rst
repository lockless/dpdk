..  BSD LICENSE
    Copyright(c) 2010-2015 Intel Corporation. All rights reserved.
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

.. _Poll_Mode_Driver:

轮询模式驱动
================

DPDK包括Gigabit、10Gigabit 及 40Gigabit 和半虚拟化IO的轮询模式驱动程序。

轮询模式驱动程序(PMD)由通过在用户空间中运行的BSD驱动提供的API组成，以配置设备及各自的队列。

此外，PMD直接访问 RX 和 TX 描述符，且不会有任何中断（链路状态更改中断除外）产生，这可以保证在用户空间应用程序中快速接收，处理和传送数据包。
本节介绍PMD的要求、设计原则和高级架构，并介绍了以太网PMD的对外通用API。

要求及假设条件
-----------------

DPDK环境支持两种模式的数据包处理，RTC和pipeline：

*   在 *run-to-completion*  模式中，通过调用API来轮询指定端口的RX描述符以获取报文。
    紧接着，在同一个core上处理报文，并通过API调用将报文放到接口的TX描述符中以发送报文。

*   在 *pipe-line*  模式中，一个core轮询一个或多个接口的RX描述符以获取报文。然后报文经由ring被其他core处理。
    其他core可以继续处理报文，最终报文被放到TX描述符中以发送出去。

在同步 run-to-completion 模式中，每个逻辑和处理数据包的流程包括以下步骤：

*   通过PMD报文接收API来获取报文

*   一次性处理每个数据报文，直到转发阶段

*   通过PMD发包API将报文发送出去

相反地，在异步的pipline模式中，一些逻辑核可能专门用于接收报文，其他逻辑核用于处理前面收到的报文。
收到的数据包通过报文ring在逻辑核之间交换。
数据包收包过程包括以下步骤：

*   通过PMD收包API获取报文

*   通过数据包队列想逻辑核提供接收到的数据包

数据包处理过程包括以下步骤：

*   从数据包队列中获取数据包

*   处理接收到的数据包，直到重新发送出去

为了避免任何不必要的中断处理开销，执行环境不得使用任何异步通知机制。即便有需要，也应该尽量使用ring来引入通知信息。

在多核环境中避免锁竞争是一个关键问题。
为了解决这个问题，PMD旨在尽可能地使用每个core的私有资源。
例如，PMD每个端口维护每个core单独的传输队列。
同样的，端口的每个接收队列都被分配给单个逻辑核并由其轮询。

为了适用NUMA架构，内存管理旨在为每个逻辑核分配本地（相同插槽）中的专用缓冲池，以最大限度地减少远程内存访问。
数据包缓冲池的配置应该考虑到DIMMs、channels 和 ranks等底层物理内存架构。
应用程序必须确保在内存池创建时给出合适的参数。具体内容参阅 :ref:`Mempool Library <Mempool_Library>` 。

设计原则
-----------

Ethernet* PMDs的API和架构设计遵考虑到以下原则。

PMDs 必须能够帮助上层的应用实现全局的策略。
反之，不能阻止或妨碍上层应用的实施。

例如，PMD的发送和接收函数都有大量的报文或描述符需要轮询。
这运行RTC处理协议栈通过不同的全局循环策略静态修护或动态调整其行为如：

*   立即接收，处理并以零碎的方式一次传送数据包。

*   尽可能所的接收数据包，然后处理所有数据包，再发送。

*   接收给定的最大量的数据包，处理接收的数据包，累加，最后将累加的数据包发送出去。

为了实现最优性能，需要考考整体软件设计选择和纯软件优化技术，并与可用的低层次硬件优化功能（如CPU缓存属性、总线速度、NIC PCI带宽等）进行考虑和平衡。
报文传输的情况就是突发性网络报文处理是软硬件权衡问题的一个例子。
在初始情况下，PMD只能导出一个 rte_eth_tx_one 函数，以便在给定的队列上一次传输一个数据包。
最重要的是，可以轻松构建一个 rte_eth_tx_burst 函数，循环调用 rte_eth_tx_one 函数以便一次传输多个数据包。
然而，PMD有效地实现了 rte_eth_tx_burst 函数，以通过以下优化来最小化每个数据包的驱动级传输开销：

*   在多个数据包之间共享调用 rte_eth_tx_one 函数的非摊销成本。

*   启用 rte_eth_tx_burst 函数以利用burst-oriented 硬件特性（缓存数据预取、使用NIC头/尾寄存器）以最小化每个数据包的CPU周期数，
    例如，通过避免对环形缓传输描述符的不必要的读取寄存器访问，或通过系统地使用精确匹配告诉缓存行边界大小的指针数组。

*   使用burst-oriented软件优化技术来移除失败的操作结果，如ring索引的回滚。

还通过API引入了Burst-oriented函数，这些函数在PMD服务中密集使用。
这些函数特别适用于NIC ring的缓冲区分配器，他们提供一次分配/释放多个缓冲区的功能。
例如，一个 mbuf_multiple_alloc 函数返回一个指向 rte_mbuf 缓冲区的指针数组，它可以在向ring添加多个描述符来加速PMD的接收轮询功能。

逻辑核、内存及网卡队列的联系
------------------------------

当处理器的逻辑核和接口利用其本地存储时，DPDK提供NUMA支持，以提供更好的性能。
因此，与本地PCIE接口相关的mbuf分配应从本地内存中创建的内存池中申请。
如果可能，缓冲区应该保留在本地处理器上以获取最佳性能，并且应使用从本地内存中分配的mempool中申请的mbuf来填充RX和TX缓冲区描述符。

如果数据包或数据操作在本地内存中，而不是在远程处理器内存上，则RTC模型也会运行得更好。
只要所有使用的逻辑核位于同一个处理器上，pipeline模型也将获得更好的性能。

所个逻辑核不应共享接口的接收或发送队列，因为这将需要全局上锁保护，而导致性能下降。

设备标识及配置
----------------

设备标识
~~~~~~~~~~~

每个NIC端口（总线/桥、设备、功能）由其PCI标识符唯一指定。该PCI标识符在DPDK初始化时执行的PCI探测/枚举功能分配。
根据PCI标识符，NIC端口被分配了两个其他的表示：

*   一个端口索引，用于在PMD API导出的所有函数中指定NIC端口

*   端口名称，用于在控制消息中指定端口，主要用于管理和调试目的。为了便于使用，端口名称包括端口索引。

设备配置
~~~~~~~~~~

每个NIC端口的配置包括以下步骤：

*   分配 PCI 资源

*   将硬件复位为公知的默认状态

*   设置PHY和链路

*   初始化统计计数器

PMD API还必须导出函数用于启动/终止端口的全部组播功能，并且可以在混杂模式下设置/取消设置端口。

某些硬件卸载功能必须通过特定的配置参数在端口初始化时单独配置。
例如，接收侧缩放（RSS）和数据中心桥接（DCB）功能就是这种情况。

On-the-Fly Configuration
~~~~~~~~~~~~~~~~~~~~~~~~

All device features that can be started or stopped "on the fly" (that is, without stopping the device) do not require the PMD API to export dedicated functions for this purpose.

All that is required is the mapping address of the device PCI registers to implement the configuration of these features in specific functions outside of the drivers.

For this purpose,
the PMD API exports a function that provides all the information associated with a device that can be used to set up a given device feature outside of the driver.
This includes the PCI vendor identifier, the PCI device identifier, the mapping address of the PCI device registers, and the name of the driver.

The main advantage of this approach is that it gives complete freedom on the choice of the API used to configure, to start, and to stop such features.

As an example, refer to the configuration of the IEEE1588 feature for the Intel® 82576 Gigabit Ethernet Controller and
the Intel® 82599 10 Gigabit Ethernet Controller controllers in the testpmd application.

Other features such as the L3/L4 5-Tuple packet filtering feature of a port can be configured in the same way.
Ethernet* flow control (pause frame) can be configured on the individual port.
Refer to the testpmd source code for details.
Also, L4 (UDP/TCP/ SCTP) checksum offload by the NIC can be enabled for an individual packet as long as the packet mbuf is set up correctly. See `Hardware Offload`_ for details.

传输队列配置
~~~~~~~~~~~~~~

Each transmit queue is independently configured with the following information:

*   The number of descriptors of the transmit ring

*   The socket identifier used to identify the appropriate DMA memory zone from which to allocate the transmit ring in NUMA architectures

*   The values of the Prefetch, Host and Write-Back threshold registers of the transmit queue

*   The *minimum* transmit packets to free threshold (tx_free_thresh).
    When the number of descriptors used to transmit packets exceeds this threshold, the network adaptor should be checked to see if it has written back descriptors.
    A value of 0 can be passed during the TX queue configuration to indicate the default value should be used.
    The default value for tx_free_thresh is 32.
    This ensures that the PMD does not search for completed descriptors until at least 32 have been processed by the NIC for this queue.

*   The *minimum*  RS bit threshold. The minimum number of transmit descriptors to use before setting the Report Status (RS) bit in the transmit descriptor.
    Note that this parameter may only be valid for Intel 10 GbE network adapters.
    The RS bit is set on the last descriptor used to transmit a packet if the number of descriptors used since the last RS bit setting,
    up to the first descriptor used to transmit the packet, exceeds the transmit RS bit threshold (tx_rs_thresh).
    In short, this parameter controls which transmit descriptors are written back to host memory by the network adapter.
    A value of 0 can be passed during the TX queue configuration to indicate that the default value should be used.
    The default value for tx_rs_thresh is 32.
    This ensures that at least 32 descriptors are used before the network adapter writes back the most recently used descriptor.
    This saves upstream PCIe* bandwidth resulting from TX descriptor write-backs.
    It is important to note that the TX Write-back threshold (TX wthresh) should be set to 0 when tx_rs_thresh is greater than 1.
    Refer to the Intel® 82599 10 Gigabit Ethernet Controller Datasheet for more details.

The following constraints must be satisfied for tx_free_thresh and tx_rs_thresh:

*   tx_rs_thresh must be greater than 0.

*   tx_rs_thresh must be less than the size of the ring minus 2.

*   tx_rs_thresh must be less than or equal to tx_free_thresh.

*   tx_free_thresh must be greater than 0.

*   tx_free_thresh must be less than the size of the ring minus 3.

*   For optimal performance, TX wthresh should be set to 0 when tx_rs_thresh is greater than 1.

One descriptor in the TX ring is used as a sentinel to avoid a hardware race condition, hence the maximum threshold constraints.

.. note::

    When configuring for DCB operation, at port initialization, both the number of transmit queues and the number of receive queues must be set to 128.

释放 Tx 缓存
~~~~~~~~~~~~~~

Many of the drivers do not release the mbuf back to the mempool, or local cache,
immediately after the packet has been transmitted.
Instead, they leave the mbuf in their Tx ring and
either perform a bulk release when the ``tx_rs_thresh`` has been crossed
or free the mbuf when a slot in the Tx ring is needed.

An application can request the driver to release used mbufs with the ``rte_eth_tx_done_cleanup()`` API.
This API requests the driver to release mbufs that are no longer in use,
independent of whether or not the ``tx_rs_thresh`` has been crossed.
There are two scenarios when an application may want the mbuf released immediately:

* When a given packet needs to be sent to multiple destination interfaces
  (either for Layer 2 flooding or Layer 3 multi-cast).
  One option is to make a copy of the packet or a copy of the header portion that needs to be manipulated.
  A second option is to transmit the packet and then poll the ``rte_eth_tx_done_cleanup()`` API
  until the reference count on the packet is decremented.
  Then the same packet can be transmitted to the next destination interface.
  The application is still responsible for managing any packet manipulations needed
  between the different destination interfaces, but a packet copy can be avoided.
  This API is independent of whether the packet was transmitted or dropped,
  only that the mbuf is no longer in use by the interface.

* Some applications are designed to make multiple runs, like a packet generator.
  For performance reasons and consistency between runs,
  the application may want to reset back to an initial state
  between each run, where all mbufs are returned to the mempool.
  In this case, it can call the ``rte_eth_tx_done_cleanup()`` API
  for each destination interface it has been using
  to request it to release of all its used mbufs.

To determine if a driver supports this API, check for the *Free Tx mbuf on demand* feature
in the *Network Interface Controller Drivers* document.

硬件卸载
~~~~~~~~~~

Depending on driver capabilities advertised by
``rte_eth_dev_info_get()``, the PMD may support hardware offloading
feature like checksumming, TCP segmentation or VLAN insertion.

The support of these offload features implies the addition of dedicated
status bit(s) and value field(s) into the rte_mbuf data structure, along
with their appropriate handling by the receive/transmit functions
exported by each PMD. The list of flags and their precise meaning is
described in the mbuf API documentation and in the in :ref:`Mbuf Library
<Mbuf_Library>`, section "Meta Information".

Poll Mode Driver API
--------------------

Generalities
~~~~~~~~~~~~

By default, all functions exported by a PMD are lock-free functions that are assumed
not to be invoked in parallel on different logical cores to work on the same target object.
For instance, a PMD receive function cannot be invoked in parallel on two logical cores to poll the same RX queue of the same port.
Of course, this function can be invoked in parallel by different logical cores on different RX queues.
It is the responsibility of the upper-level application to enforce this rule.

If needed, parallel accesses by multiple logical cores to shared queues can be explicitly protected by dedicated inline lock-aware functions
built on top of their corresponding lock-free functions of the PMD API.

Generic Packet Representation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A packet is represented by an rte_mbuf structure, which is a generic metadata structure containing all necessary housekeeping information.
This includes fields and status bits corresponding to offload hardware features, such as checksum computation of IP headers or VLAN tags.

The rte_mbuf data structure includes specific fields to represent, in a generic way, the offload features provided by network controllers.
For an input packet, most fields of the rte_mbuf structure are filled in by the PMD receive function with the information contained in the receive descriptor.
Conversely, for output packets, most fields of rte_mbuf structures are used by the PMD transmit function to initialize transmit descriptors.

The mbuf structure is fully described in the :ref:`Mbuf Library <Mbuf_Library>` chapter.

Ethernet Device API
~~~~~~~~~~~~~~~~~~~

The Ethernet device API exported by the Ethernet PMDs is described in the *DPDK API Reference*.

Extended Statistics API
~~~~~~~~~~~~~~~~~~~~~~~

The extended statistics API allows each individual PMD to expose a unique set
of statistics. Accessing these from application programs is done via two
functions:

* ``rte_eth_xstats_get``: Fills in an array of ``struct rte_eth_xstat``
  with extended statistics.
* ``rte_eth_xstats_get_names``: Fills in an array of
  ``struct rte_eth_xstat_name`` with extended statistic name lookup
  information.

Each ``struct rte_eth_xstat`` contains an identifier and value pair, and
each ``struct rte_eth_xstat_name`` contains a string. Each identifier
within the ``struct rte_eth_xstat`` lookup array must have a corresponding
entry in the ``struct rte_eth_xstat_name`` lookup array. Within the latter
the index of the entry is the identifier the string is associated with.
These identifiers, as well as the number of extended statistic exposed, must
remain constant during runtime. Note that extended statistic identifiers are
driver-specific, and hence might not be the same for different ports.

A naming scheme exists for the strings exposed to clients of the API. This is
to allow scraping of the API for statistics of interest. The naming scheme uses
strings split by a single underscore ``_``. The scheme is as follows:

* direction
* detail 1
* detail 2
* detail n
* unit

Examples of common statistics xstats strings, formatted to comply to the scheme
proposed above:

* ``rx_bytes``
* ``rx_crc_errors``
* ``tx_multicast_packets``

The scheme, although quite simple, allows flexibility in presenting and reading
information from the statistic strings. The following example illustrates the
naming scheme:``rx_packets``. In this example, the string is split into two
components. The first component ``rx`` indicates that the statistic is
associated with the receive side of the NIC.  The second component ``packets``
indicates that the unit of measure is packets.

A more complicated example: ``tx_size_128_to_255_packets``. In this example,
``tx`` indicates transmission, ``size``  is the first detail, ``128`` etc are
more details, and ``packets`` indicates that this is a packet counter.

Some additions in the metadata scheme are as follows:

* If the first part does not match ``rx`` or ``tx``, the statistic does not
  have an affinity with either receive of transmit.

* If the first letter of the second part is ``q`` and this ``q`` is followed
  by a number, this statistic is part of a specific queue.

An example where queue numbers are used is as follows: ``tx_q7_bytes`` which
indicates this statistic applies to queue number 7, and represents the number
of transmitted bytes on that queue.
