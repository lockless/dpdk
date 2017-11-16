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

.. _Development_Kit_Build_System:

开发套件构建系统
==================

DPDK 需要一个构建系统用于编译等操作。
本节介绍 DPDK 框架中使用的约束和机制。

这个框架有两个使用场景：

*   编译DPDK库和示例应用程序，该框架生成特定的二进制库，包含文件和示例应用程序。

*   使用安装的DPDK二进制编译外部的应用程序或库。

编译DPDK二进制文件
--------------------

以下提供了如何构建DPDK二进制文件。

建立目录概念
~~~~~~~~~~~~~~

安装之后，将创建一个构建目录结构。
每个构件目录包含文件、库和应用程序。

构建目录特定于配置的体系结构、执行环境、工具链。
可以存在几个构建目录共享源码，但是配置不一样的情况。

例如，要使用配置模板 config/defconfig_x86_64-linuxapp 创建一个名为 my_sdk_build_dir 的构建目录，我们使用如下命令：

.. code-block:: console

    cd ${RTE_SDK}
    make config T=x86_64-native-linuxapp-gcc O=my_sdk_build_dir

这会创建一个新的 new my_sdk_build_dir 目录，之后，我们可以使用如下的命令进行编译：

.. code-block:: console

    cd my_sdk_build_dir
    make

相当于：

.. code-block:: console

    make O=my_sdk_build_dir

目录 my_sdk_build_dir 的内容是：

::

    -- .config                         # used configuration

    -- Makefile                        # wrapper that calls head Makefile
                                       # with $PWD as build directory


        -- build                              #All temporary files used during build
        +--app                                # process, including . o, .d, and .cmd files.
            |  +-- test                       # For libraries, we have the .a file.
            |  +-- test.o                     # For applications, we have the elf file.
            |  `-- ...
            +-- lib
                +-- librte_eal
                |   `-- ...
                +-- librte_mempool
                |  +--  mempool-file1.o
                |  +--  .mempool-file1.o.cmd
                |  +--  .mempool-file1.o.d
                |  +--   mempool-file2.o
                |  +--  .mempool-file2.o.cmd
                |  +--  .mempool-file2.o.d
                |  `--  mempool.a
                `-- ...

    -- include                # All include files installed by libraries
        +-- librte_mempool.h  # and applications are located in this
        +-- rte_eal.h         # directory. The installed files can depend
        +-- rte_spinlock.h    # on configuration if needed (environment,
        +-- rte_atomic.h      # architecture, ..)
        `-- \*.h ...

    -- lib                    # all compiled libraries are copied in this
        +-- librte_eal.a      # directory
        +-- librte_mempool.a
        `-- \*.a ...

    -- app                    # All compiled applications are installed
    + --test                  # here. It includes the binary in elf format

请参阅 :ref:`Development Kit Root Makefile Help <Development_Kit_Root_Makefile_Help>` 获取更详细的信息。

构建外部应用程序
------------------

由于DPDK本质上是一个开发工具包，所以最终用户的第一个目标就是使用这个SDK创建新的应用程序。
要编译应用程序，用户必须设置 RTE_SDK 和 RTE_TARGET 环境变量。

.. code-block:: console

    export RTE_SDK=/opt/DPDK
    export RTE_TARGET=x86_64-native-linuxapp-gcc
    cd /path/to/my_app

对于一个新的应用程序，用户必须创建新的 Makefile 并包含指定的 .mk 文件，如 ${RTE_SDK}/mk/rte.vars.mk 和 ${RTE_SDK}/mk/rte.app.mk。
这部分内容描述请参考 :ref:`Building Your Own Application <Building_Your_Own_Application>`.

根据 Makefile 所选定的目标（架构、机器、执行环境、工具链）或环境变量，应用程序和库将使用适当的h头文件进行编译，并和适当的a库链接。
这些文件位于 ${RTE_SDK}/arch-machine-execenv-toolchain，由 ${RTE_BIN_SDK} 内部引用。

为了编译应用程序，用户只需要调用make命令。编译结果将置于 /path/to/my_app/build 目录。

示例应用程序在example目录中提供。

.. _Makefile_Description:

Makefile 描述
---------------

DPDK Makefiles 的通用规则
~~~~~~~~~~~~~~~~~~~~~~~~~~~

在DPDK中，Makefiles始终遵循相同的方案：

#. 起始处包含 $(RTE_SDK)/mk/rte.vars.mk 文件。

#. 为RTE构建系统定义特殊的变量。

#. 包含指定的 $(RTE_SDK)/mk/rte.XYZ.mk 文件，其中 XYZ 可以是 app、lib、extapp, extlib、obj、gnuconfigure等等，取决于要编译什么样的目标文件。
   请参阅 :ref:`See Makefile Types <Makefile_Types>` 描述。

