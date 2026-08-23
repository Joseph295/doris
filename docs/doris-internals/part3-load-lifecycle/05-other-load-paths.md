# 第 5 章：其他导入方式 —— 与 Stream Load 基线的差异

前三章跟着一条 Stream Load 走完了整条导入主干：HTTP 接入与开事务（第 2 章）、memtable 攒批刷 rowset（第 3 章）、commit 与 publish 让数据可见（第 4 章）。这条链路是本部分的**基线**。本章讲另外四种导入方式——Broker Load、Routine Load、Insert Into、Group Commit——但**不会再把这条链路重讲一遍**。原因在 5.1 会论证：它们和 Stream Load 共用同一条写入内核，真正的差异只发生在**触发方与数据源接入这一层**。所以本章的组织方式是"按差异点讲"：每种方式只回答三个问题——谁来触发、数据从哪来、这带来了哪些 Stream Load 没有的状态与坑。读的时候请始终把第 2~4 章那条主干当作参照系。

本章的行号引用基于写作时核实所用的 HEAD（`bad283aaa9`，源码树与系列基线 `7bc98f696f` 一致）。代码演进会让行号漂移，但对象名与状态语义不变；写作时每一处 `路径:行号` 都在当前代码里核实过。

## 5.1 问题：一条写入内核，多少种喂数据的姿势

**遇到了什么问题？** 用户手里的数据形态千差万别：本地文件、HDFS 上的 TB 级大文件、Kafka 里源源不断的消息流、另一张表的查询结果、程序里一行一行 `INSERT`。每一种形态的"取数"方式完全不同——有的要连对象存储、有的要维护 Kafka 消费位点、有的本身就是一个 SQL 查询计划。问题是：一个分析型数据库要不要为每种数据源各做一条从头到尾的专用导入链路？

**有哪些候选、各有什么优劣？**

- **候选一：每种数据源一条专用链路。** Kafka 有 Kafka 的写入路径、HDFS 有 HDFS 的、`INSERT` 有 `INSERT` 的，各自实现取数、切分、写 tablet、提交。直觉上"专用即高效"。但代价是灾难性的重复：分桶分发、多副本一致性、事务提交、版本可见——这些**与数据源无关的硬骨头**要被每条链路各啃一遍。任何一个内核改动（比如 quorum 提交语义变了）都要在 N 条链路上同步修改，且极易漏改导致行为不一致。
- **候选二：全部归一到一种内部形式，复用一条写入内核。** 承认一个事实：无论数据从哪来，最终要做的都是同一件事——**把一批行按分桶规则分发到各 tablet 副本、挂在一个批级事务下、commit 后 publish 可见**。那就把这条"内核路径"抽出来复用，各数据源只负责把自己的数据**喂成内核认识的形式**（一个个 Block），再送进同一个 `OlapTableSink`。差异被压缩到最前端的"取数 + 触发"这一薄层。

**Doris 怎么考量和解决的？** Doris 选了候选二，而且贯彻得很彻底。第 2 章已经点破：Stream Load 的写入其实就是一个"scan（读 HTTP 流）→ `OlapTableSink`（写表）"的迷你查询计划，跑在 part2 那套 fragment/pipeline 引擎上。这条"计划即导入"的设计是归一化的支点——**换数据源，无非是换掉计划最前端的 scan 源头**：Broker Load 换成读 HDFS 文件的 scan，Insert Into 干脆整个查询计划就是数据源，Routine Load 换成读 Kafka 的 scan。而计划后半程的分桶、分发、`beginTransaction / commit / publish`，全部复用第 2~4 章那条基线，一个字节都不重写。

所以本章要建立的核心心智是：**万变不离"起事务 → 起计划 → sink → commit"这条内核；四种导入方式的全部差异，都集中在"谁触发这条内核、以什么节奏触发、数据怎么喂进 scan"这一层。** 下面每一节都只讲这一层的差异，以及它衍生出的、Stream Load 没有的状态机与易错点。

## 5.2 Broker Load：FE 编排的异步批作业

