如何获取Intel平台上网卡的最佳性能
=================================

本文档一步一步教你如何在Intel平台上运行DPDK程序以获取最佳性能。


硬件及存储需求
--------------

为了获得最佳性能，请使用Intel Xeon级服务器系统，如Ivy Bridge，Haswell或更高版本。

确保每个内存通道至少插入一个内存DIMM，每个内存通道的内存大小至少为4GB。
**Note**: 这对性能有最直接的影响。

可以通过使用 ``dmidecode`` 来检查内存配置::

      dmidecode -t memory | grep Locator

      Locator: DIMM_A1
      Bank Locator: NODE 1
      Locator: DIMM_A2
      Bank Locator: NODE 1
      Locator: DIMM_B1
      Bank Locator: NODE 1
      Locator: DIMM_B2
      Bank Locator: NODE 1
      ...
      Locator: DIMM_G1
      Bank Locator: NODE 2
      Locator: DIMM_G2
      Bank Locator: NODE 2
      Locator: DIMM_H1
      Bank Locator: NODE 2
      Locator: DIMM_H2
      Bank Locator: NODE 2

上面的示例输出显示共有8个通道，从 ``A`` 到 ``H``，每个通道都有2个DIMM。

你也可以使用 ``dmidecode`` 来确定内存频率::

      dmidecode -t memory | grep Speed

      Speed: 2133 MHz
      Configured Clock Speed: 2134 MHz
      Speed: Unknown
      Configured Clock Speed: Unknown
      Speed: 2133 MHz
      Configured Clock Speed: 2134 MHz
      Speed: Unknown
      ...
      Speed: 2133 MHz
      Configured Clock Speed: 2134 MHz
      Speed: Unknown
      Configured Clock Speed: Unknown
      Speed: 2133 MHz
      Configured Clock Speed: 2134 MHz
      Speed: Unknown
      Configured Clock Speed: Unknown

输出显示2133 MHz（DDR4）和未知（不存在）的速度。这与先前的输出一致，表明每个通道都有一个存储。

网卡需求
~~~~~~~~

使用 `DPDK supported <http://dpdk.org/doc/nics>` 描述的高端NIC，如Intel XL710 40GbE。

确保每个网卡已经更新最新版本的NVM/固件。

使用PCIe Gen3 插槽，如 Gen3 ``x8`` 或者 Gen3 ``x16`` ，因为PCIe Gen2 插槽不能提供2 x 10GbE或更高的带宽。
可以使用 ``lspci`` 命令来检查PCI插槽的速率::

      lspci -s 03:00.1 -vv | grep LnkSta

      LnkSta: Speed 8GT/s, Width x8, TrErr- Train- SlotClk+ DLActive- ...
      LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ ...

当将NIC插入PCI插槽时，需要查看屏幕输出，如 CPU0 或 CPU1，以指示连接的插槽。

同时应该注意NUMA，如果使用不同网卡的2个或更多端口，最好确保这些NIC在同一个CPU插槽上，下面进一步展示了如何确定这一点。


BIOS 设置
~~~~~~~~~

以下是关于BIOS设置的一些建议。不同的平台可能会有不同的名字，因此如下仅用于参考:

#. 开始之前，请考虑将所有BIOS设置为默认值

#. 禁用所有省电选项，如电源性能调整、CPU P-State, CPU C3 Report and CPU C6 Report。

#. 选择 **Performance** 作为CPU电源及性能策略。

#. 禁用Turbo Boost以确保性能缩放随着内核数量的增加而增加。

#. 将内存频率设置为最高可用的值，NOT auto。

#. 当测试NIC的物理功能时，禁用所有的虚拟化选项，如果要使用VFIO，请打开 ``VT-d`` if you wants to use VFIO.


Linux引导选项
~~~~~~~~~~~~~

以下是GRUB启动选项的一些建议配置:

#. 使用默认的grub文件作为起点

#. 通过grub配置保留1G的hugepage。例如，保留8个1G大小的页面::

      default_hugepagesz=1G hugepagesz=1G hugepages=8

#. 隔离将用于DPDK的CPU core.如::

      isolcpus=2,3,4,5,6,7,8

#. 如果要使用VFIO，请使用以下附加的grub参数::

      iommu=pt intel_iommu=on


运行DPDK前的配置
----------------

