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

编译和运行简单应用程序
======================

本章介绍如何在DPDK环境下编译和运行应用程序。还指出应用程序的存储位置。

.. note::

    此过程的部分操作也可以使用脚本来完成。参考 :ref:`linux_setup_script` 章节描述。

编译一个简单应用程序
--------------------

一个DPDK目标环境创建完成时(如 ``x86_64-native-linuxapp-gcc``)，它包含编译一个应用程序所需要的全部库和头文件。

当在Linux* 交叉环境中编译应用程序时，以下变量需要预先导出：

* ``RTE_SDK`` - 指向DPDK安装目录。

* ``RTE_TARGET`` - 指向DPDK目标环境目录。

以下是创建 ``helloworld`` 应用程序实例，该实例将在DPDK Linux环境中运行。
这个实例可以在目录 ``${RTE_SDK}/examples`` 找到。

该目录包含 ``main.c`` 文件。该文件与DPDK目标环境中的库结合使用时，调用各种函数初始化DPDK环境，
然后，为每个要使用的core启动一个入口点（调度应用程序）。
默认情况下，二进制文件存储在build目录中。

.. code-block:: console

    cd examples/helloworld/
    export RTE_SDK=$HOME/DPDK
    export RTE_TARGET=x86_64-native-linuxapp-gcc

    make
        CC main.o
        LD helloworld
        INSTALL-APP helloworld
        INSTALL-MAP helloworld.map

    ls build/app
        helloworld helloworld.map

.. note::

    在上面的例子中， ``helloworld`` 是在DPDK的目录结构下的。
    当然，也可以将其放在DPDK目录之外，以保证DPDK的结构不变。
    下面的例子， ``helloworld`` 应用程序被复制到一个新的目录下。

    .. code-block:: console

       export RTE_SDK=/home/user/DPDK
       cp -r $(RTE_SDK)/examples/helloworld my_rte_app
       cd my_rte_app/
       export RTE_TARGET=x86_64-native-linuxapp-gcc

       make
         CC main.o
         LD helloworld
         INSTALL-APP helloworld
         INSTALL-MAP helloworld.map

运行一个简单的应用程序
----------------------

.. warning::

    UIO驱动和hugepage必须在程序运行前设置好。

.. warning::

    应用程序使用的任何端口，必须绑定到合适的内核驱动模块上，如章节 :ref:`linux_gsg_binding_kernel` 描述的那样。

应用程序与DPDK目标环境的环境抽象层（EAL）库相关联，该库提供了所有DPDK程序通用的一些选项。

以下是EAL提供的一些选项列表：

.. code-block:: console

    ./rte-app -c COREMASK [-n NUM] [-b <domain:bus:devid.func>] \
              [--socket-mem=MB,...] [-m MB] [-r NUM] [-v] [--file-prefix] \
	      [--proc-type <primary|secondary|auto>] [-- xen-dom0]

选项描述如下：

* ``-c COREMASK``:
  要运行的内核的十六进制掩码。注意，平台之间编号可能不同，需要事先确定。

* ``-n NUM``:
  每个处理器插槽的内存通道数目。

* ``-b <domain:bus:devid.func>``:
  端口黑名单，避免EAL使用指定的PCI设备。

* ``--use-device``:
  仅使用指定的以太网设备。使用逗号分隔 ``[domain:]bus:devid.func`` 值，不能与 ``-b`` 选项一起使用。

* ``--socket-mem``:
  从特定插槽上的hugepage分配内存。

* ``-m MB``:
  内存从hugepage分配，不管处理器插槽。建议使用 ``--socket-mem`` 而非这个选项。

* ``-r NUM``:
  内存数量。

* ``-v``:
  显示启动时的版本信息。

* ``--huge-dir``:
  挂载hugetlbfs的目录。

* ``--file-prefix``:
  用于hugepage文件名的前缀文本。

* ``--proc-type``:
  程序实例的类型。

* ``--xen-dom0``:
  支持在Xen Domain0上运行，但不具有hugetlbfs的程序。

* ``--vmware-tsc-map``:
  使用VMware TSC 映射而不是本地RDTSC。

* ``--base-virtaddr``:
  指定基本虚拟地址。