**差异定位：触发方从"客户端同步阻塞"变成"FE 侧的常驻异步作业"。** Stream Load 是客户端发一次 HTTP、阻塞等到 `VISIBLE` 才返回；Broker Load 是用户提交一条 SQL 作业后**立即返回**，FE 在后台把它拆成任务、调度执行、驱动事务，用户靠 `SHOW LOAD` 轮询进度。这个"异步作业"身份带来了 Stream Load 完全没有的东西：一台需要持久化、能重放、有独立状态机的 `LoadJob`。

`BrokerLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:85`）继承自 `BulkLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BulkLoadJob.java:73`）再到 `LoadJob`。它的执行被文件头注释（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:80`）明确拆成三步：

1. **Pending 阶段**：`unprotectedExecuteJob`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:142`）创建一个 `BrokerLoadPendingTask`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadPendingTask.java`）丢进 `PendingLoadTaskScheduler`（`fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java:5368`）。这一步去 Broker 侧列文件、拿文件大小，把要导的文件规划成若干 file group。
2. **Loading 阶段**：Pending 完成后 `onTaskFinished`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:162`）回调，走 `createLoadingTask`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:281`）**按表**生成一批 `LoadLoadingTask`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadLoadingTask.java`），提交到 `LoadingLoadTaskScheduler`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java:351`）。每个 loading task 本质就是去某台 BE 上跑一个"scan HDFS 文件 → `OlapTableSink`"的计划——**这里就接回了 5.1 说的内核**。
3. **Commit 阶段**：所有 loading task 完成后，再由 `onTaskFinished` 触发 commit and publish。

**与 Stream Load 的三点实质差异：**

- **作业持久化**：`LoadJob` 会写 editlog、FE 重启能重放（`replayOnCommitted` / `replayOnVisible` 一系列 `replay*` 方法）。Stream Load 没有这种作业实体，客户端断了就断了。
- **失败重试语义**：Broker Load 的失败发生在 FE 调度的 task 上，可以在作业层面重试；进度可查（`SHOW LOAD`）。Stream Load 的重试是**客户端**用同一个 Label 重投整批（第 1 章 Label 幂等）。
- **拆分粒度**：Stream Load 一次一批一台协调 BE；Broker Load 一个作业可拆成多表、多 task 并行灌，天然适配 TB 级离线批。

### 易错点：两层状态机的对应关系，排障时看哪一层

这是 Broker Load 排障最容易犯迷糊的地方。系统里**同时**有两台状态机在转：

- **作业状态机** `JobState`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/JobState.java:21`）：`PENDING → ETL → LOADING → COMMITTED → FINISHED`，失败转 `CANCELLED`（`RETRY` / `UNKNOWN` 是边角态）。
- **事务状态机** `TransactionStatus`：`PREPARE → COMMITTED → VISIBLE / ABORTED`（第 1 章 1.3）。

关键在于**这两台状态机不是平行独立的，作业状态机被事务状态机驱动**。`LoadJob` 本身就是一个事务回调 `AbstractTxnStateChangeCallback`（`fe/fe-core/src/main/java/org/apache/doris/transaction/AbstractTxnStateChangeCallback.java`）的子类（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:86`）。事务每翻一个态，就回调作业、把作业状态推一格：

| 事务侧事件 | LoadJob 回调 | 作业状态落点 |
| --- | --- | --- |
| 任务开始跑计划 | `executeLoad`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:458`） | `LOADING` |
| 事务 `COMMITTED` | `afterCommitted`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:854`） | `COMMITTED` |
| 事务 `VISIBLE` | `afterVisible`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:935`） | `FINISHED` |
| 事务 `ABORTED` | `afterAborted`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:888`） | `CANCELLED` |

（状态真正翻转在 `unprotectedUpdateState`，`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java:420`；它开头有个自保护：作业进终态后拒绝再被推回非终态。）

**排障时看哪一层，取决于作业卡在哪：**

- 作业停在 **`LOADING` 很久**——事务还没 `COMMITTED`，问题在**内核写入侧**（loading task 在 BE 上没跑完：读 HDFS 慢、数据倾斜、某台 BE flush 反压）。这时候去看事务状态**没用**，事务还停在 `PREPARE`。该看 BE 上那个 loading fragment 的执行（第 3 章 3.6 的慢因清单）。
- 作业停在 **`COMMITTED` 不到 `FINISHED`**——事务已 `COMMITTED` 但没 `VISIBLE`。这就**纯粹是事务/publish 侧的问题**了，作业只是被动等回调，去查第 4 章 4.5 症状 B 那套 publish 积压排查。

把这两层搞混，就会出现"作业卡 LOADING 却跑去查 publish 队列"或"作业卡 COMMITTED 却去 BE 翻 loading 日志"的南辕北辙。记住这条因果链：**事务态是因，作业态是果**；作业停在哪一格，就反推事务到了哪一步，再决定去哪找卡点。

## 5.3 Routine Load：常驻消费作业

**差异定位：从"一次性作业"变成"永不结束的常驻作业"，并且要维护外部数据源的消费位点。** Broker Load 干完就 `FINISHED` 了；Routine Load 是一个 FE 上常驻的作业，周期性地把自己拆成一个个短小的子任务，每个子任务消费一段 Kafka 数据、走一次**独立的批级事务**、提交后**再拆下一批**——如此循环不停。

`RoutineLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:115`）是抽象基类，Kafka 实现是 `KafkaRoutineLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/kafka/KafkaRoutineLoadJob.java:94`）。它的状态机也和 Broker Load 完全不同（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:162`）：`NEED_SCHEDULE`（等调度）、`RUNNING`（常驻消费中）、`PAUSED`（出错暂停、可恢复）、`STOPPED` / `CANCELLED`（终态）。注意 `PAUSED` 是个 Broker Load 没有的态——常驻作业不能一遇错就死，它要能"暂停—修复—恢复"。

