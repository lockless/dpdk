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

链路绑定PMD
=============

除了用于物理和虚拟硬件的轮询模式驱动程序（PMD）之外，DPDK还包括一个纯软件库，可将多个物理PMD绑定在一起以创建单个逻辑PMD。

.. figure:: img/bond-overview.*

   Bonded PMDs


Link Bonding PMD库（librte_pmd_bond）支持绑定相同速度和双工的rte_eth_dev端口组，以提供类似于Linux绑定驱动程序中的功能，以允许将多个（从属）NIC聚合到服务器和交换机中的单个逻辑接口。然后，新的聚合的PMD将根据指定的操作模式处理这些接口，以支持冗余链路，容错和/或负载均衡等功能。

librte_pmd_bond库导出一个C语言API，包括用于创建绑定设备的API，以及配置和管理绑定设备及其从属设备的API。

.. note::

    链路绑定PMD库默认情况下在构建配置文件中启用，可以通过设置CONFIG_RTE_LIBRTE_PMD_BOND = n并重新编译DPDK来禁用该库。

链路绑定模式概述
-------------------

目前，Link Bonding PMD库支持以下网卡绑定模式：

*   **轮询（模式0）:**

.. figure:: img/bond-mode-0.*

   Round-Robin (Mode 0)

    轮询模式通过从第一个可用从设备到最后一个的顺序来传输数据包，以提供负载平衡和容错。数据包是从设备批量出队，然后以循环方式提供服务。这种模式不能保证接收到数据包仍然有序，下行流需要能够处理乱序数据包。

*   **主动备份（模式1）:**

.. figure:: img/bond-mode-1.*

   Active Backup (Mode 1)

    在此模式下，在任何时间只有一个从设备处于活动状态，当且仅当当前活跃从设备发生故障时，不同的从设备才会激活，从而为故障设备提供容错。单个逻辑绑定接口的MAC地址只能在一个NIC（端口）上外部可见，以避免网络交换混淆。
    to avoid confusing the network switch.

*   **平衡策略:**

.. figure:: img/bond-mode-2.*

   Balance XOR (Mode 2)

    此模式提供传输负载均衡（基于所选传输策略）和容错。默认策略（layer2）使用基于报文流的源和目标MAC地址的简单计算以及绑定设备可用活动从设备的数量，将数据包分类到特定从设备进行传输。额外支持的备用传输策略是L2 + L3，这将IP源和目标地址用于传输从端口的计算，最终需要支持的策略是L3 + L4层，这使用IP源和目标地址以及TCP / UDP源和目的端口进行计算。

.. note::
    报文的着色差异用于识别由所选择的传输策略计算的不同流分类


*   **广播策略:**

.. figure:: img/bond-mode-3.*

   Broadcast (Mode 3)


    这种模式通过在所有从设备端口上传输数据来实现容错。

*   **链路聚合802.3AD:**

.. figure:: img/bond-mode-4.*

   Link Aggregation 802.3AD (Mode 4)


    此模式根据802.3ad规范提供了动态链路聚合。它使用所选择的均衡传输策略来协商和监视共享相同速度和双工设置的聚合组，以平衡出口流量。

    这种模式的DPDK实现对应用程序提供了一些额外的要求。

    #. 需要调用rte_eth_tx_burst和rte_eth_rx_burst，间隔时间小于100ms。

    #. 对rte_eth_tx_burst的调用必须至少具有2xN的缓冲区大小，其中N是从设备数。这是LACP帧所需的空间。另外LACP数据包也包含在统计信息中，但不会返回给应用程序。

*   **传输负载均衡策略:**

.. figure:: img/bond-mode-5.*

   Transmit Load Balancing (Mode 5)


    此模式提供自适应传输负载均衡。它根据计算的负载动态地更改发送从设备。以100ms的间隔收集统计数据，每10ms调度一次。


实现细节
---------

librte_pmd_bond绑定设备与DPDK API参考中描述的以太网PMD导出的以太网设备API兼容。

