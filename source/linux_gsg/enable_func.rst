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

.. _Enabling_Additional_Functionality:

启用附加功能
============

.. _High_Precision_Event_Timer:

高精度事件定时器 (HPET) 功能
----------------------------

BIOS 支持
~~~~~~~~~

要使用HPET功能时，必须先在平台BIOS上开启高精度定时器。否则，默认情况下使用时间戳计数器 (TSC) 。
通常情况下，起机时按 F2 可以访问BIOS。然后用户可以导航到HPET选项。
在Crystal Forest平台BIOS上，路径为：**Advanced -> PCH-IO Configuration -> High Precision Timer ->** (如果需要，将Disabled 改为 Enabled )。

在已经起机的系统上，可以使用以下命令来检查HPET是否启用 ::

   grep hpet /proc/timer_list

如果没有条目，则必须在BIOS中启用HPET，镔铁重新启动系统。

Linux 内核支持
~~~~~~~~~~~~~~

DPDK通过将定时器计数器映射到进程地址空间来使用平台的HPET功能，因此，要求开启 ``HPET_MMAP`` 系统内核配置选项。

.. warning::

    在Fedora或者其他常见的Linux发行版本（如Ubuntu）中，默认不会启用 ``HPET_MMAP`` 选项。
    要重新编译启动此选项的内核，请参阅发行版本的相关说明。

DPDK 中使能 HPET
~~~~~~~~~~~~~~~~

默认情况下，DPDK配置文件中是禁用HPET功能的。要使用HPET，需要将 ``CONFIG_RTE_LIBEAL_USE_HPET`` 设置为 ``y`` 来开启编译。

对于那些使用 ``rte_get_hpet_cycles()`` 及 ``rte_get_hpet_hz()`` API接口的应用程序，
并且选择了HPET作为rte_timer库的默认时钟源，需要在初始化时调用 ``rte_eal_hpet_init()`` API。
这个API调用将保证HPET可用，如果HPET不可用(例如，内核没有开启 ``HPET_MMAP`` 使能)，则向程序返回一个错误值。
如果HPET在运行时不可用，应用程序可以方便的采取其他措施。

.. note::

    对于那些仅需要普通定时器API，而不是HPET定时器的应用程序，建议使用 ``rte_get_timer_cycles()`` 和 ``rte_get_timer_hz()`` API调用，而不是HPET API。
    这些通用的API兼容TSC和HPET时钟源，具体时钟源则取决于应用程序是否调用 ``rte_eal_hpet_init()``初始化，以及运行时系统上可用的时钟。

没有Root权限情况下运行DPDK应用程序
----------------------------------

虽然DPDK应用程序直接使用了网络端口及其他硬件资源，但通过许多小的权限调整，可以允许除root权限之外的用户运行这些应用程序。
为了保证普通的Linux用户也可以运行这些程序，需要调整如下Linux文件系统权限:

*   所有用于hugepage挂载点的文件和目录，如 ``/mnt/huge``

*   ``/dev`` 中的UIO设备文件，如 ``/dev/uio0``, ``/dev/uio1`` 等

*   UIO系统配置和源文件，如 ``uio0``::

       /sys/class/uio/uio0/device/config
       /sys/class/uio/uio0/device/resource*

*   如果要使用HPET，那么 ``/dev/hpet`` 目录也要修改

.. note::

    在某些Linux 安装中， ``/dev/hugepages`` 也是默认创建hugepage挂载点的文件。

电源管理和节能功能
------------------

如果要使用DPDK的电源管理功能，必须在平台BIOS中启用增强的Intel SpeedStep® Technology。否则，sys文件夹下 ``/sys/devices/system/cpu/cpu0/cpufreq`` 将不存在，不能使用基于CPU频率的电源管理。请参阅相关的BIOS文档以确定如何访问这些设置。

例如，在某些Intel参考平台上，开启Enhanced Intel SpeedStep® Technology 的路径为::

   Advanced
     -> Processor Configuration
     -> Enhanced Intel SpeedStep® Tech

此外，C3 和 C6 也应该使能以支持电源管理。C3 和 C6 的配置路径为::

   Advanced
     -> Processor Configuration
     -> Processor C3 Advanced
     -> Processor Configuration
     -> Processor C6

使用 Linux Core 隔离来减少上下文切换
------------------------------------

虽然DPDK应用程序使用的线程固定在系统的逻辑核上，但Linux调度程序也可以在这些核上运行其他任务。
为了防止在这些核上运行额外的工作负载，可以使用 ``isolcpus`` Linux 内核参数来将其与通用的Linux调度程序隔离开来。

例如，如果DPDK应用程序要在逻辑核2，4，6上运行，应将以下内容添加到内核参数表中:

.. code-block:: console

    isolcpus=2,4,6

加载 DPDK KNI 内核模块
----------------------

要运行DPDK Kernel NIC Interface (KNI) 应用程序，需要将一个额外的内核模块(kni模块)加载到内核中。
该模块位于DPDK目录kmod子目录中。与 ``igb_uio`` 模块加载类似，(假设当前目录就是DPDK目录):

.. code-block:: console

   insmod kmod/rte_kni.ko

.. note::

   相关的详细信息，可以参阅 "Kernel NIC Interface Sample Application" 章节和 *DPDK 示例程序用户指南* 。

Linux IOMMU Pass-Through使用Intel® VT-d运行DPDK 
-----------------------------------------------

要在Linux内核中启用Intel® VT-d，必须配置一系列内核选项，包括：

*   ``IOMMU_SUPPORT``

*   ``IOMMU_API``

*   ``INTEL_IOMMU``

另外，要使用Intel® VT-d运行DPDK，使用 ``igb_uio`` 驱动时必须携带 ``iommu=pt`` 参数。
这使得主机可以直接通过DMA重映射查找。
另外，如果内核中没有设置 ``INTEL_IOMMU_DEFAULT_ON`` 参数，那么也必须使用 ``intel_iommu=on`` 参数。这可以确保 Intel IOMMU 被正确初始化。

请注意，对于``igb_uio`` 驱动程序，使用 ``iommu = pt`` 是必须de ，``vfio-pci`` 驱动程序实际上可以同时使用 ``iommu = pt`` 和 ``iommu = on`` 。

40G NIC上的小包处理高性能
-------------------------

由于在最新版本中可能提供用于性能提升的固件修复，因此最好进行固件更新以获取更高的性能。
请和 Intel's Network Division 工程师联系以进行固件更新。
用户可以参考DPDK版本发行说明，以使用 i40e 驱动程序识别NIC的已验证固件版本。

使用16B大小的RX描述符
~~~~~~~~~~~~~~~~~~~~~

由于 i40e PMD 支持16B和32B的RX描述符，而16B大小的描述符可以帮助小型数据包提供性能，因此，配置文件中 ``CONFIG_RTE_LIBRTE_I40E_16BYTE_RX_DESC`` 更改为使用16B大小的描述符。

高性能和每数据包延迟权衡
~~~~~~~~~~~~~~~~~~~~~~~~

由于硬件设计，每个数据包描述符回写都需要NIC内部的中断信号。中断的最小间隔可以在编译时通过配置文件中的 ``CONFIG_RTE_LIBRTE_I40E_ITR_INTERVAL`` 指定。
虽然有默认配置，但是该配置可以由用户自行调整，这取决于用户所关心的内容，整体性能或者每数据包延迟。