**task 划分**：`divideRoutineLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/kafka/KafkaRoutineLoadJob.java:248`）把当前订阅的 Kafka partition **按并发数轮询分组**，每组包一个 `KafkaTaskInfo`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/kafka/KafkaTaskInfo.java`）。并发数由 `calculateCurrentConcurrentTaskNum`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/kafka/KafkaRoutineLoadJob.java:313`）算出，取 partition 数、用户期望并发、集群上限三者的最小值。任务由 `RoutineLoadTaskScheduler`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadTaskScheduler.java:56`）派到 BE 执行。

**一个子任务什么时候结束？** 由三个"攒批上限"里最先撞到的那个决定（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:223`）：`max_batch_interval`（默认 60 秒，`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:123`）、`max_batch_rows`、`max_batch_size`（属性名定义在 `fe/fe-core/src/main/java/org/apache/doris/nereids/trees/plans/commands/info/CreateRoutineLoadInfo.java:83`）。撞到任一上限，子任务停止消费、提交这一批事务。**这三个参数直接决定了"多久提交一次、每次提交多大"——它们是本节两个易错点的总开关。**

### tricky 点：offset 提交与 Doris 事务绑定 —— exactly-once 的边界

Routine Load 号称 exactly-once，这个保证到底靠什么实现、边界在哪，是理解它的关键。答案是：**Kafka 消费位点（offset）不是独立提交给 Kafka 的，而是作为事务提交附件、随 Doris 事务一起原子落库的。**

具体看：一个子任务在 BE 上消费完一批数据，把这批的进度（各 partition 消费到的 offset）打包成 `RLTaskTxnCommitAttachment`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RLTaskTxnCommitAttachment.java:32`，其 `progress` 字段就是 `KafkaProgress`，见 `fe/fe-core/src/main/java/org/apache/doris/load/routineload/RLTaskTxnCommitAttachment.java:47`），挂在这次事务的 commit 请求上。FE 侧当事务翻到 `COMMITTED`，`afterCommitted`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:1158`）回调里走 `executeTaskOnTxnStatusChanged`，从事务附件里取出进度、调 `updateProgress`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:889`）把 offset 推进——**并且这一步和事务提交是同一条 editlog、同一个原子操作**。

这就是 exactly-once 的实现边界：**offset 前进 ⟺ 数据事务提交，两者要么同成、要么同败。**

- 事务提交成功 → offset 一起前进 → 下一个子任务从新 offset 接着消费，不会重读这批（不重复）。
- 事务失败/回滚 → offset **不前进** → 下一个子任务从老 offset 重新消费同一段（不丢）。

注意边界的**语义**：对 Kafka 而言这是 at-least-once 的重新消费（同一段可能被拉取多次），但落进 Doris 是 exactly-once ——因为"这批数据进没进库"和"offset 有没有前进"被同一个事务绑死，绝不会出现"数据进了但 offset 没记"（会重复）或"offset 记了但数据没进"（会丢）的错位。事务可见后，`afterVisible`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:1209`）再创建下一个子任务，消费循环就此接续。

