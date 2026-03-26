不仅要懂 Doris 本身，更要理解它在整个数据架构中的位置、适用边界和替代方案，从而在复杂业务场景中做出合理的技术选型。

# Doris 简介
## 定义
Apache Doris 是一款

+ 基于 MPP 架构的高性能、实时分析型数据库（读多写少）。
+ 以高效、简单和统一的特性著称
+ Doris 既能支持高并发的点查询场景，也能支持高吞吐的复杂分析场景。

## Doris 的架构和组件
Apache Doris 采用 MySQL 协议，高度兼容 MySQL 语法，支持标准 SQL。

 

Apache Doris 分为两种架构，可以根据硬件环境与业务需求选择部署方式

+ 存算一体架构
+ 存算分离架构。

Doris 集群

+ 主从架构（Master-Slave）
+ 由两类进程组成：Frontend（FE） 和 Backend（BE）。

这两种进程可以水平扩展，混合部署

### 存算一体架构
精简且易于维护，包含以下两种类型的进程：

+ Frontend (FE)： 主要负责接收用户请求、查询解析和规划、元数据管理以及节点管理。
    - FE集群应该独立部署在专属的、配置不需要特别高但稳定性要好的服务器上（例如：4C8G，SSD系统盘）
    - 通过部署多个 FE 节点以实现容灾备份，每个 FE 节点都会维护完整的元数据副本
    - FE 节点分为三种角色：Master，Follower，Observer
+ Backend (BE)： 主要负责数据存储和查询计划的执行。数据会被切分成数据分片（Shard），在 BE 中以多副本方式存储（一般为 3）。
    - 依赖 FE 生成的查询计划，分布式执行查询
    - BE 集群需要独立部署在高配置的服务器上（例如：16C+，64G+，多块SATA/SSD数据盘）

存算一体架构优势：高度集成，大幅降低了分布式系统的运维成本

+ FE 和 BE 进程都可以横向扩展。
+ 单个集群可以支持数百台机器和数十 PB 的存储容量。
+ FE 和 BE 进程通过一致性协议来保证服务的高可用性和数据的高可靠性

<img src="https://cdn.nlark.com/yuque/0/2025/png/54426354/1761968125937-f5478807-4b9e-436a-a83e-5f923f5b065e.png" width="2560" title="" crop="0,0,1,1" id="u6f864d99" class="ne-image">

### 存算分离架构（3.0 版本及以后）
存算分离架构使用统一的**共享存储层**作为数据存储空间，保证存储和计算分离，用户可以独立扩展存储容量和计算资源，从而实现最佳性能和成本效益。

存算分离架构分为以下三层：

+ **元数据层：** 多个 FE 节点构成，负责请求规划、查询解析以及元数据的存储和管理。
+ **计算层：** 由多个计算组组成。每个计算组可以作为一个独立的租户承担业务计算。每个计算组包含多个无状态的 BE 节点，可以随时弹性伸缩 BE 节点。
+ **存储层：** 可以使用 S3、HDFS、OSS、COS、OBS、Minio、Ceph 等共享存储来存放 Doris 的数据文件，包括 Segment 文件和反向索引文件等。

<img src="https://cdn.nlark.com/yuque/0/2025/png/54426354/1761968551236-47f35634-edb2-49d2-9053-a10d7fb1ffb1.png" width="2560" title="" crop="0,0,1,1" id="u823f6176" class="ne-image">

## Doris 特性
+ **高可用**：元数据和数据均采用**多副本存储**，通过 Quorum 协议同步数据**日志**。大多数副本写入完成，认为数据写入成功。
+ **高兼容**：兼容 MySQL 协议，涵盖绝大多数 MySQL 和 Hive 函数
+ **实时数仓**：基于 Doris 构建高性能、低延迟的实时数据仓库服务
    - Doris 提供**秒级数据入库**能力，上游 OLTP 的增量变化秒级捕获到 Doris 中
    - **亚秒级数据查询能力**
+ **湖仓一体**：Doris 可基于外部数据源（如数据湖或关系型数据库）构建湖仓一体架构，从而解决数据在数据湖和数据仓库之间无缝集成和自由流动的问题
+ **灵活建模：** Apache Doris 提供多种建模方式，如宽表模型、预聚合模型、星型/雪花模型等，通过视图、物化视图或实时多表关联等方式进行数据的建模操作。

# Doris 在技术体系中的典型应用场景
Apache Doris（现社区主推 StarRocks，但许多企业仍沿用 Doris 品牌）是一个