链路绑定库支持在EAL初始化期间的应用程序启动时使用 –vdev 选项以及通过C语言 API接口 rte_eth_bond_create函数以编程方式创建绑定的设备。

绑定设备支持使用接口rte_eth_bond_slave_add / rte_eth_bond_slave_remove实现动态添加和移除。

在将从设备添加到绑定设备后，从设备使用rte_eth_dev_stop停止，然后使用rte_eth_dev_configure进行重新配置，也可以使用rte_eth_tx_queue_setup / rte_eth_rx_queue_setup重新配置RX和TX队列，并配置用于配置绑定设备的参数。如果启用绑定设备的RSS，则此模式也将在新从站上启用并进行配置。
设置用于将设备绑定到RSS的多队列模式，使其完全具有RSS功能，因此所有从设备都与其配置同步。此模式旨在提供用于客户端应用程序实现的从站上的RSS配置。

绑定设备存储其自己的RSS设置版本，即RETA，RSS散列函数和RSS密钥，用于设置其从设备。 这就是为了将绑定装置的RSS配置的含义定义为整个绑定（作为一个单元）的所需配置，而不指向任何从属内部。需要确保一致性并使其更具错误性。

用于绑定设备的RSS散列函数集，是所有绑定从站支持的RSS哈希函数的最大集合。RETA大小是其所有RETA大小的GCD，因此即使从属RETA的大小不同，它也可以轻松地用作提供预期行为的模式。如果没有为绑定设备设置RSS键，则在从站上不更改，并且使用设备的默认密钥。

所有设置都通过绑定端口API进行管理，并始终沿一个方向传播（从绑定到从站）。

链路状态改变中断与轮询
~~~~~~~~~~~~~~~~~~~~~~~~

链路绑定设备支持链路状态更改回调的注册，使用rte_eth_dev_callback_register接口，当绑定设备的状态发生更改时，将调用此函数进行处理。例如，在具有3个从设备的绑定设备中，当所有从设备变为不活跃时，链路状态变为DOWN，当一个从设备变为活动状态时，链路状态将变为UP。当单个从设备更改状态并且不满足先前的条件时，没有回调通知。如果用户希望监视单个从设备，则它们必须直接向该从设备注册回调。

链路绑定库还支持不实现链路状态改变中断处理的设备，这是通过使用接口rte_eth_bond_link_monitoring_set设置的周期轮询设备链路状态来实现的，默认轮询间隔为10ms。当设备作为从设备添加到绑定设备时，使用RTE_PCI_DRV_INTR_LSC标志确定设备是支持中断还是通过轮询来监视链路状态。

要求与限制
~~~~~~~~~~~

目前的实现只支持相同速度和双工的设备作为从设备提供给同一个绑定设备。绑定设备从添加到绑定设备的第一个活动从设备上继承这些属性，然后添加到绑定设备的所有其他从设备必须支持这些参数。

绑定设备本身启动之前，必须至少一个从设备。

为了有效地使用绑定设备动态RSS配置功能，还需要所有的从设备都应该是具有RSS能力和支持的，至少有一个通用的散列函数可用于它们。只有当所有从设备支持相同的密钥大小时才可以更改RSS密钥。

为了防止从设备对于如何处理数据包产生矛盾，一旦将设备添加到绑定设备，RSS配置应通过绑定设备API进行管理，而不是直接在从设备上进行管理。

像所有其他PMD一样，PMD导出的所有功能都是无锁功能，假定不会在不同逻辑核心上并行调用以操作同一目标对象。

还应该注意的是，PMD接收功能在它们已经到达绑定设备之后不应该直接在从设备上被调用，因为直接从从设备读取的数据包将不再可用于绑定设备读取。

配置
~~~~~

链路绑定设备使用rte_eth_bond_create API创建，该API需要传入唯一的设备名称，绑定模式和套接字ID来分配绑定设备的资源。绑定设备的其他可配置参数是其从设备，主从，用户定义的MAC地址，如果设备处于平衡XOR模式还需要定义要使用的传输策略。