### 易错点：max_batch_interval 调太小 → 小事务风暴 → -235

这是 Routine Load 生产事故的高频根因，且它的因果链一直伸到 BE 存储层。

有人为了"降低延迟"把 `max_batch_interval` 从 60 秒调到 1 秒。后果是：每个子任务最多攒 1 秒就提交一次，一个作业若有 N 个并发 task，就是每秒 N 个事务、每秒 N 个新版本往 tablet 上堆。而这些小事务每个只带一点点数据——**版本数暴涨，但数据量没涨**。这正是 compaction 最怕的局面：小 rowset 生成速度远超合并速度。

顺着第 4 章 4.5 建立的结论往下推：版本堆积到 `max_tablet_version_num` 上限，**新的写入会在 prepare 阶段直接 fast-fail，报 `-235 / TOO_MANY_VERSION`**（第 4 章已明确 -235 是**写入阶段**的锅、抛在 `RowsetBuilder::init`，不在 publish 日志里）。于是 Routine Load 子任务开始成批失败，作业被 `updateState` 推进 `PAUSED`。用户看到的表象是"Routine Load 老是自己暂停"，真正的病根却是自己把 `max_batch_interval` 调太小、制造了小事务风暴、把版本数打爆。

**正确方向和第 4 章症状 C 一脉相承**：攒批要够大（调大 interval / rows / size，让单个事务多带数据、少产版本），而不是追求极低延迟。低延迟高频小写入这个诉求，Doris 给的答案不是把 Routine Load 的 batch 调小，而是下一节的 Group Commit。

## 5.4 Insert Into 与 Group Commit

### Insert Into：查询计划本身就是数据源

**差异定位：数据源不再是外部流或文件，而是一个 SQL 查询计划。** `INSERT INTO t SELECT ...` / `INSERT INTO t VALUES ...` 由 `InsertIntoTableCommand`（`fe/fe-core/src/main/java/org/apache/doris/nereids/trees/plans/commands/insert/InsertIntoTableCommand.java:126`）执行。它是 5.1 归一化设计最纯粹的体现：这里根本不需要"造"一个 scan 源头——**整个 SELECT 查询计划就是数据源**，只要在它顶上接一个 `OlapTableSink` 就成了导入。取数、计算、过滤全走 part2 那套查询引擎，导入内核只管接住结果往 tablet 写。

`InsertLoadJob`（`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/InsertLoadJob.java:44`）在这里的角色和 Broker Load 的 `LoadJob` 很不一样：它主要是一条**事后的记账记录**，在执行器里登记（`fe/fe-core/src/main/java/org/apache/doris/nereids/trees/plans/commands/insert/AbstractInsertExecutor.java:112`），用于 `SHOW LOAD` 能查到这次 insert，而不是像 Broker Load 那样驱动多阶段调度。Insert Into 本身是**同步**的，行为上更接近 Stream Load（发起方阻塞等结果），只是数据源换成了查询计划。

### Group Commit：把海量小事务攒成一个共享事务

**差异定位：不是新的数据源，而是新的"提交节奏"——服务端把短时间内的多次小写入合并进同一个事务。** 5.3 末尾留的问题——高频小写入怎么办——答案就是它。逐条 `INSERT`（或高频小 Stream Load）如果各开各的事务，就是 5.3 那个小事务风暴的翻版：每条一个事务、一个版本，FE 事务表和 BE 版本数双双被打爆。Group Commit 让服务端把这些小写入**攒进一个共享的批级事务**，多次写入共用一次 commit+publish，用可见延迟换事务/版本数的数量级下降。

BE 侧核心结构在 `be/src/load/group_commit/group_commit_mgr.h`：`GroupCommitMgr`（`be/src/load/group_commit/group_commit_mgr.h:197`）管全局，每张表一个 `GroupCommitTable`（`be/src/load/group_commit/group_commit_mgr.h:148`），表下挂一个 `LoadBlockQueue`（`be/src/load/group_commit/group_commit_mgr.h:52`）——这个队列就是"攒批"的容器：多个写入请求把自己的 Block 塞进同一个队列，共享队列持有的那一个 `txn_id`；攒到时间或大小阈值（`_group_commit_interval_ms` / `_group_commit_data_bytes`）就整队一起提交。

