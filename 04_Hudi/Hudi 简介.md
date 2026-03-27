# Hudi 简介
Apache Hudi（Hadoop <font style="background-color:#FBDE28;">Upserts Delete and Incremental</font>）是下一代流数据湖平台。

Apache Hudi 将核心仓库和数据库功能直接引入数据湖。

Hudi提供了表、事务、高效的`upserts/delete`、高级索引、流摄取服务、数据集群/压缩优化和并发，同时保持数据的开源文件格式。

Apache Hudi 不仅非常适合于**流**工作负载，而且还允许创建高效的**增量批处理管道**。

Apache Hudi 可以轻松地在任何云存储平台上使用。

Hudi的高级性能优化，使分析工作负载更快的任何流行的查询引擎，包括 Apache Spark、Flink、Presto、Trino、Hive 等。

<img src="https://cdn.nlark.com/yuque/0/2023/png/21887514/1685684603891-511d64c2-69a8-4264-bdb8-539df6b1685b.png" width="508.748046875" title="" crop="0,0,1,1" id="u9bd589ef" class="ne-image">

<img src="https://cdn.nlark.com/yuque/0/2023/png/21887514/1685684611804-50938366-f0a8-4acf-b276-a45ffd2c5340.png" width="508.748046875" title="" crop="0,0,1,1" id="u5a19bcb5" class="ne-image">

# 发展历史
+ 2015 年：发表了<font style="background-color:#FBDE28;">增量处理</font>的核心思想/原则（O'reilly 文章）。
+ 2016 年：由 Uber 创建并为所有数据库/关键业务提供支持。
+ 2017 年：由 Uber 开源，并支撑 100PB 数据湖。
+ 2018 年：吸引大量使用者，并因云计算普及。
+ 2019 年：成为 ASF 孵化项目，并增加更多平台组件。
+ 2020 年：毕业成为 Apache 顶级项目，社区、下载量、采用率增长超过 10 倍。
+ 2021 年：支持 Uber 500PB 数据湖，SQL DML、Flink 集成、索引、元服务器、缓存。

# Hudi特性
+ 可插拔索引机制支持快速`Upsert/Delete`。
+ 支持增量拉取表变更，以进行处理。
+ 支持事务提交及回滚，并发控制。
+ 支持 Spark、Presto、Trino、Hive、Flink 等引擎的SQL读写。
+ 自动管理小文件，数据聚簇，压缩，清理。
+ 流式摄入，内置 CDC 源和工具。
+ 内置可扩展存储访问的元数据跟踪。
+ 向后兼容的方式实现表结构变更的支持。

# 使用场景
## 近实时写入
+ 减少碎片化工具的使用。
+ CDC 增量导入 RDBMS 数据。
+ 限制小文件的大小和数量。

## 近实时分析
+ 相对于秒级存储（Druid, OpenTSDB），节省资源。
+ 提供<font style="background-color:#FBDE28;">分钟级别时效性</font>，支撑更高效的查询。
+ Hudi 作为 lib，非常轻量。
    - Hive：需要安装包、配置、启动
    - Hudi：通过 Jar 包的方式放到对应的引擎中，如 spark-hudi.jar

## 增量 pipeline
+ 区分 Arrive Time 和 Event Time 处理延迟数据。
+ 更短的调度 Interval，减少端到端延迟（小时 -> 分钟） => Incremental Processing。

## 增量导出
+ 替代部分 Kafka 的场景，数据导出到在线服务存储 e.g. ES。

# <font style="color:rgb(102, 102, 102) !important;">典型特点和场景</font>
1. <font style="color:rgb(51, 51, 51);">支持对海量数据的高效 </font>`<font style="color:rgb(51, 51, 51);">Insert</font>`<font style="color:rgb(51, 51, 51);">/</font>`<font style="color:rgb(51, 51, 51);">Overwrite</font>`<font style="color:rgb(51, 51, 51);"> 能力，以及完全兼容 HSQL 的 OLAP 能力。</font>
    1. <font style="color:rgb(51, 51, 51);background-color:#FBDE28;">典型场景：当前 Hive 体系下构建离线数仓的全部场景。</font>
2. <font style="color:rgb(51, 51, 51);">支持对历史数据的 </font>`<font style="color:rgb(51, 51, 51);">Update</font>`<font style="color:rgb(51, 51, 51);">/</font>`<font style="color:rgb(51, 51, 51);">Delete</font>`<font style="color:rgb(51, 51, 51);"> 能力</font>
    1. <font style="color:rgb(51, 51, 51);">典型场景：历史数据修正；</font>