#. 包含用户定义的规则及变量。

   以下是一个简单的例子，用于便于一个外部应用程序：

   ..  code-block:: make

        include $(RTE_SDK)/mk/rte.vars.mk

        # binary name
        APP = helloworld

        # all source are stored in SRCS-y
        SRCS-y := main.c

        CFLAGS += -O3
        CFLAGS += $(WERROR_FLAGS)

        include $(RTE_SDK)/mk/rte.extapp.mk

.. _Makefile_Types:

Makefile 类型
~~~~~~~~~~~~~~

根据Makefile最后包含的 .mk 文件，Makefile将具有不同的角色。
注意到，并不能在同一个Makefile文件中同时编译库和应用程序。
因此，用户必须创建两个独立的Makefile文件，最好是置于两个不同的目录中。

无论如何，rte.vars.mk 文件必须包含用户Makefile。

应用程序
^^^^^^^^^^

这些 Makefiles 生成一个二进制应用程序。

*   rte.app.mk: DPDK框架中的应用程序。

*   rte.extapp.mk: 外部应用程序。

*   rte.hostapp.mk: 建立DPDK的先决条件和工具。

库
^^^^

创建一个 .a 库。

*   rte.lib.mk: DPDK中的库。

*   rte.extlib.mk: 外部库。

*   rte.hostlib.mk: DPDK中的host库。

安装
^^^^^^^

*   rte.install.mk: 不构建任何东西，只是用于创建链接或者将文件复制到安装目录。
    这对于开发包框架中包含的文件非常有用。

内核模块
^^^^^^^^^^^^^

*   rte.module.mk: 构建DPDK内核模块。

对象
^^^^^^^

*   rte.obj.mk: DPDK中的目标文件聚合（合并一些o文件成一个）。

*   rte.extobj.mk: 外部目标文件聚合（合并外部的一些o文件）。

杂
^^^^

*   rte.doc.mk: DPDK中的文档。

*   rte.gnuconfigure.mk: 构建一个基于配置的应用程序。

*   rte.subdir.mk: 构建几个目录。

.. _Internally_Generated_Build_Tools:

内部生成的构建工具
~~~~~~~~~~~~~~~~~~~~~~

``app/dpdk-pmdinfogen``


``dpdk-pmdinfogen`` 扫描各种总所周知的符号名称对象文件。这些目标文件由各种宏定义，用于导出关于pmd文件的硬件支持和使用的重要信息。
例如宏定义：

.. code-block:: c

   RTE_PMD_REGISTER_PCI(name, drv)

创建以下的符号：

.. code-block:: c

   static char this_pmd_name0[] __attribute__((used)) = "<name>";


将被 ``dpdk-pmdinfogen`` 扫描。使用这个虚拟系，可以从目标文件中导出其他相关位信息，并用于产生硬件支持描述，
然后 ``dpdk-pmdinfogen`` 按照以下格式编码成 json 格式的字符串：

.. code-block:: c

   static char <name_pmd_string>="PMD_INFO_STRING=\"{'name' : '<name>', ...}\"";


然后可以通过外部工具搜索这些字符串，以确定给定库或应用程序的硬件支持。


.. _Useful_Variables_Provided_by_the_Build_System:

构建系统提供的有用变量
~~~~~~~~~~~~~~~~~~~~~~~~~

*   RTE_SDK: DPDK源码绝对路径。
    编译DPDK时，该变量由框架自动设置。
    如果编译外部应用程序，它必须由用户定义为环境变量。

*   RTE_SRCDIR: DPDK源码根路径。
    当编译DPDK时，RTE_SRCDIR = RTE_SDK。
    当编译外部应用程序时，该变量指向外部应用程序源码的跟目录。

*   RTE_OUTPUT: 输出文件的路径。
    通常情况下，他是 $(RTE_SRCDIR)/build，但是可以通过make命令中的 O= 选项来重新指定。

*   RTE_TARGET: 一个字符串，用于我们正在构建的目标。
    格式是arch-machine-execenv-toolchain。
    当编译DPDK时，目标是有构建系统从配置(.config)中推导出来的。
    当构建外部应用程序时，必须由用户在Makefile中指定或作为环境变量。

*   RTE_SDK_BIN: 参考 $(RTE_SDK)/$(RTE_TARGET)。

*   RTE_ARCH: 定义架构(i686, x86_64)。
    它与 CONFIG_RTE_ARCH 相同，但是没有字符串的双引号。

*   RTE_MACHINE: 定义机器。
    它与 CONFIG_RTE_MACHINE 相同，但是没有字符串的双引号。