**WAL 的角色（与第 3 章保持一致）**：第 3 章已经把这件事说死——**常规导入路径没有 per-row WAL，`be/src/load/group_commit/wal/` 下的 WAL 是 Group Commit 专属的局部机制**。这里补上"为什么专属"：Group Commit 在 async 模式下，客户端**写完就返回、不等这个共享事务 commit**。这一刻客户端手里已经没有数据可供重试了，如果 BE 在共享事务提交前宕机，攒在 `LoadBlockQueue` 里的数据就会丢。所以 async 模式必须先把数据写进 WAL（`WalWriter`，`be/src/load/group_commit/wal/wal_writer.h:32`）兜底，宕机后由 `WalManager`（`be/src/load/group_commit/wal/wal_manager.h:46`）后台线程重放（`be/src/load/group_commit/wal/wal_manager.h:73` 的 replay 逻辑）把未提交的批补回来。这是唯一需要 WAL 的导入模式，恰恰因为它是唯一"客户端提前撒手、又还没落进事务"的模式。

### tricky 点：三种模式与可见性延迟语义

Group Commit 由会话变量 `group_commit`（`fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java:599`，字段默认 `off_mode`，`fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java:2662`）控制，取三个值（`fe/fe-core/src/main/java/org/apache/doris/common/util/PropertyAnalyzer.java:252`，解析见 `GroupCommitBlockSink.parseGroupCommit`，`fe/fe-core/src/main/java/org/apache/doris/planner/GroupCommitBlockSink.java:86`）：

- **`off_mode`**：关闭，退回普通导入，每次写入独立事务。
- **`async_mode`**：写入进 WAL + 队列后**立即返回成功**，共享事务由服务端后台攒批提交。吞吐最高，但有**可见延迟**——返回成功 ≠ 立刻查得到，要等那个共享事务真正 commit+publish（延迟约等于攒批间隔）。
- **`sync_mode`**：写入进队列后**阻塞等到共享事务提交**才返回。没有可见延迟（返回即可见），但每次写入要多等一个攒批窗口的尾巴。

模式选择在 BE 的 `add_block` 调用里落地：写不写 WAL 直接由"是不是 async"决定（`be/src/exec/operator/group_commit_block_sink_operator.cpp:144`，第三个参数 `write_wal = (_group_commit_mode == ASYNC_MODE)`）。还有一个容易忽略的兜底：async 模式下如果 WAL 磁盘空间不够，会**自动降级为 sync**（`be/src/exec/operator/group_commit_block_sink_operator.cpp:193`）——没盘写 WAL 就没法保证 async 的持久性，只能退回"阻塞等提交"这条不依赖 WAL 的路。

**async 的可见延迟语义是最容易被误解的点**：它和第 4 章讲的"COMMITTED ≠ VISIBLE"窗口**不是一回事**。第 4 章那个窗口是 commit 之后 publish 之前；Group Commit async 的延迟发生得**更早**——你的这条数据可能**还没被纳入任何一个已提交的事务**，它还躺在 `LoadBlockQueue` 里等着和别的写入凑够一批。所以"async_mode 下 insert 返回成功却查不到"是**设计内行为**，不是 bug；要立刻可见就用 sync_mode 或关掉 Group Commit。

### 易错点：逐条 insert 不开 Group Commit 的事务风暴

这是把 Doris 当 OLTP 用的经典翻车。有人写一个循环，每次 `INSERT INTO t VALUES (...)` 一行，`group_commit` 保持默认 `off_mode`。于是每一行都是一个完整的批级事务：begin → 计划 → sink → commit → publish，每行产生一个新版本。几千行灌下来，tablet 版本数直冲上限，`-235` 如约而至——和 5.3 那个小事务风暴同因同果，只是触发方从 Routine Load 换成了手写循环。Group Commit 存在的全部意义就是消灭这种场景：把 `group_commit` 设成 `async_mode` / `sync_mode`，那几千行就被攒成寥寥几个共享事务，版本数从"每行一个"降到"每批一个"。

## 5.5 双模式差异

