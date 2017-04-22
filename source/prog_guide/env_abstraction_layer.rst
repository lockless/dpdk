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

.. _Environment_Abstraction_Layer:

环境适配层EAL
=============

环境抽象层为底层资源如硬件和内存空间的访问提供了接口。
这些通用的接口为APP和库隐藏了不同环境的特殊性。
EAL负责初始化及分配资源（内存、PCI设备、定时器、控制台等等）。

EAL提供的典型服务有：

*   DPDK的加载和启动：DPDK和指定的程序链接成一个独立的进程，并以某种方式加载

*   CPU亲和性和分配处理：DPDK提供机制将执行单元绑定到特定的核上，就像创建一个执行程序一样。

*   系统内存分配：EAL实现了不同区域内存的分配，例如为设备接口提供了物理内存。

*   PCI地址抽象：EAL提供了对PCI地址空间的访问接口

*   跟踪调试功能：日志信息，堆栈打印、异常挂起等等。

*   公用功能：提供了标准libc不提供的自旋锁、原子计数器等。

*   CPU特征辨识：用于决定CPU运行时的一些特殊功能，决定当前CPU支持的特性，以便编译对应的二进制文件。

*   中断处理：提供接口用于向中断注册/解注册回掉函数。

*   告警功能：提供接口用于设置/取消指定时间环境下运行的毁掉函数。

Linux环境下的EAL
----------------

在Linux用户空间环境，DPDK APP通过pthread库作为一个用户态程序运行。
设备的PCI信息和地址空间通过 /sys 内核接口及内核模块如uio_pci_generic或igb_uio来发现获取的。
linux内核文档中UIO描述，设备的UIO信息是在程序中用mmap重新映射的。

EAL通过对hugetlb使用mmap接口来实现物理内存的分配。这部分内存暴露给DPDK服务层，如 :ref:`Mempool Library <Mempool_Library>`。

据此，DPDK服务层可以完成初始化，接着通过设置线程亲和性调用，每个执行单元将会分配给特定的逻辑核，以一个user-level等级的线程来运行。

定时器是通过CPU的时间戳计数器TSC或者通过mmap调用内核的HPET系统接口实现。


初始化和运行
~~~~~~~~~~~~

初始化部分从glibc的开始函数就执行了。
检查也在初始化过程中被执行，用于保证配置文件所选择的架构宏定义是本CPU所支持的，然后才开始调用main函数。
Core的初始化和运行时在rte_eal_init()接口上执行的（参考API文档）。它包括对pthread库的调用（更具体的说是
pthread_self(), pthread_create(), 和pthread_setaffinity_np()函数）。

.. _figure_linuxapp_launch:

.. figure:: img/linuxapp_launch.*

   EAL在Linux APP环境中被初始化。


.. note::

    对象的初始化，例如内存区间、ring、内存池、lpm表或hash表等，必须作为整个程序初始化的一部分，在主逻辑核上完成。
    创建和初始化这些对象的函数不是多线程安全的，但是，一旦初始化完成，这些对象本身可以作为安全线程运行。

多进程支持
~~~~~~~~~~

Linux下APP支持像多线程一样部署多进程运行模式，具体参考 :ref:`Multi-process Support <Multi-process_Support>` 。

内存映射和内存分配
~~~~~~~~~~~~~~~~~~~

大量连续的物理内存分配时通过hugetlbfs内核文件系统来实现的。
EAL提供了相应的接口用于申请指定名字的连续内存空间。
这个API同时会将这段连续空间的地址返回给用户程序。

.. note::

    内存申请时使用rte_malloc接口来做的，它也是hugetlbfs文件系统大页支持的。

Xen Dom0非大页运行支持
~~~~~~~~~~~~~~~~~~~~~~

现存的内存管理是基于Linux内核的大页机制，
然而，Xen Dom0并不支持大页，所以要将一个新的内核模块rte_dom0_mem加载上，以便避开这个限制。

EAL使用IOCTL接口用于通告Linux内核模块rte_mem_dom0去申请指定大小的内存块，并从该模块中获取内存段的信息。
EAL使用MMAP接口来映射这段内存。
对于申请到的内存段，在其内的物理地址都是连续的，但是实际上，硬件地址只在2M内连续。

PCI 访问
~~~~~~~~

EAL使用Linux内核提供的文件系统 /sys/bus/pci 来扫描PCI总线上的内容。
内核模块uio_pci_generic提供了/dev/uioX设备文件及/sys下对应的资源文件用于访问PCI设备。
DPDK特有的igb_uio模块也提供了相同的功能用于PCI设备的访问。
这两个驱动模块都用到了Linux内核提供的uio特性。

