..  BSD LICENSE
    Copyright(c) 2017 Intel Corporation. All rights reserved.
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

.. _Metrics_Library:

Metrics 库
============

Metrics 库实现了一个机制，通过这个机制，*producers* 可以发布numeric信息，供 *consumers* 后续查询。
实际上，生产者通常是其他库或者主进程，而消费者通常是应用程序。

Metrics 本身是一个静态值，并不是有PMD产生的。
Metric 信息是由推送模型填充的，其中生产者通过调用相关的更新函数来跟新metric库中包含的值。
消费者通过查询共享内存中的metric数据来获取metric信息。

对于每个mettic，为每个端口ID保留一个单独的值，并且在发布metric时，生产者需要指定哪个端口正在更新。
此外，还有一个特殊的ID ``RTE_METRICS_GLOBAL``， 用于全局统计，不与任何单个设备关联。
由于metric库是自包含的，因此，对端口号的唯一限制是他们小于 ``RTE_MAX_ETHPORTS``，不需要实际端口存在。

初始化库
----------

在使用库之前，必须通过调用在共享内存中设置mettic存储的 ``rte_metrics_init()`` 来初始化它。
这也就是生产者将metric信息发布到哪里以及消费者从哪里查新metric信息。

.. code-block:: c

    rte_metrics_init(rte_socket_id());

这个初始化函数必须在主函数中调用，否则生产者和消费者可能在主程序或次进程中多次调用。？？

注册metrics 
------------

Metrics 必须先注册，这是生产者声明他们将要发布的metric的方式。
注册可以单独完成，也可以将一组metric标注为一个组。
单独注册使用接口 ``rte_metrics_reg_name()`` 实现：

.. code-block:: c

    id_1 = rte_metrics_reg_name("mean_bits_in");
    id_2 = rte_metrics_reg_name("mean_bits_out");
    id_3 = rte_metrics_reg_name("peak_bits_in");
    id_4 = rte_metrics_reg_name("peak_bits_out");

一组metric注册使用 ``rte_metrics_reg_names()`` 完成：

.. code-block:: c

    const char * const names[] = {
        "mean_bits_in", "mean_bits_out",
        "peak_bits_in", "peak_bits_out",
    };
    id_set = rte_metrics_reg_names(&names[0], 4);

如果返回负数，表示注册失败。否则，返回值表示更新metic时使用的 *key* 值。
可以使用 ``rte_metrics_get_names()`` 获得将这些key值与metric名称映射起来的映射表。

更新 metric 值
----------------

一旦注册，生产者可以使用 ``rte_metrics_update_value()`` 函数更新给定端口的metric。
这个函数使用metric注册时返回的key值，也可以使用 ``rte_metrics_get_names()`` 查找。

.. code-block:: c

    rte_metrics_update_value(port_id, id_1, values[0]);
    rte_metrics_update_value(port_id, id_2, values[1]);
    rte_metrics_update_value(port_id, id_3, values[2]);
    rte_metrics_update_value(port_id, id_4, values[3]);

如果metric被注册为一个集合，则可以使用 ``rte_metrics_update_value()`` 单独更新他们，或者使用 ``rte_metrics_update_values()`` 一起更新：

.. code-block:: c

    rte_metrics_update_value(port_id, id_set, values[0]);
    rte_metrics_update_value(port_id, id_set + 1, values[1]);
    rte_metrics_update_value(port_id, id_set + 2, values[2]);
    rte_metrics_update_value(port_id, id_set + 3, values[3]);

    rte_metrics_update_values(port_id, id_set, values, 4);

注意，``rte_metrics_update_values()`` 不能用来更新 *multiple* *sets* 的metric，因为不能保证两个集合一个接一个地注册了连续的ID值。

查询 metrics
---------------

消费者可以通过使用返回 ``struct rte_metric_value`` 数组的接口 ``rte_metrics_get_values()`` 来查询metric库。
该数组中的每个条目都包含一个metric值及其关联的key。
key值和名称的映射可以使用 ``rte_metrics_get_names()`` 函数来获得，该函数返回由key索引的 ``struct rte_metric_name`` 数组。
以下将打印给定端口的所有metric：