四种导入方式的差异都集中在 FE 编排层（触发、拆 task、驱动状态机），这一层在存算一体与存算分离下**是一致的**——`BrokerLoadJob`、`RoutineLoadJob`、`GroupCommitMgr` 不关心底下是哪种模式。真正的双模式分野发生在它们共用的那条内核尾部：commit 落点是走 Publish 还是走 MetaService、可见性怎么达成。这部分已由第 4 章完整覆盖（4.3 存算分离的 MetaService 提交），本章不再重复。

## 5.6 动手实验

环境搭建见 part1 第 5 章，不重复。本实验**不依赖外部组件**（Kafka 那套见文末说明），核心点和易错点都用 Insert Into + Group Commit 复现——它们和 Routine Load 的小事务风暴同根同源，能在单机集群直接看到版本数的变化。

### 实验一（核心点）：观察 Group Commit 把多次写入攒成一个事务

**目标**：亲眼看到"多次 insert 只涨少数几个版本"，验证 5.4 的共享事务攒批。

1. 建一张单分区表，记下它当前版本：`SHOW PARTITIONS FROM t;` 看 `VisibleVersion`（记为 V0）。
2. **开 Group Commit**：`SET group_commit = async_mode;` 然后在**同一个会话**里快速连发若干条单行插入：
   ```sql
   INSERT INTO t VALUES (1);
   INSERT INTO t VALUES (2);
   -- ... 连发 20 条
   ```
3. 等一个攒批间隔（默认约 10 秒）后再查 `SHOW PARTITIONS FROM t;` 的 `VisibleVersion`。**观察点**：版本从 V0 只涨了个位数（多条 insert 被合并进同一个共享事务），而不是涨 20。用 `SHOW PROC '/transactions/<dbId>/finished'` 也能看到事务个数远少于 insert 条数。
4. **顺便踩一下 async 的可见延迟**（5.4 tricky 点）：第 2 步发完后**立刻** `SELECT COUNT(*) FROM t;`——很可能查不到刚插的行，因为它们还在 `LoadBlockQueue` 里没进事务。等几秒再查才齐。把 `group_commit` 改成 `sync_mode` 重做，返回即可见。

**这个实验验证的核心点**：Group Commit 用"攒批共享事务"把版本增长从 O(写入次数) 压到 O(批数)，代价是 async 模式下一段可见延迟。

### 实验二（踩易错点）：关掉 Group Commit 的逐条 insert 事务风暴

**目标**：亲手制造版本暴涨、逼近 `-235`，验证 5.4 易错点与第 4 章 -235 结论。

1. `SET group_commit = off_mode;`（回到默认）。
2. 对同一张表跑一个循环，逐条插入几百到上千行（用脚本发 `INSERT INTO t VALUES (...)`，一次一行、不用 Group Commit）。
3. 边插边盯版本：`SHOW PARTITIONS FROM t;` 的 `VisibleVersion` 会**跟着 insert 条数几乎线性上涨**——每行一个事务一个版本，正是事务风暴的指纹。
4. **逼近 -235**：把 BE 的 `max_tablet_version_num` 临时调小（配置见第 4 章），继续灌，直到新 insert 报 `[E-235] TOO_MANY_VERSION`。确认它抛在**写入 prepare 阶段**（BE `be.INFO` 里的 `RowsetBuilder::init` 报错），而不是 publish 日志——复刻第 4 章"别把 -235 当 publish 卡点找"的结论。
5. **对照修复**：把同样的循环改成 `SET group_commit = async_mode;` 重跑，版本数增长立刻塌下来，-235 不再出现。

**要建立的认知**：高频小写入的敌人是**版本数**不是数据量；逐条独立事务是版本暴涨之源，Group Commit（以及 Routine Load 里调大 `max_batch_interval`）是同一味解药——都是"攒批换版本数"。

**关于 Routine Load + Kafka（诚实说明依赖）**：要复现 5.3 的 offset-事务绑定和 `max_batch_interval` 小事务风暴，需要一个可用的 Kafka。若环境具备，可 `CREATE ROUTINE LOAD` 订阅一个 topic，把 `max_batch_interval` 设为 1 观察版本暴涨与作业 `PAUSED`；用 `SHOW ROUTINE LOAD` 看进度与 offset。因为它引入外部组件、不易在单机稳定复现，本章把可独立运行的核心/易错点都落在了上面 Insert Into + Group Commit 的两个实验上——两者的版本增长机理与 Routine Load 完全一致。