逻辑核及共享变量
~~~~~~~~~~~~~~~~

.. note::

    逻辑核就是处理器的逻辑单元，有时候也称为硬件 *线程*。

共享变量是默认的做法。
每逻辑核变量的实现则是通过线程局部存储技术TLS来实现的，它提供了每个线程本地存储的功能。


日志
~~~~

EAL提供了日志信息接口。
默认的，在linux 应用程序中，日志信息被发送到syslog和concole中。
当然，用户可以通过使用不同的日志机制来代替DPDK的日志功能。



跟踪与调试功能
^^^^^^^^^^^^^^

Glibc中提供了一些调试函数用于打印堆栈信息。
Rte_panic函数可以产生一个SIG_ABORT信号，这个信号可以触发产生core文件，可以通过gdb来加载调试。

CPU 特性识别
~~~~~~~~~~~~

EAL可以在运行时查询CPU状态（使用rte_cpu_get_feature()接口），用于决定哪个CPU可用。

用户空间中断事件
~~~~~~~~~~~~~~~~

+ 主线程中的用户空间中断和警告处理

EAL创建一个主线程用于轮询UIO设备描述文件以检测中断。
可以通过EAL提供的函数为特定的中断事件注册/解注册回掉函数，回掉函数在主线程中被异步调用。
EAL同时也允许像NIC中断那样定时调用回掉函数。

.. note::

    在DPDK的PMD中，主线程只对连接状态改变的中断处理，例如网卡的打开和关闭。


+ RX 中断事件

PMD提供的报文收发程序并不只限制于自身轮询下执行。
为了缓解小吞吐量下轮询模式对CPU资源的浪费，暂停轮询并等待唤醒事件发生时一种有效的手段。
收包中断是这种场景的一种很好的选择，但也不是唯一的。

EAL提供了事件驱动模式相关的API。以Linux APP为例，其实现依赖于epoll技术。
每个线程可以监控一个epoll实例，而在实例中可以添加所有需要的wake-up事件文件描述符。
事件文件描述符创建并根据UIO/VFIO的说明来映射到制定的中断向量上。
对于BSD APP，可以使用kqueue来代替，但是目前尚未实现。

EAL初始化中断向量和事件文件描述符之间的映射关系，同时每个设备初始化中断向量和队列之间的映射关系，
这样，EAL实际上并不知道在指定向量上发生的中断，由设备驱动负责执行后面的映射。

.. note::

    每个RX中断事件队列只支持VFIO模式，VFIO支持多个MSI-X向量。
    在UIO中，收包中断和其他中断共享中断向量，因此，当RX中断和LSC（连接状态改变）中断同时发生时，只有前者生效。

RX中断由API（rte_eth_dev_rx_intr_*）来实现控制、使能、关闭。当PMD不支持时，这些API返回失败。Intr_conf.rxq标识用于打开每个设备的RX中断。

黑名单
~~~~~~

EAL PCI设备的黑名单功能是用于标识制定NIC端口，以便DPDK忽略该端口。
可以使用PCIe设备地址描述符（Domain:Bus:Device:Function）将对应端口标记为黑名单。

Misc功能
~~~~~~~~

包括锁和原子操作（i686和x86-64架构）。

内存段和内存区间
----------------

物理内存映射就是通过EAL的这个特性实现的。
物理内存块之间可能是不连续的，所有的内存通过一个内存描述符表进行管理，且每个描述符指向一块连续的物理内存。

基于此，内存区块分配器的作用就是保证分配到一块连续的物理内存。
这些区块被分配出来时会用一个唯一的名字来标识。

Rte_memzone描述符也在配置结构体中，可以通过rte_eal_get_configuration()接口来获取。
通过名字访问一个内存区块会返回对应内存区块的描述符。

内存分配可以从指定开始地址和对齐方式来分配（默认是cache line大小对齐），对齐一般是以2的次幂来的，并且不小于64字节对齐。
内存区可以是2M或是1G大小的内存页，这两者系统都支持。


多线程
------

DPDK通常制定在core上跑线程以避免任务在核上切换的开销。
这有利于性能的提升，但不总是有效的，并且缺乏灵活性。

电源管理通过限制CPU的运行频率来提升CPU的工作效率。
当然，我们也可以通过充分利用CPU的空闲周期来提升效率。

通过使用cgroup技术，CPU的使用量可以很方便的分配，这也提供了新的方法来提升CPU性能，
但是这里有个前提，DPDK必须处理每个核多线程的上下文切换。

