# 数据处理语义
一般来说，流处理引擎通常为用户的应用程序提供三种数据处理语义：最多一次、至少一次、精确一次。

+ **最多一次（At-Most-Once）**：用户的数据只会被处理一次，不管成功还是失败，不会重试也不会重发。
+ **至少一次（At-Least-Once）**：保证数据或事件至少被处理一次。如果中间发生错误或者丢失，那么会从源头重新发送一条然后进入处理系统，所以同一个事件或者消息可能会被处理多次。
+ **精确一次（Exactly-Once）**：表示每一条数据只会被精确地处理一次，不多也不少。



**端到端（End to End）的精确一次**

+ 指的是 Flink 应用从 Source 端开始到 Sink 端结束，数据必须经过的起始点和结束点
+ Flink 自身是无法保证外部系统“精确一次”语义的

Flink 若要实现所谓“端到端（End to End）的精确一次”的要求，那么

+ **外部系统必须支持“精确一次”语义**；
+ 同时，借助 Flink 提供的**分布式快照**和**两阶段提交**才能实现

# 分布式快照理论
## 定义
分布式快照（Distributed Snapshot）

+ 是指：在分布式系统中捕获所有参与进程的本地状态，以及它们之间通道（channel）状态的，全局一致状态。
+ 用于记录分布式系统在**某个时间点的全局状态**，这个状态包括每个进程的**本地状态**以及**通道（channel）中的消息状态**。
+ 由于分布式系统的状态分散在不同的进程和节点上，并且这些进程之间通过消息传递进行通信，因此获取一个一致的全局状态并不简单。



类比：给一个正在进行的马拉松比赛拍一张全局全景照片，要求这张照片能精确反映某个瞬间所有选手的位置和状态。但比赛不能停，相机快门速度有限

## Chandy-Lamport算法（经典理论模型）
这是分布式快照的理论基石，Flink的Checkpoint机制直接受其启发。

这个算法由K. Mani Chandy和Leslie Lamport在1985年提出，它可以在**不停止系统运行的情况下**，获取一个一致的全局快照

### 关键概念
+ 标记消息（Marker）：特殊控制消息，用于划分快照边界
+ 进程状态：每个计算节点的内存状态
+ 通道状态：节点之间传输中的消息集合

### 原理及步骤
Chandy-Lamport算法的**核心思想**是：**通过插入标记消息（marker）来划分快照的边界**。算法大致步骤如下：

1. 初始化：任意进程发起快照，记录本地状态，向所有输出通道发送Marker
2. 传播：进程首次收到Marker时：
    1. 记录本地状态
    2. 标记该输入通道为空（开始记录后续消息）
    3. 向所有输出通道发送Marker
3. 终止：所有进程都收到Marker并完成状态记录



当所有进程都完成了本地状态和通道状态的记录，我们就得到了一个一致的全局快照。

这个全局快照对应于一个一致的割集（consistent cut），即快照中每个进程的状态和消息都反映了某个全局一致的状态

## 一致性保证的关键特性
| <font style="color:rgb(15, 17, 21);">特性</font> | <font style="color:rgb(15, 17, 21);">说明</font> |
| --- | --- |
| **<font style="color:rgb(15, 17, 21);">因果一致性</font>** | <font style="color:rgb(15, 17, 21);">如果事件 e 在快照中，则 e 的所有因果前驱事件也必须在快照中</font> |
| **<font style="color:rgb(15, 17, 21);">非侵入性</font>** | <font style="color:rgb(15, 17, 21);">无需停止系统运行即可捕获快照</font> |
| **<font style="color:rgb(15, 17, 21);">异步性</font>** | <font style="color:rgb(15, 17, 21);">各节点可独立记录状态，不需全局时钟同步</font> |


# Flink Checkpoint 机制
## Checkpoint 简介
Flink 的 Checkpoint（检查点）

+ 是基于分布式快照的概念实现的
+ Flink 定期为所有算子的状态创建一致性的快照，并将这些快照持久化存储。这样，在发生故障时，Flink 可以将任务恢复到最近一次成功快照的状态
+ 通常说的“做Checkpoint”就是指触发一次分布式快照，将状态保存下来。



后续源码基于 Flink1.17

## 核心概念：Barrier（数据栅栏/屏障）
### 定义
<img src="https://cdn.nlark.com/yuque/0/2026/png/54426354/1767443499211-ad74ca01-6abf-48df-879f-0d62b4a75bc7.png?x-oss-process=image%2Fformat%2Cwebp" width="1390" title="" crop="0,0,1,1" id="RXHmY" class="ne-image">

