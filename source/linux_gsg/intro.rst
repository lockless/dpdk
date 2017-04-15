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

简介
====

本文档包含DPDK软件安装和配置的相关说明。旨在帮助用户快速启动和运行软件。文档主要描述了在Linux环境下编译和
运行DPDK应用程序，但是文档并不深入DPDK的具体实现细节。

文档地图
--------

以下是一份建议顺序阅读的DPDK参考文档列表：

*   发布说明：提供特性发行版本的信息，包括支持的功能，限制，修复的问题，已知的问题等等。此外，还以FAQ方式提供了常见问题及解答。

*   入门指南（本文档）：介绍如何安装和配置DPDK，旨在帮助用户快速上手。

*   编程指南：描述如下内容：

    *   软件架构及如何使用（实例介绍），特别是在Linux环境中的用法

    *   DPDK的主要内容，系统构建（包括可以在DPDK根目录Makefile中来构建工具包和应用程序的命令）及应用移植细则。

    *   软件中使用的，以及新开发中需要考虑的一些优化。

    还提供了文档使用的术语表。

*   API参考：提供有关DPDK功能、数据结构和其他编程结构的详细信息。

*   示例程序用户指南：描述了一组例程。
    每个章节描述了一个用例，展示了具体的功能，并提供了有关如何编译、运行和使用的说明。