想要更多的灵活性，就要设置线程的CPU亲和性是对CPU集合而不是CPU了。

EAL pthread and lcore Affinity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The term "lcore" refers to an EAL thread, which is really a Linux/FreeBSD pthread.
"EAL pthreads"  are created and managed by EAL and execute the tasks issued by *remote_launch*.
In each EAL pthread, there is a TLS (Thread Local Storage) called *_lcore_id* for unique identification.
As EAL pthreads usually bind 1:1 to the physical CPU, the *_lcore_id* is typically equal to the CPU ID.

When using multiple pthreads, however, the binding is no longer always 1:1 between an EAL pthread and a specified physical CPU.
The EAL pthread may have affinity to a CPU set, and as such the *_lcore_id* will not be the same as the CPU ID.
For this reason, there is an EAL long option '--lcores' defined to assign the CPU affinity of lcores.
For a specified lcore ID or ID group, the option allows setting the CPU set for that EAL pthread.

The format pattern:
	--lcores='<lcore_set>[@cpu_set][,<lcore_set>[@cpu_set],...]'

'lcore_set' and 'cpu_set' can be a single number, range or a group.

A number is a "digit([0-9]+)"; a range is "<number>-<number>"; a group is "(<number|range>[,<number|range>,...])".

If a '\@cpu_set' value is not supplied, the value of 'cpu_set' will default to the value of 'lcore_set'.

    ::

    	For example, "--lcores='1,2@(5-7),(3-5)@(0,2),(0,6),7-8'" which means start 9 EAL thread;
    	    lcore 0 runs on cpuset 0x41 (cpu 0,6);
    	    lcore 1 runs on cpuset 0x2 (cpu 1);
    	    lcore 2 runs on cpuset 0xe0 (cpu 5,6,7);
    	    lcore 3,4,5 runs on cpuset 0x5 (cpu 0,2);
    	    lcore 6 runs on cpuset 0x41 (cpu 0,6);
    	    lcore 7 runs on cpuset 0x80 (cpu 7);
    	    lcore 8 runs on cpuset 0x100 (cpu 8).

Using this option, for each given lcore ID, the associated CPUs can be assigned.
It's also compatible with the pattern of corelist('-l') option.

非EAL的线程支持
~~~~~~~~~~~~~~~

可以在任何用户线程（non-EAL线程）上执行DPDK任务上下文。
在non-EAL线程中，*_lcore_id* 始终是 LCORE_ID_ANY，它标识一个no-EAL线程的有效、唯一的 *_lcore_id*。
一些库可能会使用一个唯一的ID替代，一些库将不受影响，有些库虽然能工作，但是会受到限制（如定时器和内存池库）。

所有这些影响将在 :ref:`known_issue_label` 章节中提到。

公共线程API
~~~~~~~~~~~

DPDK为线程操作引入了两个公共API ``rte_thread_set_affinity()`` 和 ``rte_pthread_get_affinity()``。
当他们在任何线程上下文中调用时，将获取或设置线程本地存储(TLS)。

这些TLS包括 *_cpuset* 和 *_socket_id*：

*	*_cpuset* 存储了与线程相关联的CPU位图。

*	*_socket_id* 存储了CPU set所在的NUMA节点。如果CPU set中的cpu属于不同的NUMA节点, *_socket_id* 将设置为SOCKET_ID_ANY。


.. _known_issue_label:

已知问题
~~~~~~~~

+ rte_mempool

  rte_mempool在mempool中使用per-lcore缓存。对于non-EAL线程，``rte_lcore_id()`` 无法返回一个合法的值。
  因此，当rte_mempool与non-EAL线程一起使用时，put/get操作将绕过默认的mempool缓存，这个旁路操作将造成性能损失。
  结合 ``rte_mempool_generic_put()`` 和 ``rte_mempool_generic_get()`` 可以在non-EAL线程中使用用户拥有的外部缓存。

+ rte_ring

  rte_ring支持多生产者入队和多消费者出队操作。
  然而，这是非抢占的，这使得rte_mempool操作都是非抢占的。

  .. note::

    "非抢占" 意味着：

    - 在给定的ring上做入队操作的pthread不能被另一个在同一个ring上做入队的pthread抢占
    - 在给定ring上做出对操作的pthread不能被另一个在同一ring上做出队的pthread抢占

    绕过此约束则可能造成第二个进程自旋等待，知道第一个进程再次被调度为止。
    此外，如果第一个线程被优先级较高的上下文抢占，甚至可能造成死锁。

  这并不意味着不能使用它，简单讲，当同一个core上的多线程使用时，需要缩小这种情况。

  1. 它可以用于任一单一生产者或者单一消费者的情况。

  2. 它可以由多生产者/多消费者使用，要求调度策略都是SCHED_OTHER(cfs)。用户需要预先了解性能损失。

  3. 它不能被调度策略是SCHED_FIFO 或 SCHED_RR的多生产者/多消费者使用。