Barrier

+ 定义：在数据流中，Barrier是一种特殊的标记，由 Source 节点定期插入到数据流中。
+ 每个 Barrier 带有一个检查点 ID，将数据流划分为两个部分：
    - 属于当前检查点的数据（在Barrier之前）
    - 属于下一个检查点的数据（在Barrier之后）



Barrier 对象

```java
// flink-runtime/src/main/java/org/apache/flink/streaming/runtime/io/checkpointing/CheckpointBarrier.java
public class CheckpointBarrier implements CheckpointBarrierHandler.Event {
    private final long id;                    // Checkpoint ID
    private final long timestamp;             // 创建时间戳
    private final CheckpointOptions options;  // Checkpoint选项
    private final boolean isExactlyOnceMode;  // Exactly-Once标志
}
```

**Barrier 携带信息**

+ Checkpoint ID
+ 是否是 Exactly-Once 的标志
+ 创建时间
+ Checkpoint 选项

### Barrier 工作原理
Barrier 的工作流程：Source 算子周期性地注入 Barrier，Barrier 在流经整个 DAG 图的过程中，会触发每个算子进行状态快照。

+ Flink的检查点协调者（JobManager）会触发检查点的开始
+ Source节点接收到触发指令后，会生成一个Barrier，并将其插入到数据流中
+ 然后，Barrier会随着数据流向下游传递。

### Barrier 插入时机
```java
// flink-streaming-java/src/main/java/org/apache/flink/streaming/api/functions/source/SourceFunction.java
// Source算子周期性地注入Barrier
public interface SourceFunction<T> {
    // JobManager通过RPC触发Source创建Barrier
    // CheckpointCoordinator → TaskManager → SourceTask
}
```

### Barrier 的处理
Barrier 的传播路径

```latex
Source1 → Map → KeyBy → Window → Sink
  │         │       │        │      │
Barrier→ Barrier→ Barrier→ Barrier→ Barrier
  │         │       │        │      │
  ↓         ↓       ↓        ↓      ↓
 JobManager收集所有Task的确认
```

`CheckpointBarrierHandler`，负责处理 Barrier，有两种实现：

+ `BarrierBuffer`：用于 Exactly-Once 语义，缓存快流数据
+ `BarrierTracker`：用于 At-Least-Once 语义，不缓存数据

## 协调者（Checkpoint Coordinator）
协调者（Checkpoint Coordinator）：JobManager 中的一个组件

职责包括

1. 发起快照：定期（如每 10 秒）向所有 Source 任务发送指令：“现在开始做快照 N，请注入 Barrier N”。
2. 收集确认：等待所有任务（最终是所有的 Sink 任务）报告“快照 N 已完成”。
3. 最终确认：收到所有确认后，将本次快照标记为全局完成，并（可选）通知各任务可以清理旧的快照。

# Flink Checkpoint 工作流程
## Exactly-Once 语义
**Exactly-Once 语义处理关键点：****<font style="background-color:#FBDE28;">多流对齐，快流等慢流，快流缓存数据</font>**

### 场景假设
双流输入，算子链如下图

```latex
假设拓扑结构：
SourceA -> \
              Union -> Map -> KeyBy -> Reduce -> Sink
SourceB -> /
```

数据流顺序

```latex
时间轴:    t1    t2    t3    t4               t5    t6    t7              t8
数据流:    [ea1] [eb1] [ea2] [BarrierA(id=1)] [eb2] [ea3] [BarrierB(id=1)] [eb3]
输入通道:    A     B     A       A             B     A        B             B
```

### 算子行为分析
1. `SourceA`和`SourceB`分别产生数据，并在适当的时候注入屏障（Barrier）
2. `Union`算子：合并数据，将数据转发给下游
3. `Map`算子：接收数据，处理数据，并处理屏障（Barrier）
4. `KeyBy`算子：分区，将数据发送到`Reduce`算子，不做状态快照
5. `Reduce`算子：接收数据，处理数据，并处理屏障（Barrier），做快照
6. `Sink`算子：有状态，做快照

当所有的算子都完成了快照，并且将快照句柄报告给JobManager后，JobManager就得到了一个完整的分布式快照（检查点1）。

