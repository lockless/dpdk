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

.. _Ring_Library:

Ring 库
=======

环形缓冲区支持队列管理。rte_ring并不是具有无限大小的链表，它具有如下属性：

*   先进先出（FIFO）

*   最大大小固定，指针存储在表中

*   无锁实现

*   多消费者或单消费者出队操作

*   多生产者或单生产者入队操作

*   批量出队 - 如果成功，将指定数量的元素出队，否则什么也不做

*   批量入队 - 如果成功，将指定数量的元素入队，否则什么也不做

*   突发出队 - 如果指定的数目出队失败，则将最大可用数目对象出队

*   突发入队 - 如果指定的数目入队失败，则将最大可入队数目对象入队

相比于链表，这个数据结构的优点如下：

*   更快；只需要一个sizeof(void \*)的Compare-And-Swap指令，而不是多个双重比较和交换指令

*   与完全无锁队列像是

*   适应批量入队/出队操作。
    因为指针是存储在表中的，应i多个对象的出队将不会产生于链表队列中一样多的cache miss。
    此外，批量出队成本并不比单个对象出队高。

缺点：

*   大小固定

*   大量ring相比于链表，消耗更多的内存，空ring至少包含n个指针。

数据结构中存储的生产者和消费者头部和尾部指针显示了一个简化版本的ring。

.. _figure_ring1:

.. figure:: img/ring1.*

   Ring 结构


FreeBSD* 中 Ring 的实现参考
---------------------------

FreeBSD 8.0中添加了如下代码，并应用到了某些网络设备驱动程序中（至少Interl驱动中应用了）：

    * `bufring.h in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/sys/buf_ring.h?revision=199625&amp;view=markup>`_

    * `bufring.c in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/kern/subr_bufring.c?revision=199625&amp;view=markup>`_

Linux* 中的无锁环形缓冲区
-------------------------

参考链接 `Linux Lockless Ring Buffer Design <http://lwn.net/Articles/340400/>`_.

附加功能
--------

Name
~~~~

每个ring都有唯一的名字。
用户不可能创建两个具有相同名称的ring（如果尝试调用rte_ring_create()这样做的话，将返回NULL）。

使用场景
---------

Ring库的使用情况包括：

    *  DPDK app之间的交互

    *  用于内存池申请

Ring Buffer解析
---------------

本节介绍ring buffer的运行方式。
Ring结构有两组头尾指针组成，一组被生产者调用，一组被消费者调用。
以下将简单称为 prod_head、prod_tail、cons_head 及 cons_tail。

每个图代表了ring的简化状态，是一个循环缓冲器。
本地变量的内容在图上方表示，Ring结构的内容在图下方表示。

单生产者入队
~~~~~~~~~~~~

本节介绍了一个生产者向队列添加对象的情况。
在本例中，只有生产者头和尾指针(prod_head and prod_tail)被修改，只有一个生产者。

初始状态是将prod_head 和 prod_tail 指向相同的位置。

入队第一步
^^^^^^^^^^

首先， *ring->prod_head* 和 *ring->cons_tail*复制到本地变量中。
*prod_next* 本地变量指向下一个元素，或者，如果是批量入队的话，指向下几个元素。

如果ring中没有足够的空间存储元素的话(通过检查cons_tail来确定)，则返回错误。


.. _figure_ring-enqueue1:

.. figure:: img/ring-enqueue1.*

   Enqueue first step


入队第二步
^^^^^^^^^^

第二步是在环结构中修改 *ring->prod_head*，以指向与prod_next相同的位置。

指向待添加对象的指针被复制到ring中。


.. _figure_ring-enqueue2:

.. figure:: img/ring-enqueue2.*

   Enqueue second step


入队最后一步
^^^^^^^^^^^^

一旦将对象添加到ring中，ring结构中的 *ring->prod_tail* 将被修改，指向与 *ring->prod_head* 相同的位置。
入队操作完成。


.. _figure_ring-enqueue3:

.. figure:: img/ring-enqueue3.*

   Enqueue last step


单消费者出队
~~~~~~~~~~~~

本节介绍一个消费者从ring中取出对象的情况。
在本例中，只有消费者头尾指针(cons_head and cons_tail)被修改，只有一个消费者。

初始状态是将cons_head 和 cons_tail指向相同位置。

出队第一步
^^^^^^^^^^^^^

首先，将 *ring->cons_head* 和 *ring->prod_tail*复制到局部变量中。
*cons_next* 本地变量指向表的下一个元素，或者在批量出队的情况下指向下几个元素。

如果ring中没有足够的对象用于出队(通过检查prod_tail)，将返回错误。


.. _figure_ring-dequeue1:

.. figure:: img/ring-dequeue1.*

   Dequeue last step


出队第二步
^^^^^^^^^^^^

第二步是修改ring结构中 ring->cons_head，以指向cons_next相同的位置。

指向出队对象(obj1) 的指针被复制到用户指定的指针中。


.. _figure_ring-dequeue2:

.. figure:: img/ring-dequeue2.*

   Dequeue second step


出队最后一步
^^^^^^^^^^^^^

最后，ring中的ring->cons_tail被修改为指向ring->cons_head相同的位置。
出队操作完成。


.. _figure_ring-dequeue3:

.. figure:: img/ring-dequeue3.*

   Dequeue last step


多生产者入队
~~~~~~~~~~~~~

本节说明两个生产者同时向ring中添加对象的情况。
在本例中，仅修改生产者头尾指针(prod_head and prod_tail)。

初始状态是将prod_head 和 prod_tail 指向相同的位置。

多生产者入队第一步
^^^^^^^^^^^^^^^^^^^^^^

在生产者的两个core上， *ring->prod_head* 及 *ring->cons_tail* 都被复制到局部变量。
局部变量prod_next指向下一个元素，或者在批量入队的情况下指向下几个元素。

如果ring中没有足够的空间用于入队(通过检查cons_tail)，将返回错误。


.. _figure_ring-mp-enqueue1:

.. figure:: img/ring-mp-enqueue1.*

   Multiple producer enqueue first step


多生产者入队第二步
^^^^^^^^^^^^^^^^^^^^^

第二步是修改ring结构中 ring->prod_head ，来指向prod_next相同的位置。
此操作使用比较和交换(CAS)指令，该指令以原子操作的方式执行以下操作：

*   如果ring->prod_head 与本地变量prod_head不同，则CAS操作失败，代码将在第一步重新启动。

*   否则，ring->prod_head设置为本地变量prod_next，CAS操作成功并继续下一步处理。

在图中，core1执行成功，core2重新启动。


.. _figure_ring-mp-enqueue2:

.. figure:: img/ring-mp-enqueue2.*

   Multiple producer enqueue second step


多生产者入队第三步
^^^^^^^^^^^^^^^^^^^^

Core 2的CAS操作成功重试。

Core 1更新一个对象(obj4)到ring上。Core 2更新一个对象(obj5)到ring上


.. _figure_ring-mp-enqueue3:

.. figure:: img/ring-mp-enqueue3.*

   Multiple producer enqueue third step


多生产者入队地四步
^^^^^^^^^^^^^^^^^^^^

每个core现在都想更新 ring->prod_tail。
只有ring->prod_tail等于prod_head本地变量，core才能更新它。
当前只有core 1满足，操作在core 1上完成。


.. _figure_ring-mp-enqueue4:

.. figure:: img/ring-mp-enqueue4.*

   Multiple producer enqueue fourth step


多生产者入队最后一步
^^^^^^^^^^^^^^^^^^^^^^

一旦ring->prod_tail被core 1更新完，core 2也满足条件，允许更新。
Core 2上也完成了操作。


.. _figure_ring-mp-enqueue5:

.. figure:: img/ring-mp-enqueue5.*

   Multiple producer enqueue last step


32-bit取模索引
~~~~~~~~~~~~~~~

在前面的途中，prod_head, prod_tail, cons_head 和 cons_tail索引由箭头表示。
但是，在实际实现中，这些值不会假定在0和 size(ring)-1 之间。
索引值在 0 ～ 2^32 -1之间，当我们访问ring本身时，我们屏蔽他们的值。
32bit模数也意味着如果溢出32bit的范围，对索引的操作将自动执行2^32 模。

以下是两个例子，用于帮助解释索引值如何在ring中使用。

.. note::

    为了简化说明，使用模16bit操作，而不是32bit。
    另外，四个索引被定义为16bit无符号整数，与实际情况下的32bit无符号数相反。


.. _figure_ring-modulo1:

.. figure:: img/ring-modulo1.*

   Modulo 32-bit indexes - Example 1


这个ring包含11000对象。


.. _figure_ring-modulo2:

.. figure:: img/ring-modulo2.*

      Modulo 32-bit indexes - Example 2


这个ring包含12536个对象。

.. note::

    为了便于理解，我们在上面的例子中使用模65536操作。
    在实际执行情况下，这种低效操作是多余的，但是，当溢出时会自动执行。

代码始终保证生产者和消费者之间的距离在0 ～ size(ring)-1之间。
基于这个属性，我们可以对两个索引值做减法，而不用考虑溢出问题

任何情况下，ring中的对象和空闲对象都在 0 ～ size(ring)-1之间，即便第一个减法操作已经溢出：

.. code-block:: c

    uint32_t entries = (prod_tail - cons_head);
    uint32_t free_entries = (mask + cons_tail -prod_head);

参考文档
----------

    *   `bufring.h in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/sys/buf_ring.h?revision=199625&amp;view=markup>`_ (version 8)

    *   `bufring.c in FreeBSD <http://svn.freebsd.org/viewvc/base/release/8.0.0/sys/kern/subr_bufring.c?revision=199625&amp;view=markup>`_ (version 8)

    *   `Linux Lockless Ring Buffer Design <http://lwn.net/Articles/340400/>`_