+ rte_timer

  不允许在non-EAL线程上运行 ``rte_timer_manager()``。但是，允许在non-EAL线程上重置/停止定时器。

+ rte_log

  在non-EAL线程上，没有per thread loglevel和logtype，但是global loglevels可以使用。

+ misc

  在non-EAL线程上不支持rte_ring, rte_mempool 和rte_timer的调试统计信息。

cgroup控制
~~~~~~~~~~

以下是cgroup控件使用的简单示例，在同一个核心($CPU)上两个线程(t0 and t1)执行数据包I/O。
我们期望只有50%的CPU消耗在数据包IO操作上。

  .. code-block:: console

    mkdir /sys/fs/cgroup/cpu/pkt_io
    mkdir /sys/fs/cgroup/cpuset/pkt_io

    echo $cpu > /sys/fs/cgroup/cpuset/cpuset.cpus

    echo $t0 > /sys/fs/cgroup/cpu/pkt_io/tasks
    echo $t0 > /sys/fs/cgroup/cpuset/pkt_io/tasks

    echo $t1 > /sys/fs/cgroup/cpu/pkt_io/tasks
    echo $t1 > /sys/fs/cgroup/cpuset/pkt_io/tasks

    cd /sys/fs/cgroup/cpu/pkt_io
    echo 100000 > pkt_io/cpu.cfs_period_us
    echo  50000 > pkt_io/cpu.cfs_quota_us


内存申请
--------

EAL提供了一个malloc API用于申请任意大小内存。


这个API的目的是提供类似malloc的功能，以允许从hugepage中分配内存并方便应用程序移植。
*DPDK API参考手册* 介绍了可用的功能。

通常，这些类型的分配不应该在数据面处理中进行，因为他们比基于池的分配慢，并且在分配和释放路径中使用了锁操作。
但是，他们可以在配置代码中使用。

更多信息请参阅 *DPDK API参考手册* 中rte_malloc()函数描述。

Cookies
~~~~~~~

当 CONFIG_RTE_MALLOC_DEBUG 开启时，分配的内存包括保护字段，这个字段用于帮助识别缓冲区溢出。

对齐和NUMA约束
~~~~~~~~~~~~~~

接口rte_malloc()传入一个对齐参数，该参数用于请求在该值的倍数上对齐的内存区域(这个值必须是2的幂)。

在支持NUMA的系统上，对rte_malloc()接口调用将返回在调用函数的core所在的插槽上分配的内存。
DPDK还提供了另一组API，以允许在NUMA插槽上直接显式分配内存，或者分配另一个NUAM插槽上的内存。

用例
~~~~

这个API旨在由初始化时需要类似malloc功能的应用程序调用。

为了在运行时分配/释放数据，在应用程序的快速路径中，应该使用内存池库。

内部实现
~~~~~~~~

数据结构
^^^^^^^^

Malloc库中内部使用两种数据结构类型：

*   struct malloc_heap - 用于在每个插槽上跟踪可用内存空间

*   struct malloc_elem - 库内部分配和释放空间跟踪的基本要素

Structure: malloc_heap
""""""""""""""""""""""

数据结构malloc_heap用于管理每个插槽上的可用内存空间。
在内部，每个NUMA节点有一个堆结构，这允许我们根据此线程运行的NUMA节点为线程分配内存。
虽然这并不能保证在NUMA节点上使用内存，但是它并不比内存总是在固定或随机节点上的方案更糟。

堆结构及其关键字段和功能描述如下：

*   lock - 需要锁来同步对堆的访问。
    假定使用链表来跟踪堆中的可用空间，我们需要一个锁来防止多个线程同时处理该链表。

*   free_head - 指向这个malloc堆的空闲结点链表中的第一个元素

.. note::

    数据结构malloc_heap并不会跟踪使用的内存块，因为除了要再次释放他们之外，他们不会被接触，需要释放时，将指向块的指针作为参数传给fres函数。

.. _figure_malloc_heap:

.. figure:: img/malloc_heap.*

   Malloc库中malloc heap 和 malloc elements。


.. _malloc_elem:

Structure: malloc_elem
""""""""""""""""""""""

数据结构malloc_elem用作各种内存块的通用头结构。
它以三种不同的方式使用，如上图所示：