1. 构建目标文件，预留hugepage。参阅前面 :ref:`linux_gsg_hugepages` 描述。

   以下命令为具体过程:

   .. code-block:: console

      # Build DPDK target.
      cd dpdk_folder
      make install T=x86_64-native-linuxapp-gcc -j

      # Get the hugepage size.
      awk '/Hugepagesize/ {print $2}' /proc/meminfo

      # Get the total huge page numbers.
      awk '/HugePages_Total/ {print $2} ' /proc/meminfo

      # Unmount the hugepages.
      umount `awk '/hugetlbfs/ {print $2}' /proc/mounts`

      # Create the hugepage mount folder.
      mkdir -p /mnt/huge

      # Mount to the specific folder.
      mount -t hugetlbfs nodev /mnt/huge

2. 使用命令 ``cpu_layout`` 来检查CPU布局:

   .. code-block:: console

      cd dpdk_folder

      usertools/cpu_layout.py

   或者运行 ``lscpu`` 检查每个插槽上的core。

3. 检查NIC ID和插槽ID:

   .. code-block:: console

      # 列出所有的网卡的PCI地址及设备ID.
      lspci -nn | grep Eth

   例如，假设你的输入如下::

      82:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
      82:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
      85:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
      85:00.1 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]

   检测PCI设备相关联的NUMA节点:

   .. code-block:: console

      cat /sys/bus/pci/devices/0000\:xx\:00.x/numa_node

   通常的，``0x:00.x`` 表示在插槽0，而 ``8x:00.x`` 表示在插槽1。
   **Note**: 为了说去最佳性能，请保证core和NIC位于同一插槽中。
   在上面的例子中 ``85:00.0`` 在插槽1，因此必须被插槽1上的core使用才能获得最佳性能。

4. 将测试端口绑定到DPDK兼容的驱动程序，如igb_uio。例如，将两个端口绑定到兼容DPDK的驱动程序并检查状态:

   .. code-block:: console


      # 绑定端口 82:00.0 和 85:00.0 到DPDK驱动
      ./dpdk_folder/usertools/dpdk-devbind.py -b igb_uio 82:00.0 85:00.0

      # 检查端口驱动状态
      ./dpdk_folder/usertools/dpdk-devbind.py --status

   运行 ``dpdk-devbind.py --help`` 以获取更多信息。


有关DPDK设置和Linux内核需求的更多信息，请参阅 :ref:`linux_gsg_compiling_dpdk` 。

网卡最佳性能实践举例
--------------------

以下是运行DPDK ``l3fwd`` 例程并获取最佳性能的例子。使用 Intel 服务平台和Intel XL710 NICs。
具体的40G NIC配置请参阅i40e NIC指南。

本例场景是通过两个Intel XL710 40GbE端口获取最优性能。请参阅 :numref:`figure_intel_perf_test_setup` 用于性能测试设置。

.. _figure_intel_perf_test_setup:

.. figure:: img/intel_perf_test_setup.*

   性能测试搭建


1. 将两个Intel XL710 NIC添加到平台，并使用每个卡一个端口来获得最佳性能。使用两个NIC的原因是克服PCIe Gen3的限制，因为它不能提供80G带宽。
   对于两个40G端口，但两个不同的PCIe Gen3 x8插槽可以。
   请参考上面的示例NIC输出，然后我们可以选择 ``82:00.0`` 及 ``85:00.0`` 作为测试端口::

      82:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]
      85:00.0 Ethernet [0200]: Intel XL710 for 40GbE QSFP+ [8086:1583]

2. 将端口连接到打流机，对于高速测试，最好有专用的打流设备。

3. 检测PCI设备的numa节点，并获取该插槽id上的core。
   在本例中， ``82:00.0`` 和 ``85:00.0`` 都在插槽1上，插槽1上的core id为18-35 和 54-71。
   Note: 不要在同一个core上使用两个逻辑核(e.g core18 有两个逻辑核core18 and core54)，而是使用来自不同core的两个逻辑核。

4. 将这两个端口绑定到igb_uio。

5. 对于XL710 40G 端口，我们需要至少两个队列来实现最佳性能，因此每个端口需要两个队列，每个队列将需要专用的CPU内核来接收/发送数据包。

6. 使用DPDK示例程序 ``l3fwd`` 做性能测试，两个端口进行双向转发，使用默认的lpm模式编译 ``l3fwd sample``。

7. 运行l3fwd的命令如下所示::

      ./l3fwd -c 0x3c0000 -n 4 -w 82:00.0 -w 85:00.0 \
              -- -p 0x3 --config '(0,0,18),(0,1,19),(1,0,20),(1,1,21)'

   命令表示应用程序使用（core18，port0，队列0），（core19，port0，队列1）, （core20，port1，队列0），（core18，port1，队列1）。

8. 配置打流机用于发包

   * 创建流

   * 设置报文类型为Ethernet II type to 0x0800。