#### Map 算子处理数据时间线分析
<img src="https://cdn.nlark.com/yuque/0/2026/png/54426354/1767450951133-0264c011-767f-4eae-b72d-cbb3ef1d195e.png" width="1122" title="" crop="0,0,1,1" id="z4dDp" class="ne-image">

1. 收到`ea1`：处理`ea1`，输出`transformed(ea1)`，发送给`KeyBy`算子。
2. 收到`eb1`：处理`eb1`，输出`transformed(eb1)`，发送给`KeyBy`算子。
3. 收到`ea2`：处理`ea2`，输出`transformed(ea2)`，发送给`KeyBy`算子。
4. 收到`barrierA(id=1)`：
    1. `Map`算子开始对齐检查点1，等待来自通道B的屏障（id=1）
    2. <font style="background-color:#FBDE28;">暂停处理来自通道 A 的后续数据（即</font>`<font style="background-color:#FBDE28;">ea3</font>`<font style="background-color:#FBDE28;">会被缓冲）</font>
    3. 在收到所有输入的屏障之前，它不会将屏障发送到下游
5. 收到`eb2`：处理`eb2`，输出`transformed(eb2)`，发送给`KeyBy`算子。
    1. 因为通道B的屏障还没有到达，所以eb2是屏障之前的数据，会被正常处理
6. 收到`ea3`：<font style="background-color:#FBDE28;">被缓存，不处理</font>
    1. 因为`barrierA`已经到达，所以来自通道 A 的屏障之后的数据`ea3`会被缓冲，不处理。
7. 收到`barrierB(id=1)`：
    1. Map算子收到来自通道B的屏障，现在，它已经收到了所有输入通道的屏障（id=1），对齐完成。
    2. Map算子会将屏障（id=1）发送到下游`KeyBy`，<font style="background-color:#FBDE28;">意味着所有属于快照1的数据已经处理完了</font>
    3. 然后，它会处理缓冲的数据（ea3），并继续处理后续数据。
8. 收到`eb3`：处理`eb3`，输出`transformed(eb3)`，发送给`KeyBy`算子。
    1. 此时屏障已经发送，eb3是屏障之后的数据，正常处理。

#### KeyBy 算子处理分析
`KeyBy`算子：

+ 分区算子
+ 本身不维护状态，所以它不会做状态快照，只是将**屏障转发给下游的**`**Reduce**`**算子**。

`KeyBy`算子需要将屏障发送到所有的下游任务

+ 根据`key`分组，同一个`key`的数据发送到同一个下游任务
+ 但<font style="background-color:#FBDE28;">屏障需要广播到所有下游任务，因为屏障不参与计算，只是用来触发检查点</font>

#### Reduce 算子处理分析
`Reduce`算子：

+ 有状态算子
+ 当它收到屏障时，需要对齐，也会做快照。

#### Sink 算子处理分析
`Sink`算子

+ 收到屏障后，会做状态快照
+ 如果是精确一次，可能还需要预提交事务，比如Kafka Sink的两阶段提交

### 算子快照内容分析
+ `Source`算子
    - Source A：当前读取的位置，如：Kafka的`offset`
        * 对于检查点1，它可能记录了读到`ea2`之后的位置，因为`ea2`是屏障之前最后一个元素，屏障是在`ea2`之后注入的
    - Source B：类似，快照内容为当前读取的位置，即`eb2`之后
+ `Map`算子：
    - 如果`Map`算子有状态，则快照内容为当前状态，比如通过`MapFunction`维护的状态
    - 注意：在快照时，状态包括直到屏障之前的所有数据（`ea1, eb1, ea2, eb2`）所导致的状态更新。而`ea3`和`eb3`是在屏障之后，不会包括在本次快照中。
+ `Reduce`算子：
    - 快照内容为当前的状态，即直到屏障之前的所有数据所导致的聚合状态。包括每个`key`的当前值、Key Group 的划分信息、序列化的状态数据等
+ `Sink`算子：包括
    - 已写入外部系统的状态，比如 Kafka 事务的 ID，或者缓冲的数据等
    - 如果是两阶段提交，那么快照会包括预提交的事务状态。

### 全局一致快照
全局一致的快照：

+ 由`JobManager`收集所有算子（任务）的快照句柄（指向持久化状态的指针）以及一些元数据
+ 最终存储到状态后端
+ 当需要恢复时，`JobManager`将快照句柄分发给每个算子，算子从持久化存储中恢复状态。



全局快照包括