#.  作为一个释放/申请内存的头部 -- 正常使用

#.  作为内存块内部填充头

#.  作为内存结尾标记

结构中重要的字段和使用方法如下所述：

.. note::

    如果一个字段没有上述三个用法之一的用处，则可以假设对应字段在该情况下具有未定义的值。
    例如，对于填充头，只有 "state" 和 "pad"字段具有有效的值。

*   heap - 这个指针指向了该内存块从哪个堆申请。
    它被用于正常的内存块，当他们被释放时，将新释放的块添加到堆的空闲列表中。

*   prev - 这个指针用于指向紧跟这当前memseg的头元素。当释放一个内存块时，该指针用于引用上一个内存块，检查上一个块是否也是空闲。
    如果空闲，则将两个空闲块合并成一个大块。

*   next_free - 这个指针用于将空闲块列表连接在一起。
    它用于正常的内存块，在 ``malloc()`` 接口中用于找到一个合适的空闲块申请出来，在 ``free()`` 函数中用于将内存块添加到空闲链表。

*   state - 该字段可以有三个可能值：``FREE``, ``BUSY`` 或 ``PAD``。
    前两个是指示正常内存块的分配状态，后者用于指示元素结构是在块开始填充结束时的虚拟结构，即，由于对齐限制，块内的数据开始的地方不在块本身的开始处。
	在这种情况下，pad头用于定位块的实际malloc元素头。
    对于结尾的结构，这个字段总是 ``BUSY`` ，它确保没有元素在释放之后搜索超过 memseg的结尾以供其它块合并到更大的空闲块。

*   pad - 这个字段为块开始处的填充长度。
    在正常块头部情况下，它被添加到头结构的结尾，以给出数据区的开始地址，即在malloc上传回的地址。
    在填充虚拟头部时，存储相同的值，并从虚拟头部的地址中减去实际块头部的地址。

*   size - 数据块的大小，包括头部本身。
    对于结尾结构，这个大小需要指定为0，虽然从未使用。
    对于正在释放的正常内存块，使用此大小值替代 "next" 指针，以标识下一个块的存储位置，在 ``FREE`` 情况下，可以合并两个空闲块。

申请内存
^^^^^^^^

On EAL initialization, all memsegs are setup as part of the malloc heap.
This setup involves placing a dummy structure at the end with ``BUSY`` state,
which may contain a sentinel value if ``CONFIG_RTE_MALLOC_DEBUG`` is enabled,
and a proper :ref:`element header<malloc_elem>` with ``FREE`` at the start
for each memseg.
``FREE``元素被添加到malloc堆的空闲列表中。

当应用程序调用类似malloc功能的函数时，malloc函数将首先为调用线程索引 ``lcore_config`` 结构，并确定该线程的NUMA节点。
NUMA节点将作为参数传给 ``heap_alloc()``函数，用于索引 ``malloc_heap`` 结构数组。参与索引参数还有大小、类型、对齐方式和边界参数。

函数 ``heap_alloc()`` 将扫描堆的空闲链表，尝试找到一个适用于所请求的大小、对齐方式和边界约束的内存块。

当已经识别出合适的空闲元素时，将计算要返回给用户的指针。
紧跟在该指针之前的内存的高速缓存行填充了一个malloc_elem头部。
由于对齐和边界约束，在元素的开头和结尾可能会有空闲的空间，这将导致已下行为：

#. 检查尾随空间。
   如果尾部空间足够大，例如 > 128 字节，那么空闲元素将被分割。
   否则，仅仅忽略它（浪费空间）。

#. 检查元素开始处的空间。
   如果起始处的空间很小， <=128 字节，那么使用填充头，这部分空间被浪费。
   但是，如果空间很大，那么空闲元素将被分割。

从现有元素的末尾分配内存的优点是不需要调整空闲链表，
空闲链表中现有元素仅调整大小指针，并且后面的元素使用 "prev" 指针重定向到新创建的元素位置。

释放内存
^^^^^^^^

要释放内存，将指向数据区开始的指针传递给free函数。
从该指针中减去 ``malloc_elem`` 结构的大小，以获得内存块元素头部。
如果这个头部类型是 ``PAD``，那么进一步减去pad长度，以获得整个块的正确元素头。

从这个元素头中，我们获得指向块所分配的堆的指针及必须被释放的位置，以及指向前一个元素的指针，
并且通过size字段，可以计算下一个元素的指针。
这意味着我们永远不会有两个相邻的 ``FREE`` 内存块，因为他们总是会被合并成一个大的块。
