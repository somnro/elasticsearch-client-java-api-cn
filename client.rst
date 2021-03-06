.. _client:

########################################
客户端
########################################

你可以以多种方式使用 *Java 客户端*:

* 在现有集群上执行标准的 :ref:`index_api`, :ref:`get_api`, :ref:`delete_api` 和 :ref:`search_api` 操作
* 在运行着的集群上执行管理任务

获取一个 Elasticsearch 客户端很简单, 最常见的方式就是创建一个可以连接到集群的 :ref:`transport_client` 对象。

.. important:: 客户端和集群中的节点必须使用相同的主版本 (例如, ``2.x`` 或 ``5.x``)。客户端可以连接到使用不同次要版本的集群 (例如 ``2.3.x``), 但是这样的话可能就没办法支持新的功能。理想情况下, 客户端和集群应该使用完全相同的版本。

.. warning::
  我们计划在 Elasticsearch 7.0 版本中关闭 TransportClient 并且在 8.0 版本中完全删除它。你应该使用 Java 高级 REST 客户端作为替代, 它可以执行 HTTP 请求, 而不是序列化 Java 请求。`迁移指南 <https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.2/java-rest-high-level-migration.html>`_ 中描述了迁移所需的所有步骤。

  Java 高级 REST 客户端目前已经支持常用的 API, 但是仍然有很多需要添加进来。你可以对 `Java 高级 REST 客户端完整性 <https://github.com/elastic/elasticsearch/issues/27205>`_ 这个问题添加评论来告诉我们你的应用程序需要哪些缺少的API，从而帮助我们确定优先级。

  任何缺少的 API 现在都可以通过使用具有 JSON 请求和响应体的 `低级 Java REST 客户端 <https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-low.html>`_ 来实现。


.. _transport_client:

****************************************
传输客户端
****************************************

``TransportClient`` 使用传输模块远程连接到 Elasticsearch 集群。它不加入集群, 而只是简单地获取一个或多个初始传输地址并且针对每个动作以轮询的方式与传输地址进行通信(尽管大多数动作可能是"两跳"操作)。

.. code-block:: java

    // on startup

    TransportClient client = new PreBuiltTransportClient(Settings.EMPTY)
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host1"), 9300))
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host2"), 9300));

    // on shutdown

    client.close();

注意: 如果你使用的集群名称不是 "elasticsearch", 那么你需要设置一个不同的集群名称:

.. code-block:: java

    Settings settings = Settings.builder()
        .put("cluster.name", "myClusterName").build();
    TransportClient client = new PreBuiltTransportClient(settings);
    //Add transport addresses and do something with the client...

传输客户端附带了一个集群嗅探特性, 使它能够动态地添加新的主机并删除旧的主机。当启用嗅探时, 传输客户端将连接到其内部节点列表中的节点, 这个节点列表是通过调用
``addTransportAddress`` 方法来构建的。之后, 客户端将会在这些节点上调用内部的集群状态 API 来发现可用的数据节点。客户端的内部节点列表只会被那些数据节点替换。默认情况下,
该列表每五秒钟刷新一次。请注意, 嗅探器连接到的 IP 地址是在那些节点的 Elasticsearch 配置中被声明为 'publish' 的地址。

请记住, 传输客户端的内部节点列表可能不包括它连接到的原始节点, 如果那个节点不是一个数据节点的话。例如，如果你最开始连接到的是一个主节点, 在经过嗅探之后, 后续的请求没有去到该主节点,
而是到达任意的数据节点。传输客户端排除非数据节点的原因是为了避免将搜索流量发送到仅作为主节点的节点。

如果要启用嗅探, 需要将 ``client.transport.sniff`` 的值设置为 ``true``:

.. code-block:: java

    Settings settings = Settings.settingsBuilder()
        .put("client.transport.sniff", true).build();
    TransportClient client = new PreBuiltTransportClient(settings);

其它传输客户端级别的设置包括:

===============================================      ===============================================
参数                                                  描述
===============================================      ===============================================
``client.transport.ignore_cluster_name``             设置为 ``true`` 可以忽略已连接节点的集群名称校验。(从 0.19.4 版本开始)
``client.transport.ping_timeout``                    节点 ping 响应的等待时长, 默认值是 ``5s`` 。
``client.transport.nodes_sampler_interval``          对列出和连接的节点进行抽样或ping操作的频率, 默认值是 ``5s``。
===============================================      ===============================================


****************************************
客户端连接到协调节点
****************************************

你可以在本地启动一个 `协调节点 <https://www.elastic.co/guide/en/elasticsearch/reference/6.2/modules-node.html#coordinating-only-node>`_ , 然后在你的应用程序中简单地创建一个 :ref:`TransportClient` 对象并连接到这个协调节点。

这样, 协调节点能够加载任何你所需要的插件(例如发现插件)。