*   RTE_TOOLCHAIN: 定义工具链 (gcc , icc)。
    它与 CONFIG_RTE_TOOLCHAIN 相同，但是没有字符串的双引号。

*   RTE_EXEC_ENV: 定义运行环境 (linuxapp)。
    它与 CONFIG_RTE_EXEC_ENV 相同，但是没有字符串的双引号。

*   RTE_KERNELDIR: 这个变量包含了将被用于编译内核模块的内核源的绝对路径。
    内核头文件必须与目标机器（将运行应用程序的机器）上使用的头文件相同。
    默认情况下，变量设置为 /lib/modules/$(shell uname -r)/build，当目标机器也是构建机器时，这是正确的。

*   RTE_DEVEL_BUILD: 更严格的选项（停止警告）。 它在默认的git树中。

只能在Makefile中设置/覆盖的变量
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*   VPATH: 构建系统将搜索源码的路径列表。默认情况下，RTE_SRCDIR将被包含在VPATH中。

*   CFLAGS: 用于C编译的标志。用户应该使用+=在这个变量中附加数据。

*   LDFLAGS: 用于链接的标志。用户应该使用+=在这个变量中附加数据。

*   ASFLAGS: 用于汇编的标志。用户应该使用+=在这个变量中附加数据。

*   CPPFLAGS: 用于给C预处理器赋予标志的标志（仅在汇编.S文件时有用）。用户应该使用+=在这个变量中附加数据。

*   LDLIBS: 在应用程序中，链接的库列表（例如，-L / path / to / libfoo -lfoo）。用户应该使用+=在这个变量中附加数据。

*   SRC-y: 在应用程序，库或对象Makefiles的情况下，源文件列表（.c，.S或.o，如果源是二进制文件）。源文件必须可从VPATH获得。

*   INSTALL-y-$(INSTPATH): 需要安装到 $(INSTPATH) 的文件列表。
    这些文件必须在VPATH中可用，并将复制到 $(RTE_OUTPUT)/$(INSTPATH)。几乎可以在任何 RTE Makefile 中可用。

*   SYMLINK-y-$(INSTPATH): 需要安装到 $(INSTPATH) 的文件列表。
    这些文件必须在VPATH中可用并将链接到 (symbolically) 在 $(RTE_OUTPUT)/$(INSTPATH)。
    几乎可以在任何 RTE Makefile 中可用。

*   PREBUILD: 构建之前要采取的先决条件列表。用户应该使用+=在这个变量中附加数据。

*   POSTBUILD: 主构建之后要执行的操作列表。用户应该使用+=在这个变量中附加数据。

*   PREINSTALL: 安装前要执行的先决条件操作的列表。 用户应该使用+=在这个变量中附加数据。

*   POSTINSTALL: 安装后要执行的操作列表。用户应该使用+=在这个变量中附加数据。

*   PRECLEAN: 清除前要执行的先决条件操作列表。用户应该使用+=在这个变量中附加数据。

*   POSTCLEAN: 清除后要执行的先决条件操作列表。用户应该使用+=在这个变量中附加数据。

*   DEPDIRS-$(DIR): 仅在开发工具包框架中用于指定当前目录的构建是否依赖于另一个的构建。这是正确支持并行构建所必需的。

只能在命令行上由用户设置/覆盖的变量
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一些变量可以用来配置构建系统的行为。在文件 :ref:`Development Kit Root Makefile Help <Development_Kit_Root_Makefile_Help>` 及 :ref:`External Application/Library Makefile Help <External_Application/Library_Makefile_Help>` 中有描述。

    *   WERROR_CFLAGS: 默认情况下，它被设置为一个依赖于编译器的特定值。
        鼓励用户使用这个变量，如下所示：

            CFLAGS += $(WERROR_CFLAGS)

这避免了根据编译器（icc或gcc）使用不同的情况。而且，这个变量可以从命令行覆盖，这允许绕过标志用于测试目的。

可以在Makefile或命令行中由用户设置/覆盖的变量
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*   CFLAGS_my_file.o: 为my_file.c的C编译添加的特定标志。

*   LDFLAGS_my_app: 链接my_app时添加的特定标志。

*   EXTRA_CFLAGS: 在编译时，这个变量的内容被附加在CFLAGS之后。

*   EXTRA_LDFLAGS: 链接后，将此变量的内容添加到LDFLAGS之后。

*   EXTRA_LDLIBS: 链接后，此变量的内容被添加到LDLIBS之后。

*   EXTRA_ASFLAGS: 组装后这个变量的内容被附加在ASFLAGS之后。

*   EXTRA_CPPFLAGS: 在汇编文件上使用C预处理器时，此变量的内容将附加在CPPFLAGS之后。