从设备
^^^^^^^^

绑定设备支持相同速度和双工的设备，最大数目为RTE_MAX_ETHPORTS。每个以太网设备可以作为从设备添加到最多一个绑定设备上。从设备在被加入绑定设备时被重新配置为绑定设备的配置。

绑定还保证将从设备的MAC地址返回到其原始值。

主从
^^^^^^

主从关系用于定义绑定设备处于主动备份模式（模式1）时使用的默认端口。当且仅当当前主端口关闭时，才会使用不同的端口。如果用户没有指定主端口，则默认为添加到绑定设备的第一个端口。

MAC地址
^^^^^^^^^

绑定设备可以配置用户指定的MAC地址，该地址将由某些或所有从设备根据操作模式继承。如果设备处于主动备份模式，则只有主设备具有用户指定的MAC，所有其他从设备将保留其原始MAC地址。在模式0,2,3,4中，所有从站设备都配置了绑定设备的MAC地址。

如果未定义用户定义的MAC地址，则绑定设备将默认使用主从站MAC地址。

均衡XOR模式传输策略
^^^^^^^^^^^^^^^^^^^^^

对于在均衡XOR模式下运行的绑定设备，有3种支持的传输策略。层2，层2 + 3，层3 + 4。

*   **Layer 2:**   默认的传输策略是以太网基于MAC地址的均衡策略。它对包的源MAC地址和目的MAC地址使用简单的XOR计算，然后计算该值的模数，以计算需要输出数据包的从设备。

*   **Layer 2 + 3:** 以太网MAC地址和基于IP地址的均衡策略使用源/目的MAC地址和数据包的源/目的IP地址组合来决定数据包将被传输的从设备端口。

*   **Layer 3 + 4:**  IP地址和UDP基于端口的均衡策略使用源/目的IP地址和数据包的数据包的源/目的UDP端口的组合来决定数据包将被传输的从设备端口。

所有这些策略都支持802.1Q VLAN以太网报文，还支持IPv4，IPv6和UDP协议进行负载分担。

使用链路绑定设备
-----------------

librte_pmd_bond库支持两种设备创建模式，库导出完整的C API或使用EAL命令行在应用程序启动时静态配置链路绑定设备。使用EAL选项，可以透明地使用链接绑定功能，而不需要库API的具体知识，例如，可以使用这种功能来将绑定功能（如主动备份）添加到不了解链接的现有应用程序上。

程序中使用轮询模式驱动
~~~~~~~~~~~~~~~~~~~~~~~~

使用librte_pmd_bond库API，可以在任何应用程序内动态创建和管理链路绑定设备。链路绑定设备使用rte_eth_bond_create API创建，该API需要唯一的设备名称，用于初始化设备的链路绑定模式，以及最后将要分配设备资源的套接字ID。在成功创建绑定设备之后，必须使用通用的以太网设备配置API rte_eth_dev_configure来配置，然后使用rte_eth_tx_queue_setup/rte_eth_rx_queue_setup将要使用的RX和TX队列进行设置。

可以使用rte_eth_bond_slave_add/rte_eth_bond_slave_remove API对链路绑定设备动态添加和删除从设备，但在使用rte_eth_dev_start启动链路绑定设备之前，必须至少添加一个从设备。

绑定设备的链路状态由其从设备的链路状态决定，如果所有从设备链路状态都关闭，或者所有从设备都从链路绑定设备中删除，则绑定设备的链路状态为DOWN。

还可以使用提供的rte_eth_bond_mode_set/get，rte_eth_bond_primary_set/get，rte_eth_bond_mac_set/reset和rte_eth_bond_xmit_policy_set/get来配置/查询绑定设备的控制参数的配置。

在EAL命令行中使用链路绑定设备
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