## 5.7 排查清单

按"症状 → 定位路径"组织，覆盖本章三种导入方式各自最典型的卡点。事务/publish 侧的通用排查见第 4 章 4.5，写入侧慢因见第 3 章 3.6。

### 症状 A：Broker Load 长时间卡在 LOADING

- **先分清作业停在哪一格**（5.2 易错点）：`SHOW LOAD` 看 `State`。停在 `LOADING` 说明事务还没 `COMMITTED`，问题在**内核写入侧**，别去查 publish。
- 去执行 loading task 的那台 BE 看 fragment：多半是读 HDFS/对象存储慢、数据倾斜（个别 task 数据量畸大）、或某台 BE flush 反压（第 3 章 `MemTableMemoryLimiter` 那套指纹）。
- 若停在 `COMMITTED` 不到 `FINISHED`，那才是事务/publish 的事，转第 4 章 4.5 症状 B（publish 积压）。
- 别把两层状态机搞混——作业态是果、事务态是因，先反推事务停在哪步再找卡点。

### 症状 B：Routine Load 自己停了（PAUSED），怎么定位原因

- `SHOW ROUTINE LOAD` 看 `State` 与 `ReasonOfStateChanged` / `ErrorLogUrls`。`PAUSED` 是可恢复态，重点是读暂停原因。
- **原因一：错误行超阈值**——`updateState(PAUSED, TOO_MANY_FAILURE_ROWS_ERR)`（`fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java:941`）。去 `ErrorLogUrls` 看脏数据，是 schema 不匹配还是格式错。
- **原因二：写入侧 -235**（5.3 易错点的因果链）——子任务反复因 `TOO_MANY_VERSION` 提交失败导致暂停。查是不是 `max_batch_interval` / `max_batch_rows` 设太小造成小事务风暴、版本堆积；解药是**调大攒批**，不是调小。
- **原因三：连不上 Kafka / offset 越界**——看 reason 里的 Kafka 报错。offset 因 Doris 事务绑定（5.3 tricky 点），修复后从上次成功提交的 offset 接续，不会重复也不丢。

### 症状 C：insert 频繁失败或超慢

- **十有八九是逐条 insert 没开 Group Commit 的事务风暴**（5.4 易错点）。查 `SHOW PARTITIONS` 的 `VisibleVersion` 是不是跟 insert 次数同步猛涨；若已报 `-235`，确认它抛在写入 prepare 阶段（第 4 章结论）。解药是开 `group_commit`。
- 若已开 async_mode 却"插了查不到"，那**不是失败**，是 async 的可见延迟（5.4 tricky 点）——数据还在 `LoadBlockQueue` 没进事务。要即时可见改 sync_mode。
- `INSERT INTO ... SELECT` 慢，多半慢在 SELECT 侧（它就是个查询计划，5.4）——按 part2 的查询排查思路先看 SELECT 本身的执行，再看 sink 写入。

---

本章没有重走导入主干，而是把四种导入方式**按差异点**钉在了第 2~4 章那条基线上。先论证了 Doris 为什么不为每种数据源造专用链路、而是归一到"起事务 → 起计划 → sink → commit"一条内核、各方式只在最前端换取数方式（5.1）；再逐一挖了每种方式在这条内核之外**新增**的东西：Broker Load 带来的 `LoadJob` 异步作业状态机、以及"作业态被事务态驱动"这个排障时必须分清的两层对应（5.2）；Routine Load 常驻消费的 task 划分、offset 随事务原子提交的 exactly-once 边界、以及 `max_batch_interval` 调太小引发小事务风暴逼近 -235 的因果链（5.3）；Insert Into"查询计划即数据源"的纯粹归一化、以及 Group Commit 用共享事务攒批消灭小事务、三种模式的可见性延迟语义与 WAL 的专属角色（5.4）。贯穿全章的一条主线是：**高频小写入的敌人永远是版本数**，Routine Load 调大攒批、Insert 开 Group Commit，是同一味解药的不同外观。到这里，一批数据从五种不同的源头进入、汇入同一条内核、直到可见的全景就此闭环——接下来数据留在磁盘上会随版本累积而碎片化，需要后台把它们合并整理，那是本部分后续 compaction 章节的主题。
