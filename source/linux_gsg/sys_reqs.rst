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

系统要求
========

本章描述了编译DPDK所需的软件包。

.. note::

    假如在Intel公司的89xx通信芯片组平台上使用DPDK，请参阅文档 *Intel® Communications Chipset 89xx Series Software for Linux Getting Started Guide*。

X86 上预先设置 BIOS
-------------------

对大多数平台，使用基本DPDK功能无需对BIOS进行特殊设置。然而，对于HPET定时器和电源管理功能，以及为了获得40G网卡上小包处理的高性能，则可能需要更改BIOS设置。可以参阅章节 :ref:`Enabling Additional Functionality <Enabling_Additional_Functionality>`
以获取更为详细的信息。

编译DPDK
--------

**工具集：**

.. note::

    以下说明在Fedora 18上通过了测试。其他系统所需要的安装命令和软件包可能有所不同。有关其他Linux发行版本和测试版本的详细信息，请参阅DPDK发布说明。

*   GNU ``make``.

*   coreutils: ``cmp``, ``sed``, ``grep``, ``arch`` 等.

*   gcc: 4.9以上的版本适用于所有的平台。
    在某些发布版本中，启用了一些特定的编译器标志和链接标志（例如``-fstack-protector``）。请参阅文档的发布版本和 ``gcc -dumpspecs``.

*   libc 头文件，通常打包成 ``gcc-multilib`` (``glibc-devel.i686`` / ``libc6-dev-i386``;
    ``glibc-devel.x86_64`` / ``libc6-dev`` 用于Intel 64位架构编译;
    ``glibc-devel.ppc64`` 用于IBM 64位架构编译;)

*   构建Linux内核模块所需要的头文件和源文件。(kernel - devel.x86_64; kernel - devel.ppc64)

*   在64位系统上编译32位软件包额外需要的软件为：

    * glibc.i686, libgcc.i686, libstdc++.i686 及 glibc-devel.i686， 适用于Intel的i686/x86_64;

    * glibc.ppc64, libgcc.ppc64, libstdc++.ppc64 及 glibc-devel.ppc64 适用于 IBM ppc_64;

    .. note::

       x86_x32 ABI目前仅在Ubuntu 13.10及以上版本或者Debian最近的发行版本上支持。编译器必须是gcc 4.9+版本。

*   Python, 2.7+ or 3.2+版本, 用于运行DPDK软件包中的各种帮助脚本。


**可选工具：**

*   Intel® C++ Compiler (icc). 安装icc可能需要额外的库，请参阅编译器安装目录下的icc安装指南。

*   IBM® Advance ToolChain for Powerlinux. 这是一组开源开发工具和运行库。允许用户在Linux上使用IBM最新POWER硬件的优势。具体安装请参阅IBM的官方安装文档。

*   libpcap 头文件和库 (libpcap-devel) ，用于编译和使用基于libcap的轮询模式驱动程序。默认情况下，该驱动程序被禁用，可以通过在构建时修改配置文件 ``CONFIG_RTE_LIBRTE_PMD_PCAP=y`` 来开启。

*   需要使用libarchive 头文件和库来进行某些使用tar获取资源的单元测试。


运行DPDK应用程序
----------------

要运行DPDK应用程序，需要在目标机器上进行某些定制。

系统软件
~~~~~~~~

**需求：**

*   Kernel version >= 2.6.34

    当前内核版本可以通过命令查看::
	
        uname -r

*   glibc >= 2.7 (方便使用cpuset相关特性)

    版本信息通命令 ``ldd --version`` 查看。

*   Kernel configuration

    在 Fedora OS 及其他常见的发行版本中，如 Ubuntu 或 Red Hat Enterprise Linux，供应商提供的配置可以运行大多数的DPDK应用程序。
	对于其他内核构件，应为DPDK开启的选项包括：

    *   UIO 支持

    *   HUGETLBFS 支持

    *   PROC_PAGE_MONITOR  支持

    *   如果需要HPET支持，还应开启 HPET and HPET_MMAP 配置选项。有关信息参考 :ref:`High Precision Event Timer (HPET) Functionality <High_Precision_Event_Timer>` 章节获取更多信息。

.. _linux_gsg_hugepages:

在 Linux 环境中使用 Hugepages 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

用于数据包缓冲区的大型内存池分配需要 Hugepages 支持（如上节所述，必须在运行的内核中开启 HUGETLBFS 选项）。通过使用大页分配，程序需要更少的页面，性能增加，
因为较少的TLB减少了将虚拟页面地址翻译成物理页面地址所需的时间。如果没有大页，标准大小4k的页面会导致频繁的TLB miss，性能下降。