+ Checkpoint ID
+ 所有算子状态句柄，包括
    - 算子 ID
    - 状态大小
    - 状态数据位置（文件路径 + 偏移量）
    - 检查点 ID
+ 元数据（时间戳、配置等）



检查点1的全局一致性状态包含：

+ `Source`算子
    - `SourceA`：读取到ea2（包含）为止
    - `SourceB`: 读取到eb2（包含）为止
+ `Map`算子: 
    - 处理了`ea1, eb1, ea2, eb2`
    - 状态：`count=4`
    - 未处理：`ea3`（缓冲中）, `eb3`（未到达）
+ `Reduce`算子：处理了基于上述数据的聚合结果
+ `Sink`算子：写入了上述数据处理后的结果

关键：所有算子对检查点1的快照都基于相同的数据集合，即`BarrierA`和`BarrierB`之前的所有数据

### 故障恢复
从 Checkpoint 恢复流程

+ 获取 Checkpoint 状态
+ 从状态句柄读取状态数据，为每个算子恢复状态
    - `Source`：重置到记录的`offset`
    - `Map`：恢复`count=4`的状态
    - `Reduce`：恢复聚合状态
    - `Sink`：恢复待提交的事务
+ 从`Source`开始重放数据

## At-Least-Once 语义
关键点：At-Least-Once 语义下，<font style="background-color:#FBDE28;">会进行 Barrier 对齐，但是不会在等待对齐时缓存数据</font>

### 场景假设
双流输入，算子链如下图

```latex
假设拓扑结构：
SourceA -> \
              Union -> Map -> KeyBy -> Reduce -> Sink
SourceB -> /
```

数据流顺序

```latex
时间轴:    t1    t2    t3    t4               t5    t6    t7              t8
数据流:    [ea1] [eb1] [ea2] [BarrierA(id=1)] [eb2] [ea3] [BarrierB(id=1)] [eb3]
输入通道:    A     B     A       A             B     A        B             B
```

### 算子行为分析
1. `SourceA`和`SourceB`分别产生数据，并在适当的时候注入屏障（Barrier）
2. `Union`算子：合并数据，将数据转发给下游
3. `Map`算子：接收数据，处理数据，并处理屏障（Barrier）
    - 当收到第一个Barrier（barrierA）时，<font style="background-color:#FBDE28;">不触发快照</font>，因为 Barrier 没有全都到达
    - 继续处理后续数据，<font style="background-color:#FBDE28;">但不会缓存</font>`<font style="background-color:#FBDE28;">SourceA</font>`<font style="background-color:#FBDE28;">的数据</font>
4. `KeyBy`算子：分区，将数据发送到`Reduce`算子，不做状态快照
5. `Reduce`算子：接收数据，处理数据，并处理屏障（Barrier），做快照
6. `Sink`算子：有状态，做快照



**Map 算子处理时间线分析**

1. 收到`ea1`：处理`ea1`，输出`transformed(ea1)`，发送给`KeyBy`算子。
2. 收到`eb1`：处理`eb1`，输出`transformed(eb1)`，发送给`KeyBy`算子。
3. 收到`ea2`：处理`ea2`，输出`transformed(ea2)`，发送给`KeyBy`算子。
4. 收到`barrierA(id=1)`：
    1. <font style="background-color:#FBDE28;">不触发快照，因为 Barrier 没有全都到达</font>
    2. 继续处理后续数据
5. 收到`eb2`：处理`eb2`，输出`transformed(eb2)`，发送给`KeyBy`算子。
6. 收到`ea3`：处理`ea3`，输出`transformed(ea3)`，发送给`KeyBy`算子。<font style="background-color:#FBDE28;">不会被缓存</font>
7. 收到`barrierB(id=1)`：
    1. <font style="background-color:#FBDE28;">Barrier 全部到达，触发快照</font>
8. 收到`eb3`：处理`eb3`，输出`transformed(eb3)`，发送给`KeyBy`算子。



在 At-Least-Once下，快照触发后，不会阻塞后续数据，所以数据会继续被处理，因此`ea3`、`eb2`、`eb3`等都会在快照之后被处理，这就意味着如果故障恢复，这些数据可能会被重复处理。

### 全局快照内容分析
检查点1的全局状态可能包含：

+ `SourceA`状态：读取到`ea2`（包含）
+ `SourceB`状态：读取到`eb2`（包含）
+ `Map`状态：
    - 已经处理了（`ea1, eb1, ea2, eb2, ea3`）状态计数为 5
