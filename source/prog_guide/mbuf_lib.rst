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

.. _Mbuf_Library:

Mbuf 库
========

Mbuf库提供了申请和释放mbufs的功能，DPDK应用程序使用这些buffer存储消息缓冲。
消息缓冲存储在mempool中，使用 :ref:`Mempool Library <Mempool_Library>` 。

数据结构rte_mbuf可以承载网络数据包buffer或者通用控制消息buffer(由CTRL_MBUF_FLAG指示)。
也可以扩展到其他类型。
rte_mbuf头部结构尽可能小，目前只使用两个缓存行，最常用的字段位于第一个缓存行中。

Packet Buffer 设计
--------------------

为了存储数据包数据(报价协议头部), 考虑了两种方法：

#.  在单个存储buffer中嵌入metadata，后面跟着数据包数据固定大小区域

#.  为metadata和报文数据分别使用独立的存储buffer。

第一种方法的优点是他只需要一个操作来分配/释放数据包的整个存储表示。
但是，第二种方法更加灵活，并允许将元数据的分配与报文数据缓冲区的分配完全分离。

DPDK选择了第一种方法。
Metadata包含诸如消息类型，长度，到数据开头的偏移量等控制信息，以及允许缓冲链接的附加mbuf结构指针。

用于承载网络数据包buffer的消息缓冲可以处理需要多个缓冲区来保存完整数据包的情况。
许多通过下一个字段链接在一起的mbuf组成的jumbo帧，就是这种情况。

对于新分配的mbuf，数据开始的区域是buffer之后 RTE_PKTMBUF_HEADROOM 字节的位置，这是缓存对齐的。
Message buffers可以在系统中的不同实体中携带控制信息，报文，事件等。
Message buffers也可以使用起buffer指针来指向其他消息缓冲的数据字段或其他数据结构。

:numref:`figure_mbuf1` and :numref:`figure_mbuf2` 显示了其中个一些场景。

.. _figure_mbuf1:

.. figure:: img/mbuf1.*

   An mbuf with One Segment


.. _figure_mbuf2:

.. figure:: img/mbuf2.*

   An mbuf with Three Segments


Buffer Manager实现了一组相当标准的buffer访问操作来操纵网络数据包。

存储在Mempool中的Buffer
-------------------------

Buffer Manager 使用 :ref:`Mempool Library <Mempool_Library>` 来申请buffer。
因此确保了数据包头部均衡分布到信道上并进行L3处理。
mbuf中包含一个字段，用于表示它从哪个池中申请出来。
当调用 rte_ctrlmbuf_free(m) 或 rte_pktmbuf_free(m)，mbuf被释放到原来的池中。

构造函数
------------

Packet 及 control mbuf构造函数由API提供。
接口rte_pktmbuf_init() 及 rte_ctrlmbuf_init() 初始化mbuf结构中的某些字段，这些字段一旦创建将不会被用户修改（如mbuf类型、源池、缓冲区起始地址等）。
此函数在池创建时作为rte_mempool_create()函数的回掉函数给出。

申请及释放 mbufs
------------------

分配一个新mbuf需要用户指定从哪个池中申请。
对于任意新分配的mbuf，它包含一个段，长度为0。
缓冲区到数据的偏移量被初始化，以便使得buffer具有一些字节（RTE_PKTMBUF_HEADROOM）的headroom。

释放mbuf意味着将其返回到原始的mempool。
当mbuf的内容存储在一个池中（作为一个空闲的mbuf）时，mbuf的内容不会被修改。
由构造函数初始化的字段不需要在mbuf分配时重新初始化。

当释放包含多个段的数据包mbuf时，他们都被释放，并返回到原始mempool。

操作 mbufs 
------------

这个库提供了一些操作数据包mbuf中的数据的功能。 例如：

    *  获取数据长度

    *  获取指向数据开始位置的指针

    *  数据前插入数据

    *  数据之后添加数据

    *  删除缓冲区开头的数据(rte_pktmbuf_adj())

    *  删除缓冲区末尾的数据(rte_pktmbuf_trim()) 详细信息请参阅 *DPDK API Reference* 

元数据信息
-------------

部分信息由网络驱动程序检索并存储在mbuf中使得处理更简单。
例如，VLAN、RSS哈希结果(参见 :ref:`Poll Mode Driver <Poll_Mode_Driver>`)及校验和由硬件计算的标志等。

mbuf中还包含数据源端口和报文链中mbuf数目。
对于链接的mbuf，只有链的第一个mbuf存储这个元信息。

例如，对于IEEE1588数据包，RX侧就是这种情况，时间戳机制，VLAN标记和IP校验和计算。
在TX端，应用程序还可以将一些处理委托给硬件。 例如，PKT_TX_IP_CKSUM标志允许卸载IPv4校验和的计算。

以下示例说明如何在vxlan封装的tcp数据包上配置不同的TX卸载：``out_eth/out_ip/out_udp/vxlan/in_eth/in_ip/in_tcp/payload``