+ MPP 架构的高性能、实时、统一分析型数据库
+ 其核心优势在于【高并发点查 + 复杂多表关联 + 实时写入 + 易运维】的平衡。

在大厂中，它通常部署在以下关键场景：

1. 数据中台
2. 实时数仓
3. BI 分析
4. 日志分析

## 数据中台（Data Middle Platform）
角色：作为统一的 OLAP 服务层，承接来自 ODS/DWD/DWS 层的数据，对外提供标准化查询接口。

典型用法：

+ 将 Hive/Spark 清洗后的宽表导入 Doris，供下游应用直接查询；
+ 通过 Routine Load 消费 Kafka 中的 DWD 层日志，构建实时 DWS 表；
+ 提供统一 SQL 接口，屏蔽底层存储差异（如 HDFS vs MySQL vs Kafka）。

价值：避免各业务线重复建设 OLAP 引擎，提升数据复用率与一致性。

📌 示例：美团内部将 Doris 作为“实时数仓统一出口”，支撑数百个 BI 报表和运营看板。

## 实时数仓（Real-time Data Warehouse）
角色：承担 Lambda 架构中的 Speed Layer 或 Kappa 架构的唯一处理层。

关键能力：

+ 支持 秒级延迟 的数据摄入（Stream Load / Routine Load）；
+ 支持 Exactly-Once 语义（通过两阶段提交 + Kafka offset 管理）；
+ 支持 窗口聚合、多流 Join（通过 Flink 预处理 + Doris 落盘）。

典型链路：用户行为日志 → Kafka → Flink（ETL/Join） → Doris → BI / API 查询

优势：相比传统 Hive 数仓 T+1 延迟，Doris 可实现分钟级甚至秒级可见性。

⚠️ 注意：Doris 本身不擅长复杂流计算（如 Session Window），需与 Flink 协同。

## BI 分析（Business Intelligence）
角色：作为 高性能 BI 引擎，直接对接 Tableau、Superset、QuickBI 等工具。

关键特性：

+ 高并发（数千 QPS）支持多用户同时查询；
+ 复杂多表 Join（尤其是 Colocate Join）性能优异；
+ 支持标准 MySQL 协议，BI 工具零改造接入。

对比传统方案：

+ 替代 MySQL 汇总表（避免大表 Join 性能瓶颈）；
+ 替代 Presto/Trino（降低资源消耗，提升稳定性）。

✅ 案例：某电商大促期间，Doris 承载 5000+ QPS 的实时 GMV 监控看板，P99 < 800ms。

## 日志分析（Log Analytics）
角色：用于 结构化日志的即席查询（Ad-hoc Query）。

适用条件：

+ 日志已结构化（JSON/CSV）；
+ 查询模式以【时间范围 + 过滤条件】为主；
+ 需要快速下钻（如“某用户最近 1 小时的所有操作”）。

优势：

+ 列存 + 稀疏索引 + ZoneMap，大幅减少 I/O；
+ 支持 `LIKE`、`REGEXP`、`JSON` 函数（新版本）；
+ 比 Elasticsearch 更节省存储（无倒排索引冗余）。

局限：

+ 不适合全文检索（Elasticsearch 仍是首选）；
+ 写入吞吐低于 ClickHouse（不适合超高频日志采集）。

🔍 场景：APP 埋点日志分析、风控审计日志回溯、客服工单查询。

# 主流 OLAP 引擎横向对比（深度剖析）
| 引擎 | 架构 | 实时写入 | 多表 Join | 高并发点查 | 运维复杂度 | 云原生 | 典型场景 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Doris / StarRocks | MPP + 列存 | ✅<br/>秒级 | ✅✅✅<br/>Colocate/<br/>Broadcast | ✅✅✅ | 低<br/>无依赖 | 部分 | 实时数仓、BI、日志分析 |
| ClickHouse | Shared-Nothing | ✅<br/>但小批量差 | ❌（弱，需子查询） | ✅（单表极快） | 中<br/>需 ZK | 否 | 单表聚合、日志分析 |
| Druid | Lambda + 列存 | ✅<br/>实时+批 | ❌<br/>仅 lookup join | ✅<br/>时间序列 | 高<br/>多组件 | 是 | 时序监控、广告报表 |
| Pinot | Real-time OLAP | ✅ | ❌ | ✅✅ | 高：<br/>需：Kafka<br/>/Helix | 是 | Uber 实时 ETA、LinkedIn Feed |
| Snowflake | 存算分离 | ⚠️<br/>分钟级 | ✅ | ✅ | 极低（SaaS） | ✅✅✅ | 企业级数仓、跨云分析 |
| Redshift | MPP（列存） | ⚠️<br/>COPY 为主 | ✅ | ✅ | 中（需 VACUUM） | ✅ | AWS 生态数仓 |
| TiDB HTAP | TiKV + TiFlash | ✅<br/>强一致 | ✅（但复杂 Join 慢） | ✅（简单查询） | 高 | 部分 | 混合事务/分析（如金融） |