+ `Reduce`状态
    - 聚合了（`ea1, eb1, ea2, eb2, ea3`）
+ `Sink`状态: 已经写入了`ea1, eb1, ea2, eb2, ea3`

### 故障恢复
从检查点1进行恢复

故障前处理情况

+ 已处理：`ea1, eb1, ea2, eb2, ea3`
+ Map状态：`counter=5`
+ 检查点保存：`counter=5`



恢复后重新处理

+ `SourceA`重放：从`ea3`开始，`ea3`被重复处理
+ `SourceB`重放：从`eb3`开始

## Exactly-Once 和 At-Least-Once 的区别
Exactly-Once 和 At-Least-Once 实际区别在于：<font style="background-color:#FBDE28;">数据是否在 Barrier 对齐期间被缓存</font>

+ 两者都等待所有的 Barrier 到达才触发检查点



Exactly-Once (`BarrierAligner`):

1. 收到第一个Barrier后，缓冲该通道后续数据
2. 等待其他通道的Barrier
3. 所有Barrier到齐后，触发检查点
4. 转发Barrier，释放缓冲数据



At-Least-Once (`BarrierTracker`):

1. 收到 Barrier，记录但不立即触发
2. 继续处理所有数据（不缓冲）
3. 所有 Barrier 到齐后，触发检查点
4. 可能已经处理了 Barrier 之后的数据

# 两阶段提交协议与 Exactly-Once 语义
## 两阶段提交协议（2PC）概述
两阶段提交协议：Two Phase Commit

+ 是一种**分布式一致性协议**
+ 用于确保分布式系统中所有参与者对事务的提交或中止达成一致决定



在Flink中，2PC被用于确保在检查点完成时，所有算子对外部系统的写入操作要么全部提交，要么全部回滚，从而实现端到端的 Exactly-Once 语义。



协议的两个阶段

**阶段1：准备阶段（投票阶段）**

1. 协调者询问所有参与者是否可以提交
2. 参与者执行事务操作，但不提交
3. 参与者回复"是"或"否"

**阶段2：提交阶段（执行阶段）**

1. 如果所有参与者都同意，协调者发送提交指令
2. 参与者正式提交事务
3. 参与者回复确认



**2PC 的 ACID 数据保证**

+ 原子性（Atomicity）：
    - 所有参与者要么都提交，要么都回滚
    - <font style="background-color:#FBDE28;">这是 2PC 最核心的保证</font>
+ 一致性（Consistency）：
    - 事务从一个一致状态转换到另一个一致状态
+ 隔离性（Isolation）：
    - 2PC本身不直接保证，需要底层系统支持
+ 持久性（Durability）：
    - 一旦提交，更改永久保存

## 2PC 在 Flink 中的实现
两阶段搭配特定的 source 和 sink（特别是 0.11 版本 Kafka）使得“精确一次处理语义”成为可能

### Flink 中 2PC 概述
在 Flink 中两阶段提交的实现方法被封装到了 `TwoPhaseCommitSinkFunction` 这个抽象类中，我们只需要实现其中的`beginTransaction`、`preCommit`、`commit`、`abort` 四个方法就可以实现“精确一次”的处理语义，实现的方式我们可以在官网中查到：

+ `beginTransaction`：在开启事务之前，我们在目标文件系统的临时目录中创建一个临时文件，后面在处理数据时将数据写入此文件；
+ `preCommit`：在预提交阶段，刷写（flush）文件，然后关闭文件，之后就不能写入到文件了，我们还将为属于下一个检查点的任何后续写入启动新事务；
+ `commit`：在提交阶段，我们将预提交的文件原子性移动到真正的目标目录中，请注意，这会增加输出数据可见性的延迟；
+ `abort`：在中止阶段，我们删除临时文件。

### 在 Exactly-Once 语义下的具体实现
文件 Sink

```java
// flink-connectors/flink-connector-files/src/main/java/org/apache/flink/connector/file/sink/
public class FileSink<IN> implements TwoPhaseCommittingSink<IN, FileSinkCommittable> {
    
    // 阶段1：预提交（对应snapshotState）
    public List<FileSinkCommittable> prepareCommit() {
        // 1. 刷新所有缓冲区数据
        // 2. 将文件从".in-progress"重命名为".pending"
        // 3. 返回待提交的信息（不实际提交）
        // 此时数据对用户不可见
    }
    
    // 阶段2：提交（对应notifyCheckpointComplete）
    public void commit(List<FileSinkCommittable> committables) {
        // 将".pending"文件重命名为最终文件名
        // 此时数据对用户可见
    }
}
```