预留 Hugepages 给 DPDK 使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^

大页分配应该在系统引导时或者启动后尽快完成，以避免物理内存碎片化。要在引导时预留大页，需要给Linux内核命令行传递一个参数。

对于2MB大小的页面，只需要将hugepages选项传递给内核。如，预留1024个2MB大小的page，使用::

    hugepages=1024

对于其他大小的hugepage，例如1G的页，大小必须同时指定。例如，要预留4个1G大小的页面给程序，需要传递以下选项给内核::

    default_hugepagesz=1G hugepagesz=1G hugepages=4

.. note::

    CPU支持的hugepage大小可以从Intel架构上的CPU标志位确定。如果存在pse，则支持2M个hugepages，如果page1gb存在，则支持1G的hugepages。
	在IBM Power架构中，支持的hugepage大小为16MB和16GB。

.. note::

    对于64位程序，如果平台支持，建议使用1GB的hugepages。

在双插槽NUMA的系统上，在启动时预留的hugepage数目通常在两个插槽之间评分（假设两个插槽上都有足够的内存）。

有关这些和其他内核选项的信息，请参阅Linux源代码目录中/kernel-parameter.txt文件。

**特例：**

对于2MB页面，还可以在系统启动之后再分配，通过向 ``/sys/devices/`` 目录下的nr_hugepages文件写入hugepage数目来实现。
对于单节点系统，使用的命令如下（假设需要1024个页）::

    echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

在NUMA设备中，分配应该明确指定在哪个节点上::

    echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
    echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

.. note::

    对于1G页面，系统启动之后无法预留页面内存。

DPDK 使用 Hugepages
^^^^^^^^^^^^^^^^^^^

一旦预留了hugepage内存，为了使内存可用于DPDK，请执行以下步骤::

    mkdir /mnt/huge
    mount -t hugetlbfs nodev /mnt/huge

通过将一下命令添加到 ``/etc/fstab`` 文件中，安装点可以在重启时永久保存::

    nodev /mnt/huge hugetlbfs defaults 0 0

对于1GB内存，页面大小必须在安装选项中指定::

    nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0

Linux 环境中 Xen Domain0 支持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现有的内存管理实现是基于Linux内核的hugepage机制。在Xen虚拟机管理程序中，对于DomainU客户端的支持意味着DPDK程序与客户一样正常工作。

但是，Domain0不支持hugepages。为了解决这个限制，添加了一个新的内核模块rte_dom0_mm用于方便内存的分配和映射，通过 **IOCTL** (分配) 和 **MMAP** (映射).

DPDK 中使能 Xen Dom0模式
^^^^^^^^^^^^^^^^^^^^^^^^

默认情况下，DPDK构建时禁止使用 Xen Dom0 模式。要支持 Xen Dom0，CONFIG_RTE_LIBRTE_XEN_DOM0设置应该改为 “y”，编译时弃用该模式。

此外，为了允许接收错误套接字ID，CONFIG_RTE_EAL_ALLOW_INV_SOCKET_ID也必须设置为 “y”。

加载 DPDK rte_dom0_mm 模块
^^^^^^^^^^^^^^^^^^^^^^^^^^

要在Xen Dom0下运行任何DPDK应用程序，必须使用rsv_memsize选项将 ``rte_dom0_mm`` 模块加载到运行的内核中。该模块位于DPDK目标目录的kmod子目录中。应该使用insmod命令加载此模块，如下所示::

    sudo insmod kmod/rte_dom0_mm.ko rsv_memsize=X

X的值不能大于4096(MB).

配置内存用于DPDK使用
^^^^^^^^^^^^^^^^^^^^

在加载rte_dom0_mm.ko内核模块之后，用户必须配置DPDK使用的内存大小。这也是通过将内存大小写入到目录 ``/sys/devices/`` 下的文件memsize中来实现的。
使用以下命令(假设需要2048MB)::

    echo 2048 > /sys/kernel/mm/dom0-mm/memsize-mB/memsize

用户还可以使用下面命令检查已经使用了多少内存::

    cat /sys/kernel/mm/dom0-mm/memsize-mB/memsize_rsvd

Xen Domain0 不支持NUMA配置，因此 ``--socket-mem`` 命令选项对Xen Domain0无效。

.. note::

    memsize的值不能大于rsv_memsize。

在 Xen Domain0上运行DPDK程序
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

要在Xen Domain0上运行DPDK程序，需要一个额外的命令行选项 ``--xen-dom0``。
