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

.. _linux_setup_script:

使用脚本快速构建
================

usertools目录中的dpdk-setup.sh脚本，向用户提供了快速执行如下任务功能:

*   构建DPDK库

*   加载/卸载DPDK IGB_UIO内核模块

*   加载/卸载VFIO内核模块

*   加载/卸载DPDK KNI内核模块

*   创建/删除NUMA 或 non-NUMA平台的hugepages

*   查看网络端口状态和预留给DPDK应用程序使用的端口

*   设置非root用户使用VFIO的权限

*   运行test和testpmd应用程序

*   查看meminfo中的hugepages

*   列出在 ``/mnt/huge`` 中的hugepages 

*   删除内置的DPDK库

对于其中一个EAL目标，一旦完成了这些步骤，用户就可以编译自己的在EAL库中链接的应用程序来创建DPDK映像。

脚本组织
--------

dpdk-setup.sh脚本在逻辑上组织成用户按顺序执行的一系列步骤。每个步骤都提供了许多选项来指导用户完成所需的任务。以下是每个步骤的简单介绍：

**Step 1: Build DPDK Libraries**

最开始，用户必须指定tagert的类型以便编译正确的库。

如本入门指南前面的章节描述，用户必须在此之前就安装好所有的库、模块、更新和编译器。

**Step 2: Setup Environment**

用户需要配置Linux* 环境以支持DPDK应用程序的运行。
可以为NUMA 或non-NUMA系统分配Hugepages。任何原来已经存在的hugepages将被删除。
也可以在此步骤中插入所需的DPDK内核模块，并且可以将网络端口绑定到此模块供DPDK使用。

**Step 3: Run an Application**

一旦执行了其他步骤，用户就可以运行test程序。该程序允许用户为DPDK运行一系列功能测试。也可以运行支持数据包接收和发送的testpmd程序。

**Step 4: Examining the System**

此步骤提供了一些用于检查Hugepage映射状态的工具。

**Step 5: System Cleanup**

最后一步具有将系统恢复到原始状态的选项。

Use Cases
---------

以下是使用dpdk-setup.sh的示例。脚本应该使用source命令运行。脚本中的某些选项在继续操作之前提示用户需要进一步的数据输入。

.. warning::

    必须与root全选运行dpdk-setup.sh。

.. code-block:: console

    source usertools/dpdk-setup.sh

    ------------------------------------------------------------------------

    RTE_SDK exported as /home/user/rte

    ------------------------------------------------------------------------

    Step 1: Select the DPDK environment to build

    ------------------------------------------------------------------------

    [1] i686-native-linuxapp-gcc

    [2] i686-native-linuxapp-icc

    [3] ppc_64-power8-linuxapp-gcc

    [4] x86_64-native-bsdapp-clang

    [5] x86_64-native-bsdapp-gcc

    [6] x86_64-native-linuxapp-clang

    [7] x86_64-native-linuxapp-gcc

    [8] x86_64-native-linuxapp-icc

    ------------------------------------------------------------------------

    Step 2: Setup linuxapp environment

    ------------------------------------------------------------------------

    [11] Insert IGB UIO module

    [12] Insert VFIO module

    [13] Insert KNI module

    [14] Setup hugepage mappings for non-NUMA systems

    [15] Setup hugepage mappings for NUMA systems

    [16] Display current Ethernet device settings

    [17] Bind Ethernet device to IGB UIO module

    [18] Bind Ethernet device to VFIO module

    [19] Setup VFIO permissions

    ------------------------------------------------------------------------

    Step 3: Run test application for linuxapp environment

    ------------------------------------------------------------------------

    [20] Run test application ($RTE_TARGET/app/test)

    [21] Run testpmd application in interactive mode ($RTE_TARGET/app/testpmd)

    ------------------------------------------------------------------------

    Step 4: Other tools

    ------------------------------------------------------------------------

    [22] List hugepage info from /proc/meminfo

    ------------------------------------------------------------------------

    Step 5: Uninstall and system cleanup

    ------------------------------------------------------------------------

    [23] Uninstall all targets

    [24] Unbind NICs from IGB UIO driver

    [25] Remove IGB UIO module

    [26] Remove VFIO module

    [27] Remove KNI module

    [28] Remove hugepage mappings

    [29] Exit Script

Option:

以下选项演示了 “x86_64-native-linuxapp-gcc“ DPDK库的创建。