.. code-block:: c

    void print_metrics() {
        struct rte_metric_name *names;
        int len;

        len = rte_metrics_get_names(NULL, 0);
        if (len < 0) {
            printf("Cannot get metrics count\n");
            return;
        }
        if (len == 0) {
            printf("No metrics to display (none have been registered)\n");
            return;
        }
        metrics = malloc(sizeof(struct rte_metric_value) * len);
        names =  malloc(sizeof(struct rte_metric_name) * len);
        if (metrics == NULL || names == NULL) {
            printf("Cannot allocate memory\n");
            free(metrics);
            free(names);
            return;
        }
        ret = rte_metrics_get_values(port_id, metrics, len);
        if (ret < 0 || ret > len) {
            printf("Cannot get metrics values\n");
            free(metrics);
            free(names);
            return;
        }
        printf("Metrics for port %i:\n", port_id);
        for (i = 0; i < len; i++)
            printf("  %s: %"PRIu64"\n",
                names[metrics[i].key].name, metrics[i].value);
        free(metrics);
        free(names);
    }


Bit-rate 统计库
-----------------

Bit-rate 库计算每个活动端口（即网络设备）的指数加权平均值和峰值比特率。
这些统计信息通过metric库使用以下名称进行发布：

    - ``mean_bits_in``: 平均入站比特率
    - ``mean_bits_out``:  平均出站比特率
    - ``ewma_bits_in``: 平均入站比特率 (EWMA 平滑)
    - ``ewma_bits_out``:  平均出站比特率 (EWMA 平滑)
    - ``peak_bits_in``:  峰值入站比特率
    - ``peak_bits_out``:  峰值出站比特率

一旦初始化，并以适当的频率计时，可以通过查询metric库来获取metric值。

初始化
~~~~~~~~

在使用库之前，必须通过接口 ``rte_stats_bitrate_create()`` 来初始化，这个函数返回一个bit-rate计算对象。
由于bit-rate库使用metric来报告计算的统计量，因此bit-rate库需要将计算的统计量与metric库一起注册。
这通过辅助函数 ``rte_stats_bitrate_reg()`` 完成。

.. code-block:: c

    struct rte_stats_bitrates *bitrate_data;

    bitrate_data = rte_stats_bitrate_create();
    if (bitrate_data == NULL)
        rte_exit(EXIT_FAILURE, "Could not allocate bit-rate data.\n");
    rte_stats_bitrate_reg(bitrate_data);

控制采样速率
~~~~~~~~~~~~~~

由于库通过定期采样来工作，而不是使用内部线程，应用程序必须定期调用 ``rte_stats_bitrate_calc()`` 。
这个函数被调用的频率应该是计算统计所需要的预期采样频率。
例如，需要按秒统计，那么应该每秒钟调用一次这个函数。

.. code-block:: c

    tics_datum = rte_rdtsc();
    tics_per_1sec = rte_get_timer_hz();

    while( 1 ) {
        /* ... */
        tics_current = rte_rdtsc();
	if (tics_current - tics_datum >= tics_per_1sec) {
	    /* Periodic bitrate calculation */
	    for (idx_port = 0; idx_port < cnt_ports; idx_port++)
	            rte_stats_bitrate_calc(bitrate_data, idx_port);
		tics_datum = tics_current;
	    }
        /* ... */
    }


延迟统计库
------------

延迟统计库计算DPDK应用程序的数据包处理延迟，报告数据包处理所需的最小，平均和最大纳秒，以及处理延迟中的抖动。
使用以下名称通过metric库报告这些统计信息：

    - ``min_latency_ns``: 最小处理延迟（纳秒）
    - ``avg_latency_ns``:  平均处理延迟（纳秒）
    - ``mac_latency_ns``:  最大处理延迟（纳秒）
    - ``jitter_ns``: 处理等待时间的变化（纳秒）

一旦初始化并以适当的频率采样，可以通过查询metric库来获得这些统计数据。

初始化
~~~~~~~~~

使用库之前，需要调用函数 ``rte_latencystats_init()`` 进行初始化。

.. code-block:: c

    lcoreid_t latencystats_lcore_id = -1;

    int ret = rte_latencystats_init(1, NULL);
    if (ret)
        rte_exit(EXIT_FAILURE, "Could not allocate latency data.\n");


触发统计值更新
~~~~~~~~~~~~~~~~

需要定期调用 ``rte_latencystats_update()`` 函数，以便更新延迟统计值信息。

.. code-block:: c

    if (latencystats_lcore_id == rte_lcore_id())
        rte_latencystats_update();

关闭库
~~~~~~~~

完成之后，需要调用 ``rte_latencystats_uninit()`` 来关闭延迟统计库。

.. code-block:: c

    rte_latencystats_uninit();