Kafka Sink

```java
// flink-connectors/flink-connector-kafka/src/main/java/org/apache/flink/connector/kafka/sink/
public class KafkaWriter<IN> {
    
    private KafkaProducer<byte[], byte[]> producer;
    
    public List<KafkaCommittable> prepareCommit(boolean flush) {
        // 阶段1：预提交
        // 1. 刷新生产者缓冲区
        producer.flush();
        
        // 2. 提交当前事务（但不使数据对消费者可见）
        // 在Kafka中，commitTransaction()会使数据对消费者可见
        // 所以Flink Kafka Sink的实现有所不同
        // 实际上，Kafka的2PC是特殊的：
        //   - 写入数据到事务
        //   - 事务保持打开状态
        //   - 在检查点完成时提交事务
        
        // 3. 开始新事务
        String newTransactionId = generateTransactionId(nextCheckpointId);
        producer.beginTransaction();
        
        return Collections.singletonList(
            new KafkaCommittable(producer, transactionId)
        );
    }
    
    public void commit(List<KafkaCommittable> committables) {
        // 阶段2：提交
        // 实际在Kafka中，提交是在协调阶段完成的
        // 这里的commit可能只是确认操作
    }
}
```



数据库 Sink

```java
public class JdbcSink<IN> {
    
    private Connection connection;
    private String transactionId;
    
    public List<JdbcCommittable> prepareCommit(boolean flush) {
        // 阶段1：预提交
        // 1. 执行所有INSERT/UPDATE语句
        // 2. 但不执行connection.commit()
        // 3. 返回事务ID和连接信息
        
        return Collections.singletonList(
            new JdbcCommittable(connection, transactionId)
        );
    }
    
    public void commit(List<JdbcCommittable> committables) {
        // 阶段2：提交
        for (JdbcCommittable committable : committables) {
            Connection conn = committable.getConnection();
            try {
                conn.commit();  // 正式提交事务
            } catch (SQLException e) {
                conn.rollback();  // 提交失败则回滚
                throw new RuntimeException("Commit failed", e);
            }
        }
    }
}
```

## Exactly-Once 语义与两阶段提交
### Sink 算子内部结构
```java
// flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/sink/
// StreamSinkOperator是执行Sink逻辑的算子
public class StreamSinkOperator<InputT> extends AbstractStreamOperator<Object>
    implements OneInputStreamOperator<InputT, Object> {
    
    private transient SinkWriter<InputT> sinkWriter;
    private transient Committer<?> committer;
    private transient ListState<byte[]> committerState;  // 存储待提交事务
    
    // 处理元素的核心方法
    public void processElement(StreamRecord<InputT> element) throws Exception {
        // 直接将数据写入SinkWriter
        sinkWriter.write(element.getValue(), context);
    }
}
```

主要分为 3 个部分

+ `SinkWriter`
+ `Committer`
+ `ListState`：存储待提交的事务

### Sink 算子与两阶段提交
Sink 算子在 Barrier 对齐后，会进行以下两个操作

1. Sink 算子快照：包括了 2PC 的第一阶段（预提交）
    1. Flink 实现：Sink 算子的`snapshotState()`方法
2. 全局快照完成：JobManager通知所有算子检查点完成，Sink 算子此时执行第二阶段（提交）
    1. Flink 实现：Sink 算子的`notifyCheckpointComplete()`方法



```java
// flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/sink/
public class StreamSinkOperator<InputT> extends AbstractStreamOperator<Object> {
    
    // ========== 阶段1: 在快照中预提交 ==========
    @Override
    public void snapshotState(StateSnapshotContext context) throws Exception {
        // 1. 准备提交(第一阶段)
        List<Committable> committables = sinkWriter.prepareCommit(false);
        
        // 2. 保存预提交信息到状态
        // 这相当于2PC中参与者保存"同意提交"的投票
        pendingCommits.add(serializeCommittables(committables));
        
        // 3. 重要：此时数据在外部系统处于"预提交"状态
        // 对于文件系统：文件在临时位置，用户不可见
        // 对于Kafka：事务已提交但消费者不可见(事务隔离)
    }
    
    // ========== 阶段2: 在通知中完成提交 ==========
    @Override
    public void notifyCheckpointComplete(long checkpointId) throws Exception {
        // 1. 只有当全局快照完成后才执行这里
        // 2. 从状态中读取预提交信息
        List<Committable> committables = getPendingCommits(checkpointId);
        
        // 3. 执行提交(第二阶段)
        committer.commit(committables);
        
        // 现在数据在外部系统真正可见
    }
}
```

