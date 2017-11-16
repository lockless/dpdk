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

.. _LPM_Library:

LPM库
======

DPDK LPM库组件实现了32位Key的最长前缀匹配（LPM）表搜索方法，该方法通常用于在IP转发应用程序中找到最佳路由。

LPM API概述
----------------

LPM组件实例的主要配置参数是要支持的最大数量的规则。LPM前缀由一对参数（32位Key，深度）表示，深度范围为1到32。LPM规则由LPM前缀和与前缀相关联的一些用户数据表示。该前缀作为LPM规则的唯一标识符。在该实现中，用户数据为1字节长，被称为下一跳，与其在路由表条目中存储下一跳的ID的主要用途相关。

LPM组件导出的主要方法有：

*   添加LPM规则：LPM规则作为输入参数。如果表中没有存在相同前缀的规则，则将新规则添加到LPM表中。如果表中已经存在具有相同前缀的规则，则会更新规则的下一跳。当没有可用的规则空间时，返回错误。

*   删除LPM规则：LPM规则的前缀作为输入参数。如果具有指定前缀的规则存在于LPM表中，则会被删除。

*   LPM规则查找：32位Key作为输入参数。该算法用于选择给定Key的最佳匹配的LPM规则，并返回该规则的下一跳。在LPM表中具有多个相同32位Key的规则的情况下，算法将最高深度的规则选为最佳匹配规则（最长前缀匹配），这意味着该规则Key和输入的Key之间具有最高有效位的匹配。

.. _lpm4_details:

实现细节
----------

目前的实现使用DIR-24-8算法的变体，可以改善内存使用量，以提高LPM查找速度。该算法允许以典型的单个存储器读访问来执行查找操作。在统计上看，即便是不常出现的情况，当即最佳匹配规则的深度大于24时，查找操作也仅需要两次内存读取访问。因此，特定存储器位置是否存在于处理器高速缓存中将很大程度上影响LPM查找操作的性能。

主要数据结构使用以下元素构建：

*   一个2^24个条目的表。

*   多个表（RTE_LPM_TBL8_NUM_GROUPS），每个表有2 ^ 8个条目。

第一个表，称为tbl24，使用要查找的IP地址的前24位进行索引；而第二个表，称为tbl8使用IP地址的最后8位进行索引。这意味着根据输入数据包的IP地址与存储在tbl24中的规则进行匹配的结果，我们可能需要在第二级继续查找过程。

由于tbl24的每个条目都可以指向tbl8，理想情况下，我们将具有2 ^ 24 tbl8，这与具有2 ^ 32个条目的单个表占用空间相同。因为资源限制，这显然是不可行的。相反，这种组织方法就是利用了超过24位的规则是非常罕见的这一特定。通过将这个过程分为两个不同的表/级别并限制tbl8的数量，我们可以大大降低内存消耗，同时保持非常好的查找速度（大部分时间仅一个内存访问）。


.. figure:: img/tbl24_tbl8.*

   Table split into different levels


tbl24中的条目包含以下字段：

* 下一跳，或者下一级查找表tbl8的索引值。
* 有效标志。
* 外部条目标志。
* 规则深度。

第一个字段可以包含指示查找过程应该继续的tbl8的数字，或者如果已经找到最长的前缀匹配，则可以包含下一跳本身。两个标志字段用于确定条目是否有效，以及搜索过程是否分别完成。规则的深度或长度是存储在特定条目中的规则的位数。

tbl8中的条目包含以下字段：

*   下一跳。

*   有效标志。

*   有效组。

*   深度。

下一跳和深度包含与tbl24中相同的信息。两个标志字段显示条目和表分别是否有效。

其他主要数据结构是包含有关规则（IP和下一跳）的主要信息的表。这是一个更高级别的表，用于不同的东西：

*   在添加或删除之前，检查规则是否已经存在，而无需实际执行查找。