* ``--vfio-intr``:
  指定要由VFIO使用的中断类型。(如果不支持VFIO，则配置无效)。

其中 ``-c`` 是强制性的，其他为可选配置。

将DPDK应用程序二进制文件拷贝到目标设备，按照如下命令运行(我们假设每个平台处理器有4个内存通道，并且存在core0～3用于运行程序)::

    ./helloworld -c f -n 4

.. note::

    选项 ``--proc-type`` 和 ``--file-prefix`` 用于运行多个DPDK进程。请参阅 "多应用程序实例" 章节及 *DPDK
    编程指南* 获取更多细节。

应用程序使用的逻辑Core
~~~~~~~~~~~~~~~~~~~~~~

对于DPDK应用程序，coremask参数始终是必须的。掩码的每个位对应于Linux提供的逻辑core ID。
由于这些逻辑core的编号，以及他们在NUMA插槽上的映射可能因平台而异，因此建议在选择每种情况下使用的coremaks时，都要考虑每个平台的core布局。

在DPDK程序初始化EAL层时，将显示要使用的逻辑core及其插槽位置。可以通过读取 ``/proc/cpuinfo`` 文件来获取系统上所有core的信息。例如执行 cat ``/proc/cpuinfo``。
列出来的physical id 属性表示其所属的CPU插槽。当使用了其他处理器来了解逻辑core到插槽的映射时，这些信息很有用。

.. note::

    可以使用另一个Linux工具 ``lstopo`` 来获取逻辑core布局的图形化信息。在Fedora Linux上, 可以通过如下命令安装并运行工具::

        sudo yum install hwloc
        ./lstopo

.. warning::

    逻辑core在不同的电路板上可能不同，在应用程序使用coremaks时需要先确定。

应用程序使用的Hugepage内存
~~~~~~~~~~~~~~~~~~~~~~~~~~

当运行应用程序时，建议使用的内存与hugepage预留的内存一致。如果运行时没有 ``-m`` 或 ``--socket-mem`` 参数传入，这由DPDK应用程序在启动时自动完成。

如果通过显示传入 ``-m`` 或 ``--socket-mem`` 值，但是请求的内存超过了该值，应用程序将执行失败。
但是，如果用户请求的内存小于预留的hugepage-memory，应用程序也会失败，特别是当使用了 ``-m`` 选项的时候。
因为，假设系统在插槽0和插槽1上有1024个预留的2MB页面，如果用户请求128 MB的内存，可能存在64个页不符合要求的情况:

*   内核只能在插槽1中将hugepage-memory提供给应用程序。在这种情况下，如果应用程序尝试创建一个插槽0中的对象，例如ring或者内存池，那么将执行失败
    为了避免这个问题，建议使用 ``--socket-mem`` 选项替代 ``-m`` 选项。

*   这些页面可能位于物理内存中的任意位置，尽管DPDK EAL将尝试在连续的内存块中分配内存，但是页面可能是不连续的。在这种情况下，应用程序无法分配大内存。

使用socket-mem选项可以为特定的插槽请求特定大小的内存。通过提供 ``--socket-mem`` 标志和每个插槽需要的内存数量来实现的，如 ``--socket-mem=0,512`` 用于在插槽1上预留512MB内存。
类似的，在4插槽系统上，如果只能在插槽0和2上分配1GB内存，则可以使用参数``--socket-mem=1024,0,1024`` 来实现。
如果DPDK无法在每个插槽上分配足够的内存，则EAL初始化失败。

其他示例程序
------------

其他的一些示例程序包含在${RTE_SDK}/examples 目录下。这些示例程序可以使用本手册前面部分所述的方法进行构建运行。另外，请参阅 *DPDK示例程序用户指南* 了解应用程序的描述、编译和执行的具体说明以及代码解释。

附加的测试程序
--------------

此外，还有两个在创建库时构建的应用程序。这些源文件位于 DPDK/app目录下，称为test和testpmd程序。创建库之后，可以在build目录中找到。

*   test程序为DPDK中的各种功能提供具体的测试。

*   testpmd程序提供了许多不同的数据包吞吐测试，例如，在Intel® 82599 10 Gigabit Ethernet Controller中如何使用Flow Director。