### 举例
流输入，算子链如下图

```latex
假设拓扑结构：
SourceA -> \
              Union -> Sink
SourceB -> /
```

数据流顺序

```latex
时间轴:    t1    t2    t3    t4               t5    t6    t7              t8
数据流:    [ea1] [eb1] [ea2] [BarrierA(id=1)] [eb2] [ea3] [BarrierB(id=1)] [eb3]
输入通道:    A     B     A       A             B     A        B             B
```



`Sink`算子处理时间线：

+ t1-t3：`Sink`处理`ea1, eb1, ea2`，写入到缓冲区或外部系统临时位置
+ t4：收到 BarrierA，开始对齐
+ t5：`Sink`处理`eb2`，写入到缓冲区或外部系统临时位置
+ t6：`Sink`不处理`ea3`，`ea3`被缓存
+ t7：收到 BarrierB，对齐完成
    - 触发检查点1 的快照
    - `Sink.snapshotState()`被调用
    - `<font style="background-color:#FBDE28;">sinkWriter.prepareCommit()</font>`<font style="background-color:#FBDE28;">执行，预提交</font>
    - 此时：`ea1, eb1, ea2, eb2`被预提交，预提交信息保存到算子状态
    - 释放缓冲数据`ea3`，继续处理
+ t7之后：JobManager等待所有算子确认快照，当所有算子都确认后，检查点1标记为完成
    - `JobManager`发送`notifyCheckpointComplete(1)`
    - `Sink`收到通知，<font style="background-color:#FBDE28;">执行</font>`<font style="background-color:#FBDE28;">commit()</font>`<font style="background-color:#FBDE28;">  第二阶段提交</font>
    - `ea1, eb1, ea2, eb2`对外可见

# At-Least-Once 语义下的 Sink 算子行为
## At-Least-Once 语义下无需两阶段提交
At-Least-Once 语义下，不需要两阶段提交，Sink 算子的行为：

+ Sink 通常直接写入外部系统
+ Sink 不需要实现`TwoPhaseCommittingSink`接口，没有`prepareCommit`和`commit`分离
+ 数据直接写入，立即可见

## Sink 算子的快照实现
```java
// 在AT-LEAST-ONCE下，Sink的快照通常很简单
public class SimpleSinkOperator<IN> extends AbstractStreamOperator<Object> {
    
    @Override
    public void snapshotState(StateSnapshotContext context) throws Exception {
        // 1. 刷新缓冲区，确保数据写入外部系统
        // 2. 记录写入位置（如文件偏移量、Kafka offset等）
        // 3. 没有prepareCommit阶段，数据已经是可见的！
        
        // 对于文件Sink，可能只是记录当前写入的文件和位置
        // 对于数据库Sink，可能只是记录最后写入的ID或时间戳
        
        // 这里没有"预提交"的概念，数据已经写入了
    }
}
```

## 举例
```latex
数据流顺序：ea1, eb1, ea2, barrierA(id=1), eb2, ea3, barrierB(id=1), eb3

通道分配：
- 通道0（A流）：ea1, ea2, barrierA, ea3
- 通道1（B流）：eb1, eb2, barrierB, eb3

Sink 时间线（AT-LEAST-ONCE）：
t1: 收到ea1（通道0） → 立即写入外部系统（可见）
t2: 收到eb1（通道1） → 立即写入外部系统（可见）
t3: 收到ea2（通道0） → 立即写入外部系统（可见）
t4: 收到barrierA（通道0） → 记录通道0收到Barrier，但继续处理其他通道
t5: 收到eb2（通道1） → 立即写入外部系统（可见）
t6: 收到ea3（通道0） → 立即写入外部系统（可见）
    // 注意：在EXACTLY-ONCE下ea3会被缓冲，但这里直接处理了！
t7: 收到barrierB（通道1） → 所有Barrier到齐
    → 触发检查点1快照
    → Sink.snapshotState()被调用
    → 记录当前写入位置
t8: 收到eb3（通道1） → 立即写入外部系统（可见）
```

# Checkpoint 参数配置
开启检查点

