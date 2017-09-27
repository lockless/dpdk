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

电源管理
==========

DPDK电源管理功能允许用户空间应用程序通过动态调整CPU频率或进入不同的C-State来节省功耗。

*   根据RX队列的利用率动态调整CPU频率。

*   根据自适应算法进入不同层次的C-State，以推测在没有收到数据包的情况下暂停应用的短暂时间段。

调整CPU频率的接口位于电源管理库中。C-State控制是根据不同用例实现的。

CPU频率缩放
-------------

Linux内核提供了一个用于每个lcore的CPU频率缩放的cpufreq模块。
例如，对于cpuX, /sys/devices/system/cpu/cpuX/cpufreq/具有以下用于频率缩放的sys文件：

*   affected_cpus

*   bios_limit

*   cpuinfo_cur_freq

*   cpuinfo_max_freq

*   cpuinfo_min_freq

*   cpuinfo_transition_latency

*   related_cpus

*   scaling_available_frequencies

*   scaling_available_governors

*   scaling_cur_freq

*   scaling_driver

*   scaling_governor

*   scaling_max_freq

*   scaling_min_freq

*   scaling_setspeed

在DPDK中，scaling_governor在用户空间中配置。然后，用户空间应用程序可以通过写入scaling_setspeed来提示内核以根据用户空间应用程序定义的策略来调整CPU频率。

通过C-States调节Core负载
--------------------------

只要指定的lcore无任务执行，可以通过设置睡眠来改变Core状态。
在DPDK中，如果在轮询后没有接收到分组，则可以根据用户空间应用定义的策略来触发睡眠。

电源管理库API概述
---------------------

电源管理库导出的主要方法是CPU频率缩放，包括:

*   **频率上升**: 提示内核扩大特定lcore的频率。

*   **频率下降**: 提示内核缩小特定lcore的频率。

*   **频率最大**: 提示内核将特定lcore的频率最大化。

*   **频率最小**: 提示内核将特定lcore的频率降至最低。

*   **获取有效的频率**: 从sys文件中读取特定lcore的可用频率。

*   **Freq获取**: 获取当前的特定lcore的频率。

*   **频率设置**: 提示内核为特定的lcore设置频率。

示例
------

电源管理机制可用于在进行L3转发时节省功耗。

参考
------

*   l3fwd-power: DPDK提供的示例应用程序，实现功耗管理下的L3转发。

*   “功耗管理下的L3转发”章节请参阅《DPDK Sample Application’s User Guide》。