链路绑定设备可以在应用程序启动时使用–vdev EAL命令行选项创建。 设备名称必须以net_bonding前缀开头，后跟数字或字母。每个设备的名称必须是唯一的。每个设备可以有多个选项，以逗号分隔列表排列。可以多次调用–vdev选项来安排多个设备定义。

设备名称和绑定选项必须用逗号分隔，如下所示：

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,bond_opt0=..,bond opt1=..'--vdev 'net_bond1,bond _opt0=..,bond_opt1=..'

链路绑定EAL选项
^^^^^^^^^^^^^^^^^

只要遵守以下两个规则，可以对多种定义方式组合使用：

*   提供了一种独特的设备名称，格式为net_bondingX，其中X可以是数字和/或字母的任意组合，名称不大于32个字符。

*   每个绑定设备定义提供至少一个从设备。

*   提供了所创建的绑定设备的操作模式。

不同的选项包括：

*   模式：定义设备的绑定模式的整数值。目前支持模式0,1,2,3,4,5（循环，主动备份，平衡，广播，链路聚合，传输负载均衡）。

.. code-block:: console

        mode=2

*   从设备：定义将作为从设备添加到绑定设备的PMD设备。可以多次选择此选项，每个设备要作为从设备添加。物理设备应使用其PCI地址指定，格式为 domain:bus:devid.function。

.. code-block:: console

        slave=0000:0a:00.0,slave=0000:0a:00.1

*   主设备：定义主从端口的可选参数用于主动备份模式，以便在数据TX / RX可用时选择主从机。 当主端口未被用户定义时，主端口也用于选择要使用的MAC地址。如果未指定该设备，则默认为添加到设备的第一个从设备。主设备必须是绑定设备的从设备。

.. code-block:: console

        primary=0000:0a:00.0

*   Socket_id：可选参数，用于选择NUMA设备上将分配绑定设备资源的哪个套接字。

.. code-block:: console

        socket_id=0

*   Mac：可选参数，选择链路绑定设备的MAC地址，这将覆盖主设备的值。

.. code-block:: console

        mac=00:1e:67:1d:fd:1d

*   xmit_policy：绑定设备处于均衡模式时定义传输策略的可选参数。如果没有用户指定，则默认为l2（第2层）转发，其他可用的传输策略为l23（第2层+3层）和l34层（3 + 4层）。

.. code-block:: console

        xmit_policy=l23

*   lsc_poll_period_ms：可选参数，用于定义不支持lsc中断的设备以毫秒为单位的轮询间隔，检查设备链路状态的变化。

.. code-block:: console

        lsc_poll_period_ms=100

*   ups delay：可选参数，增加了设备链路状态传播的延迟（以毫秒为单位），默认情况下该参数为零。

.. code-block:: console

        up_delay=10

*   down_delay：可选参数，以毫秒为单位，将设备链路状态DOWN的传播延迟，默认情况下，该参数为零。

.. code-block:: console

        down_delay=50

使用实例
^^^^^^^^^

以轮询模式创建一个绑定设备，两个从设备由其PCI地址指定：

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=0, slave=0000:00a:00.01,slave=0000:004:00.00' -- --port-topology=chained

以轮询模式创建一个绑定设备，其中两个从站由其PCI地址和覆盖MAC地址指定：

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=0, slave=0000:00a:00.01,slave=0000:004:00.00,mac=00:1e:67:1d:fd:1d' -- --port-topology=chained

Create a bonded device in active backup mode with two slaves specified, and a primary slave specified by their PCI addresses:

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=1, slave=0000:00a:00.01,slave=0000:004:00.00,primary=0000:00a:00.01' -- --port-topology=chained

在平衡模式下创建一个绑定设备，其中两个从站由其PCI地址指定，第3 + 4层传输策略：

.. code-block:: console

    $RTE_TARGET/app/testpmd -l 0-3 -n 4 --vdev 'net_bond0,mode=2, slave=0000:00a:00.01,slave=0000:004:00.00,xmit_policy=l34' -- --port-topology=chained
