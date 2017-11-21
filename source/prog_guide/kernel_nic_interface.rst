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

.. _kni:

内核网络接口卡接口
====================

DPDK Kernel NIC Interface（KNI）允许用户空间应用程序访问Linux *控制面。

使用DPDK KNI的好处是：

*   比现有的Linux TUN / TAP接口更快（通过消除系统调用和copy_to_user()/copy_from_user()操作）。

*   允许使用标准Linux网络工具（如ethtool，ifconfig和tcpdump）管理DPDK端口。

*   允许与内核网络堆栈的接口。

使用DPDK内核NIC接口的应用程序的组件如图所示。 :numref:`figure_kernel_nic_intf`.

.. _figure_kernel_nic_intf:

.. figure:: img/kernel_nic_intf.*

   Components of a DPDK KNI Application


DPDK KNI内核模块
------------------

KNI内核可加载模块支持两种类型的设备：

*   其他设备：(/dev/kni) 

    *   创建网络设备（通过ioctl调用）。

    *   维护所有KNI实例共享的内核线程上下文（模拟网络驱动程序的RX端）。

    *   对于单内核线程模式，维护所有KNI实例共享的内核线程上下文（模拟网络驱动程序的RX端）。

    *   对于多个内核线程模式，为每个KNI实例（模拟新驱动程序的RX侧）维护一个内核线程上下文。

*   网络设备：

    *   通过实现由struct net_device定义的诸如netdev_ops，header_ops，ethtool_ops之类的几个操作提供的Net功能，包括支持DPDK mbufs和FIFO。

    *   接口名称由用户空间提供。

    *   MAC地址可以是真正的NIC MAC地址或随机的。

KNI创建及删除
--------------

KNI接口由DPDK应用程序动态创建。接口名称和FIFO详细信息由应用程序通过ioctl调用使用rte_kni_device_info结构提供，该结构包含：

*   接口名称。

*   相关FIFO的相应存储器的物理地址。

*   Mbuf mempool详细信息，包括物理和虚拟（计算mbuf指针的偏移量）。

*   PCI信息。

*   Core。

有关详细信息，请参阅DPDK源代码中的rte_kni_common.h。

物理地址将重新映射到内核地址空间，并存储在单独的KNI上下文中。

内核RX线程（单线程和多线程模式）的亲和力由force_bind和core_id配置参数控制。

创建后，DPDK应用程序可以动态删除KNI接口。此外，所有未删除的KNI接口将在杂项设备（DPDK应用程序关闭时）的释放操作中被删除。

DPDK缓冲区流
--------------

为了最小化在内核空间中运行的DPDK代码的数量，mbuf mempool仅在用户空间中进行管理。内核模块可以感知mbufs，但是所有mbuf分配和释放操作将仅由DPDK应用程序处理。

:numref:`figure_pkt_flow_kni` shows a typical scenario with packets sent in both directions.

.. _figure_pkt_flow_kni:

.. figure:: img/pkt_flow_kni.*

   Packet Flow via mbufs in the DPDK KNI


用例: Ingress
-----------------

在DPDK RX侧，mbuf由PMD在RX线程上下文中分配。该线程将mbuf入队到rx_q FIFO中。 KNI线程将轮询所有KNI活动设备。如果mbuf出队，它将被转换为sk_buff，并通过netif_rx()发送到网络协议栈。必须释放出队的mbuf，将指针返回到free_q FIFO中。

RX线程在相同的主循环中轮询该FIFO，并在出队之后释放mbuf。

用例: Egress
----------------

对于数据包出口，DPDK应用程序必须首先入队几个mbufs才能在内核端创建一个mbuf缓存。

通过调用kni_net_tx()回调，从Linux网络堆栈接收数据包。mbuf出队（因为使用缓存，所以无需等待），并填充了来自sk_buff的数据。然后释放sk_buff，并将mbuf发送到tx_q FIFO。

DPDK TX线程执行mbuf出队，并将其发送到PMD（通过rte_eth_tx_burst()）。 然后将mbuf放回缓存中。

以太网工具
-----------

Ethtool是Linux专用工具，在内核中具有相应的支持，每个网络设备必须为支持的操作注册自己的回调。目前的实现使用igb / ixgbe修改的Linux驱动程序进行ethtool支持。i40e和VM（VF或EM设备）不支持Ethtool。

链路状态及MTU改变
-------------------

链路状态和MTU变化是通常通过ifconfig完成的网络接口操作。该请求是从内核端（在ifconfig进程的上下文中）发起的，由用户空间DPDK应用程序处理。应用程序轮询请求，调用应用程序处理程序并将响应返回到内核空间。

应用处理程序可以在创建接口时注册，也可以在运行时再注册/卸载。这提供了多进程方案（其中KNI在primary process中创建，在secondary process中处理回调）的灵活性。约束是单个进程可以注册和处理请求。