## 关键维度深度解析：
### 多表 Join 能力
+ Doris 是少数原生支持高效多表 Join 的开源 OLAP，尤其 Colocate Join 可避免 Shuffle；
+ ClickHouse/Druid/Pinot 均需“打宽表”预处理，增加 ETL 复杂度；
+ Snowflake/TiDB 虽支持，但成本或性能不如 Doris 平衡。

### 实时写入模型
+ Doris 的 Stream Load 支持 HTTP 小批量写入（KB~MB 级），适合业务系统直写；
+ ClickHouse 对小批量写入极不友好（产生大量小文件）；
+ Druid/Pinot 依赖 Kafka 流式摄入，无法支持随机更新。

### 运维复杂度
+ Doris 仅需 FE + BE 两类进程，无外部依赖（如 ZK、HDFS）；
+ Druid 需 Historical/MiddleManager/Broker/Coordinator/ZK 等 5+ 组件；
+ Pinot 依赖 Helix（ZooKeeper）做集群管理。

### 生态兼容性
+ Doris 兼容 MySQL 协议，现有 BI 工具、ORM 框架可无缝接入；
+ ClickHouse 需专用驱动；
+ Snowflake 虽生态好，但锁定云厂商。

# 如何基于业务需求论证 Doris 的选型合理性？
“为什么选 Doris，而不是其他？” 以下是结构化论证框架：

## 成本维度
+ 硬件成本：Doris 列存压缩比高（通常 5~10x），比行存节省 60%+ 存储；
+ 人力成本：无需专职 DBA（对比 Oracle/Redshift），运维脚本简单；
+ 许可成本：完全开源（Apache 2.0），无商业版绑定（对比 Snowflake 按 TB 扫描计费）。

💡 举例：某公司从 Redshift 迁移至 Doris，年成本下降 70%。

## 性能维度
+ 查询延迟：复杂多表 Join 场景下，Doris 比 Presto 快 3~10 倍；
+ 写入吞吐：Routine Load 可稳定消费 10W+ msg/s 的 Kafka topic；
+ 并发能力：单集群支持 1000+ QPS 的 BI 查询（实测数据）。

## 运维复杂度
+ 部署：3 节点即可搭建高可用集群（1FE + 2BE）；
+ 扩缩容：BE 节点动态加入，自动均衡 Tablet；
+ 升级：滚动升级，业务无感知；
+ 监控：内置 Prometheus 指标，Grafana 模板开箱即用。

## 生态兼容性
+ 协议兼容：MySQL JDBC/ODBC 驱动直接连接；
+ 数据源集成：支持从 Hive/Iceberg/Hudi/MySQL/Kafka 直接同步（Multi-Catalog）；
+ 开发体验：标准 SQL（支持窗口函数、CTE、子查询），学习曲线平缓。

# 典型反例：什么情况下不该选 Doris？
边界：

| 场景 | 原因 | 替代方案 |
| --- | --- | --- |
| 超高频写入（>100W msg/s） | BE 写入瓶颈，Compaction 压力大 | ClickHouse + 物化视图 |
| 全文检索（如日志关键词搜索） | 无倒排索引 | Elasticsearch |
| 强事务 ACID（如银行转账） | 仅支持表级原子写入 | TiDB / PostgreSQL |
| 超大规模离线分析（PB 级） | 存储成本高于 HDFS | Spark on Hive/Iceberg |
| 完全托管、免运维 SaaS | 需自建集群 | Snowflake / BigQuery |


# 总结：Doris 的核心定位
Doris 是一个“<font style="background-color:#E8F7CF;">平衡型</font>”实时 OLAP 引擎：

+ 在写入实时性、查询复杂度、并发能力、运维成本之间取得最佳折衷
+ 特别适合需要“实时 + 复杂分析 + 高并发”的中大型企业场景。

在技术栈中，它往往扮演 “实时数仓统一出口” 或 “高性能 BI 引擎” 的角色，成为连接数据生产与消费的关键枢纽。