3. <font style="color:rgb(51, 51, 51);">支持对新增数据的 </font>`<font style="color:rgb(51, 51, 51);">Upsert</font>`<font style="color:rgb(51, 51, 51);">（基于主键）和 </font>`<font style="color:rgb(51, 51, 51);">Append</font>`<font style="color:rgb(51, 51, 51);">（非主键）能力。</font>
    1. <font style="color:rgb(51, 51, 51);">典型场景：事务型数据导入、日志型数据导入。</font>
4. <font style="color:rgb(51, 51, 51);">支持 Streaming Source/Sink，提供</font><font style="color:rgb(51, 51, 51);background-color:#E8F7CF;">近实时分析能力（分钟级实效性）</font><font style="color:rgb(51, 51, 51);">，同时支持接入数据集看板。</font>
    1. <font style="color:rgb(51, 51, 51);">典型场景：实时数据集</font>

<font style="color:rgb(51, 51, 51);"></font>

<font style="color:#DF2A3F;">如果需求不是严格的实时场景，不用使用 Flink，Flink 成本很高</font>

<font style="color:#DF2A3F;">近实时的场景可以选择 hudi 表</font>

<font style="color:#DF2A3F;"></font>

<font style="background-color:#E8F7CF;">Hudi 在生产环境中的选择，主要是</font>

+ <font style="background-color:#E8F7CF;">表类型</font>
+ <font style="background-color:#E8F7CF;">查询方式</font>

<font style="background-color:#E8F7CF;">的选择</font>

# <font style="color:rgb(102, 102, 102) !important;">核心概念</font>
## <font style="color:rgb(102, 102, 102) !important;">表类型</font>
+ COPY_ON_WRITE<font style="color:rgb(51, 51, 51);"> </font><font style="color:rgb(51, 51, 51);">（COW）</font>
    - <font style="color:rgb(51, 51, 51);">场景：更适合离线场景（写少读多）</font>
    - <font style="color:rgb(51, 51, 51);">存储：列存（Parquet）</font>
    - <font style="color:rgb(51, 51, 51);">写入：每次写入数据，会先读取已有数据文件，然后与更新数据合并后写入新的文件。</font>
    - <font style="color:rgb(51, 51, 51);">查询：支持与 Hive 表类似的 HSQL 查询</font>
+ MERGE_ON_READ<font style="color:rgb(51, 51, 51);"> </font><font style="color:rgb(51, 51, 51);">（MOR）：</font>
    - <font style="color:rgb(51, 51, 51);">场景：更适合近实时/实时场景（写多读少）</font>
    - <font style="color:rgb(51, 51, 51);">存储：行存（Delta Log）+ 列存（Base File）</font>
    - <font style="color:rgb(51, 51, 51);">写入：每次写入数据，更新数据会先写入行存（Delta Log），写入指定次数后，行存（Delta Log）文件会与列存（Base File）文件进行合并生成新的列存（Base File）文件，即 Compaction 过程。</font>
    - <font style="color:rgb(51, 51, 51);">查询：</font>
        * <font style="color:rgb(51, 51, 51);">RealTime（RT）查询：列存（Base File）+ 读取行存（Delta Log），数据延迟低</font>
        * <font style="color:rgb(51, 51, 51);">ReadOptimized（RO）查询：仅读取【列存（Base File）数据】，查询性能高</font>
        * <font style="color:rgb(51, 51, 51);">Incremental 查询：适用于增量消费场景</font>

|  | COW | MOR |
| --- | --- | --- |
| 数据延迟 | 高 | 低 |
| 更新成本 | 高（重写整个 Parquet） | 低（Append 到 Delta Log 文件） |
| 写放大 | 高 | 低 |
| RT查询性能 | 高（仅Parquet） | 低（Parquet + Delta Log） |


## <font style="color:rgb(102, 102, 102) !important;">索引类型</font>
+ **<font style="color:rgb(51, 51, 51);">BUCKET_INDEX</font>**
    - <font style="color:rgb(51, 51, 51);">场景：有主键</font>
    - <font style="color:rgb(51, 51, 51);">说明：数据会基于主键进行去重（Upsert），支持 CRUD 能力</font>
+ **<font style="color:rgb(51, 51, 51);">NON_INDEX</font>**
    - <font style="color:rgb(51, 51, 51);">场景：无主键</font>
    - <font style="color:rgb(51, 51, 51);">说明：数据不会基于主键进行去重（Insert），支持高效的 Append 能力</font>