- 计算out_ip的校验和::

    mb->l2_len = len(out_eth)
    mb->l3_len = len(out_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
    set out_ip checksum to 0 in the packet

  配置DEV_TX_OFFLOAD_IPV4_CKSUM支持在硬件计算。

- 计算out_ip 和 out_udp的校验和::

    mb->l2_len = len(out_eth)
    mb->l3_len = len(out_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_UDP_CKSUM
    set out_ip checksum to 0 in the packet
    set out_udp checksum to pseudo header using rte_ipv4_phdr_cksum()

  配置DEV_TX_OFFLOAD_IPV4_CKSUM 和 DEV_TX_OFFLOAD_UDP_CKSUM支持在硬件上计算。

- 计算in_ip的校验和::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM
    set in_ip checksum to 0 in the packet

  这以情况1类似，但是l2_len不同。
  配置DEV_TX_OFFLOAD_IPV4_CKSUM支持硬件计算。
  注意，只有外部L4校验和为0时才可以工作。

- 计算in_ip 和 in_tcp的校验和::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CSUM | PKT_TX_TCP_CKSUM
    在报文中设置in_ip校验和为0
    使用rte_ipv4_phdr_cksum()将in_tcp校验和设置为伪头

  这与情况2类似，但是l2_len不同。
  配置DEV_TX_OFFLOAD_IPV4_CKSUM 和 DEV_TX_OFFLOAD_TCP_CKSUM支持硬件实现。
  注意，只有外部L4校验和为0才能工作。

- segment inner TCP::

    mb->l2_len = len(out_eth + out_ip + out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->l4_len = len(in_tcp)
    mb->ol_flags |= PKT_TX_IPV4 | PKT_TX_IP_CKSUM | PKT_TX_TCP_CKSUM | PKT_TX_TCP_SEG;
    在报文中设置in_ip校验和为0
    将in_tcp校验和设置为伪头部，而不使用IP载荷长度

  配置DEV_TX_OFFLOAD_TCP_TSO支持硬件实现。
  注意，只有L4校验和为0时才能工作。

- 计算out_ip, in_ip, in_tcp的校验和::

    mb->outer_l2_len = len(out_eth)
    mb->outer_l3_len = len(out_ip)
    mb->l2_len = len(out_udp + vxlan + in_eth)
    mb->l3_len = len(in_ip)
    mb->ol_flags |= PKT_TX_OUTER_IPV4 | PKT_TX_OUTER_IP_CKSUM  | PKT_TX_IP_CKSUM |  PKT_TX_TCP_CKSUM;
    设置 out_ip 校验和为0
    设置 in_ip 校验和为0
    使用rte_ipv4_phdr_cksum()设置in_tcp校验和为伪头部

  配置DEV_TX_OFFLOAD_IPV4_CKSUM, DEV_TX_OFFLOAD_UDP_CKSUM 和 DEV_TX_OFFLOAD_OUTER_IPV4_CKSUM支持硬件实现。

Flage标记的意义在mbuf API文档(rte_mbuf.h)中有详细描述。
更多详细信息还可以参阅testpmd 源码(特别是csumonly.c)。

.. _direct_indirect_buffer:

直接及间接 Buffers 
--------------------

直接缓冲区是指缓冲区完全独立。
间接缓冲区的行为类似于直接缓冲区，但缓冲区的指针和数据便宜量指的是另一个直接缓冲区的数据。
这在数据包需要复制或分段的情况下是很有用的，因为间接缓冲区提供跨越多个缓冲区重用相同数据包数据的手段。

当使用接口 ``rte_pktmbuf_attach()`` 函数将缓冲区附加到直接缓冲区时，该缓冲区变成间接缓冲区。
每个缓冲区有一个引用计数器字段，每当直接缓冲区附加一个间接缓冲区时，直接缓冲区上的应用计数器递增。
类似的，每当间接缓冲区被分裂时，直接缓冲区上的引用计数器递减。
如果生成的引用计数器为0，则直接缓冲区将被释放，因为它不再使用。

处理间接缓冲区时需要注意几件事情。
首先，间接缓冲区从不附加到另一个间接缓冲区。
尝试将缓冲区A附加到间接缓冲区B（且B附加到C上了），将使得rte_pktmbuf_attach() 自动将A附加到C上。
其次，为了使缓冲区变成间接缓冲区，其引用计数必须等于1，也就是说它不能被另一个间接缓冲区引用。
最后，不可能将间接缓冲区重新链接到直接缓冲区（除非它已经被分离了）。

虽然可以使用推荐的rte_pktmbuf_attach（）和rte_pktmbuf_detach（）函数直接调用附加/分离操作，
但建议使用更高级的rte_pktmbuf_clone（）函数，该函数负责间接缓冲区的正确初始化，并可以克隆具有多个段的缓冲区。

由于间接缓冲区不应该实际保存任何数据，间接缓冲区的内存池应配置为指示减少的内存消耗。
可以在几个示例应用程序中找到用于间接缓冲区的内存池（以及间接缓冲区的用例示例）的初始化示例，例如IPv4组播示例应用程序。


调试
-----

在调试模式 (CONFIG_RTE_MBUF_DEBUG使能)下，mbuf库的功能在任何操作之前执行完整性检查(如缓冲区检查、类型错误等)。

用例
------

所有网络应用程序都应该使用mbufs来传输网络数据包。