.. code-block:: console

    Option: 9

    ================== Installing x86_64-native-linuxapp-gcc

    Configuration done
    == Build lib
    ...
    Build complete
    RTE_TARGET exported as x86_64-native-linuxapp-gcc

以下选项用于启动DPDK UIO驱动程序。

.. code-block:: console

    Option: 25

    Unloading any existing DPDK UIO module
    Loading DPDK UIO module

以下选项演示了在NUMA系统中创建hugepage。为每个node分配1024个2MB的页。
应用程序应该使用 -m 4096 来启动，以便访问这两个内存区域。(如果没有 -m 选项，则自动完成)。

.. note::

    如果显示提示以删除临时文件，请输入'y'。

.. code-block:: console

    Option: 15

    Removing currently reserved hugepages
    mounting /mnt/huge and removing directory
    Input the number of 2MB pages for each node
    Example: to have 128MB of hugepages available per node,
    enter '64' to reserve 64 * 2MB pages on each node
    Number of pages for node0: 1024
    Number of pages for node1: 1024
    Reserving hugepages
    Creating /mnt/huge and mounting as hugetlbfs

以下操作说明了启动测试应用程序以在单个core上运行

.. code-block:: console

    Option: 20

    Enter hex bitmask of cores to execute test app on
    Example: to execute app on cores 0 to 7, enter 0xff
    bitmask: 0x01
    Launching app
    EAL: coremask set to 1
    EAL: Detected lcore 0 on socket 0
    ...
    EAL: Master core 0 is ready (tid=1b2ad720)
    RTE>>

应用程序
--------

一旦用户运行和dpdk-setup.sh脚本，构建了目标程序并且设置了hugepages，用户就可以继续构建和运行自己的应用程序或者源码中提供的示例。

/examples 目录中提供的示例程序为了解DPDK提供了很好的起点。
以下命令显示了helloworld应用程序的构建和运行方式。
按照4.2.1节，"应用程序使用的逻辑Core"描述，当选择用于应用程序的coremask时，需要确定平台的逻辑core的布局。

.. code-block:: console

    cd helloworld/
    make
      CC main.o
      LD helloworld
      INSTALL-APP helloworld
      INSTALL-MAP helloworld.map

    sudo ./build/app/helloworld -c 0xf -n 3
    [sudo] password for rte:

    EAL: coremask set to f
    EAL: Detected lcore 0 as core 0 on socket 0
    EAL: Detected lcore 1 as core 0 on socket 1
    EAL: Detected lcore 2 as core 1 on socket 0
    EAL: Detected lcore 3 as core 1 on socket 1
    EAL: Setting up hugepage memory...
    EAL: Ask a virtual area of 0x200000 bytes
    EAL: Virtual area found at 0x7f0add800000 (size = 0x200000)
    EAL: Ask a virtual area of 0x3d400000 bytes
    EAL: Virtual area found at 0x7f0aa0200000 (size = 0x3d400000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9fc00000 (size = 0x400000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9f600000 (size = 0x400000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9f000000 (size = 0x400000)
    EAL: Ask a virtual area of 0x800000 bytes
    EAL: Virtual area found at 0x7f0a9e600000 (size = 0x800000)
    EAL: Ask a virtual area of 0x800000 bytes
    EAL: Virtual area found at 0x7f0a9dc00000 (size = 0x800000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9d600000 (size = 0x400000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9d000000 (size = 0x400000)
    EAL: Ask a virtual area of 0x400000 bytes
    EAL: Virtual area found at 0x7f0a9ca00000 (size = 0x400000)
    EAL: Ask a virtual area of 0x200000 bytes
    EAL: Virtual area found at 0x7f0a9c600000 (size = 0x200000)
    EAL: Ask a virtual area of 0x200000 bytes
    EAL: Virtual area found at 0x7f0a9c200000 (size = 0x200000)
    EAL: Ask a virtual area of 0x3fc00000 bytes
    EAL: Virtual area found at 0x7f0a5c400000 (size = 0x3fc00000)
    EAL: Ask a virtual area of 0x200000 bytes
    EAL: Virtual area found at 0x7f0a5c000000 (size = 0x200000)
    EAL: Requesting 1024 pages of size 2MB from socket 0
    EAL: Requesting 1024 pages of size 2MB from socket 1
    EAL: Master core 0 is ready (tid=de25b700)
    EAL: Core 1 is ready (tid=5b7fe700)
    EAL: Core 3 is ready (tid=5a7fc700)
    EAL: Core 2 is ready (tid=5affd700)
    hello from core 1
    hello from core 2
    hello from core 3
    hello from core 0