默认情况下，检查点机制是关闭的，需要在程序中进行开启：

```java
// 开启检查点机制，并指定状态检查点之间的时间间隔
env.enableCheckpointing(1000); 

// 其他可选配置如下：
// 设置语义
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 设置两个检查点之间的最小时间间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
// 设置执行Checkpoint操作时的超时时间
env.getCheckpointConfig().setCheckpointTimeout(60000);
// 设置最大并发执行的检查点的数量
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
// 将检查点持久化到外部存储
env.getCheckpointConfig().enableExternalizedCheckpoints(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
// 如果有更近的保存点时，是否将作业回退到该检查点
env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
```

# 优化
## 异步存储
异步存储（Asynchronous Snapshotting）：指在进行状态快照时，不阻塞数据处理的进行，将状态持久化的工作放到后台异步执行



工作原理：

+ RocksDB状态后端的异步快照
    - 检查点触发时：RocksDB创建一个一致的数据库视图
    - 异步复制文件：后台异步复制SST文件到检查点目录
    - 继续处理：主线程继续写入新的MemTable，不影响处理性能
+ 内存状态后端的异步快照
    - 写时复制技术



优点

+ 高吞吐：数据处理不中断
+ 低延迟：Barrier传播不受存储操作影响
+ 资源利用率高：CPU和I/O操作可以并行

## 增量存储
增量存储（Incremental Checkpointing）：指只保存自上一次检查点以来发生变化的状态部分，而不是每次保存完整状态

+ 优点
    - 存储空间大幅减少：通常减少50%-90%
    - 检查点速度更快：复制文件更少
    - 网络传输减少：只传输变化部分
    - 对大型状态友好：特别适合TB级状态
+ 挑战
    - 依赖链管理：每个增量检查点都依赖于之前的检查点
    - 清理策略：需要智能清理过期检查点，同时保留依赖链
    - 恢复复杂度：恢复时需要重建完整的文件集



完整存储的问题：假设一个作业有100GB的状态

+ 每次检查点都要保存100GB
+ 网络传输100GB
+ 存储占用：N个检查点 × 100GB



增量存储工作原理

+ RocksDB的增量检查点
    - RocksDB使用LSM树结构，天然适合增量存储

# 保存点机制
保存点机制（Savepoints）是检查点机制的一种特殊的实现，它允许通过手工的方式来触发 Checkpoint，并将结果持久化存储到指定路径中，主要用于避免 Flink 集群在重启或升级时导致状态丢失。示例如下：

```shell
# 触发指定id的作业的Savepoint，并将结果存储到指定目录下
bin/flink savepoint :jobId [:targetDirectory]
```

更多命令和配置可以参考官方文档：[savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.9/zh/ops/state/savepoints.html)

## Checkpoint vs Save Point
| <font style="color:rgb(15, 17, 21);">特性</font> | <font style="color:rgb(15, 17, 21);">Checkpoint（检查点）</font> | <font style="color:rgb(15, 17, 21);">Savepoint（保存点）</font> |
| --- | --- | --- |
| **<font style="color:rgb(15, 17, 21);">目的</font>** | <font style="color:rgb(15, 17, 21);">容错恢复</font> | <font style="color:rgb(15, 17, 21);">计划维护、版本升级</font> |
| **<font style="color:rgb(15, 17, 21);">触发</font>** | <font style="color:rgb(15, 17, 21);">自动、定期</font> | <font style="color:rgb(15, 17, 21);">手动、按需</font> |
| **<font style="color:rgb(15, 17, 21);">生命周期</font>** | <font style="color:rgb(15, 17, 21);">临时，可被清理</font> | <font style="color:rgb(15, 17, 21);">持久，直到手动删除</font> |
| **<font style="color:rgb(15, 17, 21);">性能影响</font>** | <font style="color:rgb(15, 17, 21);">优化过，影响小</font> | <font style="color:rgb(15, 17, 21);">可能影响性能</font> |
| **<font style="color:rgb(15, 17, 21);">状态格式</font>** | <font style="color:rgb(15, 17, 21);">可能内部格式</font> | <font style="color:rgb(15, 17, 21);">统一标准化格式</font> |
| **<font style="color:rgb(15, 17, 21);">恢复灵活性</font>** | <font style="color:rgb(15, 17, 21);">同一作业恢复</font> | <font style="color:rgb(15, 17, 21);">可恢复为不同作业</font> |


# 