*   删除时，检查是否存在包含要删除的规则。这很重要，因为主数据结构必须相应更新。

添加
~~~~~~~~

添加规则时，存在不同的可能性。如果规则的深度恰好是24位，那么：

*   使用规则（IP地址）作为tbl24的索引。

*   如果条目无效（即它不包含规则），则将其下一跳设置为其值，将有效标志设置为1（表示此条目正在使用中），并将外部条目标志设置为0（表示查找 此过程结束，因为这是匹配的最长的前缀）。

如果规则的深度正好是32位，那么：

*   使用规则的前24位作为tbl24的索引。

*   如果条目无效（即它不包含规则），则查找一个空闲的tbl8，将该值的tbl8的索引设置为该值，将有效标志设置为1（表示此条目正在使用中），并将外部 条目标志为1（意味着查找过程必须继续，因为规则尚未被完全探测）。

如果规则的深度是任何其他值，则必须执行前缀扩展。这意味着规则被复制到所有下一级条目（只要它们不被使用），这也将导致匹配。

作为一个简单的例子，我们假设深度是20位。这意味着有可能导致匹配的IP地址的前24位的2 ^（24 - 20）= 16种不同的组合。因此，在这种情况下，我们将完全相同的条目复制到由这些组合索引的每个位置。

通过这样做，我们确保在查找过程中，如果存在与IP地址匹配的规则，则可以在一个或两个内存访问中找到，具体取决于是否需要移动到下一个表。前缀扩展是该算法的关键之一，因为它通过添加冗余来显着提高速度。

查询
~~~~~~

查找过程要简单得多，速度更快。在这种情况下：

*   使用IP地址的前24位作为tbl24的索引。如果该条目未被使用，那么这意味着我们没有匹配此IP的规则。如果它有效并且外部条目标志设置为0，则返回下一跳。

*   如果它是有效的并且外部条目标志被设置为1，那么我们使用tbl8索引来找出要检查的tbl8，并且将该IP地址的最后8位作为该表的索引。类似地，如果条目未被使用，那么我们没有与该IP地址匹配的规则。如果它有效，则返回下一跳。

规则数目的限制
~~~~~~~~~~~~~~~~~~

规则数量受到诸多不同因素的限制。第一个是规则的最大数量，这是通过API传递的参数。一旦达到这个数字，就不可能再添加任何更多的规则到路由表，除非有一个或多个删除。

第二个因素是算法的内在限制。如前所述，为了避免高内存消耗，tbl8的数量在编译时间有限（此值默认为256）。如果我们耗尽tbl8，我们将无法再添加任何规则。特定路由表中需要多少路由表是很难提前确定的。

只要我们有一个深度大于24的新规则，并且该规则的前24位与先前添加的规则的前24位不同，就会消耗tbl8。如果相同，那么新规则将与前一个规则共享相同的tbl8，因为两个规则之间的唯一区别是在最后一个字节内。

默认值为256情况下，我们最多可以有256个规则，长度超过24位，且前三个字节都不同。由于长度超过24位的路由不太可能，因此在大多数设置中不应该是一个问题。即便如此，tbl8的数量也可以通过设置更改。

用例：IPv4转发
~~~~~~~~~~~~~~~~

LPM算法用于实现IPv4转发的路由器所使用的无类别域间路由（CIDR）策略。

References
~~~~~~~~~~

*   RFC1519 Classless Inter-Domain Routing (CIDR): an Address Assignment and Aggregation Strategy
    `http://www.ietf.org/rfc/rfc1519 <http://www.ietf.org/rfc/rfc1519>`_

*   Pankaj Gupta, Algorithms for Routing Lookups and Packet Classification, PhD Thesis, Stanford University, 2000  (`http://klamath.stanford.edu/~pankaj/thesis/ thesis_1sided.pdf <http://klamath.stanford.edu/~pankaj/thesis/%20thesis_1sided.pdf>`_ )
