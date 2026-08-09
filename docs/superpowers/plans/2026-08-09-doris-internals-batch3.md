# 《Doris 内核透视》第三批交付物实施计划（第三部分：一次导入的一生）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第三部分（一次导入的一生）全部 6 章及部分目录页，并将系列 README 的第三部分状态翻转为"已完成"。

**Architecture:** 纯文档写作项目。每章按设计文档第 5 节五段式骨架写作；主线是一次 Stream Load 从 HTTP 请求发出到数据可见的完整路径，6 章按"事务模型→接入与计划→写入落盘→提交可见→其他导入方式→Compaction"推进。每个任务 = 一个章节文件，流程固定为"核实素材 → 写作 → 引用校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid 图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束沿用前两批并补充第三部分锚点，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（严禁泛泛而谈）：遇到了什么问题？→ 有哪些候选方案、各有什么优劣？→ Doris 最终怎么考量和解决的？
- **源码走读分清主次**：显而易见的高度概括；不易理解的详细逐段解释；重点挖掘易错点/tricky 点并解释"为什么这么写、错写会怎样"。
- **动手实验双目的**：验证核心点 + 主动踩一遍易错点。环境说明引用 part1 第 5 章，不重复。
- 双模式并重：本部分第 1、4、6 章是双模式差异主战场（事务提交点、Publish vs MetaService、Compaction 执行位置），必须完整展开；第 2、3、5 章在对应小节交代差异即可。
- 所有源码引用格式为 `路径:行号` 或 `路径`，必须在当前 master 真实存在；**先核实再落笔，禁止凭记忆写引用**。
- **裸类名陷阱**：引用校验脚本只校验带后缀路径；正文提到的类名/方法名必须逐一 `grep -rn "class Xxx"` 核实；自查与审阅都要人工核对裸类名。
- **未成文部分的前向引用一律纯文本**（如"详见 part5"），不用 markdown 链接。
- 已核实的第三部分关键锚点（目录/文件级已核实，行号需现场核实）：
  - **本 master 导入逻辑集中在 `be/src/load/`**（旧资料常写 runtime/olap 位置，已重构）：`be/src/load/stream_load/`（`stream_load_executor.cpp`、`stream_load_context.h`）、`be/src/load/memtable/`（`memtable.cpp`、`memtable_writer.cpp`、`memtable_flush_executor.cpp`、`memtable_memory_limiter.h`）、`be/src/load/delta_writer/`（`delta_writer.cpp`）、`be/src/load/routine_load/`、`be/src/load/group_commit/`、`be/src/load/channel/`
  - BE HTTP 入口在 `be/src/service/http/action/`（`stream_load.h`、`stream_load_2pc.h`），注册在 `be/src/service/http_service.cpp`
  - BE 云侧对应：`be/src/cloud/cloud_stream_load_executor.cpp`、`be/src/cloud/cloud_delta_writer.cpp`、`be/src/cloud/cloud_base_compaction.h`、`cloud_cumulative_compaction_policy.h`、`cloud_full_compaction.cpp`
  - FE 事务：`fe/fe-core/src/main/java/org/apache/doris/transaction/`（`GlobalTransactionMgr.java`、`DatabaseTransactionMgr.java`、`TransactionState.java`（class 在约 :61）、`PublishVersionDaemon.java`、`SubTransactionState.java`）；云侧 `fe/fe-core/src/main/java/org/apache/doris/cloud/transaction/CloudGlobalTransactionMgr.java`（part1 第 4 章已证实经 `MetaServiceProxy.commitTxn` 提交）
  - FE 导入作业：`fe/fe-core/src/main/java/org/apache/doris/load/loadv2/`（`BrokerLoadJob.java`、`InsertLoadJob.java`、`LoadJob.java`、`JobState.java`）、`fe/fe-core/src/main/java/org/apache/doris/load/routineload/`（`RoutineLoadJob.java`、`kafka/`）
  - Sink 算子：`be/src/exec/operator/olap_table_sink_operator.h`、`olap_table_sink_v2_operator.h`（还有 `be/src/exec/sink/` 下的实现，现场核实分工）
  - Rowset 写出：`be/src/storage/rowset/beta_rowset_writer.cpp`（及 v2）；Publish：`be/src/storage/task/engine_publish_version_task.cpp`
  - 主键模型 Delete Bitmap：`be/src/storage/delete/`（`delete_bitmap_calculator.cpp`、`calc_delete_bitmap_executor.cpp`）
  - Compaction：`be/src/storage/compaction/`（`base_compaction.h`、`cumulative_compaction_policy.cpp`、`full_compaction.cpp`、`cold_data_compaction.h`）
  - part2 已验证可复用结论：`-235`=TOO_MANY_VERSION（`be/src/common/status.h`）、`max_tablet_version_num`（`be/src/common/config.cpp`）、版本区间语义见 part1 第 3 章 3.3
- 构建/测试命令与根 `AGENTS.md` 一致。章内交叉引用：part1/part2 用相对链接 `../part1-architecture/0X-*.md`、`../part2-query-lifecycle/0X-*.md`；本部分章间同目录相对链接。每章开头基线说明格式：「基于写作时核实所用的 HEAD（`XXXX`，源码树与系列基线 `7bc98f696f` 一致）」。
- 每完成一个文件即提交，前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行（见任务内命令）。
- 每个写作任务完成后执行统一**引用校验步骤**（含 `.g4` 后缀）：

```bash
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md|g4)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

预期输出为空；出现 MISSING 必须修正后重跑再提交。

---

### Task 1: 第 1 章《导入方式总览与事务模型》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/01-load-overview-and-txn.md`

**Interfaces:**
- Consumes: part1 第 3 章版本区间、part1 第 4 章双模式事务管理器对照。
- Produces: Label/2PC/事务状态机概念与 `TransactionState` 锚点，本部分后续各章直接引用。

- [ ] **Step 1: 核实素材**

```bash
grep -n "enum TransactionStatus" fe/fe-core/src/main/java/org/apache/doris/transaction/TransactionState.java
sed -n '61,120p' fe/fe-core/src/main/java/org/apache/doris/transaction/TransactionState.java
grep -n "beginTransaction\|commitTransaction\|abortTransaction" fe/fe-core/src/main/java/org/apache/doris/transaction/GlobalTransactionMgr.java | head
grep -n "label" fe/fe-core/src/main/java/org/apache/doris/transaction/DatabaseTransactionMgr.java | head -10
grep -rn "label_keep_max_second\|streaming_label_keep_max_second" fe/fe-core/src/main/java/org/apache/doris/common/Config.java | head -3
ls fe/fe-core/src/main/java/org/apache/doris/load/loadv2/ | head; ls fe/fe-core/src/main/java/org/apache/doris/load/routineload/ | head -5
```

- [ ] **Step 2: 写作**

创建 `01-load-overview-and-txn.md`（约 6000-8000 字），结构：

```markdown
# 第 1 章：导入方式总览与事务模型

## 1.1 问题：怎么把"一批数据"原子地放进一个分布式系统
（三连问：多 tablet 多副本下的导入原子性——候选：无事务尽力写（重复/丢失）/
 每行独立幂等（对分析型吞吐不友好）/ 批级事务+Label 幂等+两阶段提交；
 Label 机制为什么能挡重复提交；Doris 的取舍）

## 1.2 导入方式全景
（一张表：Stream Load / Broker Load / Routine Load / Insert Into /
 Group Commit 的触发方、数据源、同步性、适用场景；各自入口类一句话+
 前向指到第 2/5 章；本章只建立分类心智）

## 1.3 源码走读：事务状态机
（TransactionState 的状态枚举与流转（PREPARE→PRECOMMITTED?→COMMITTED→
 VISIBLE / ABORTED，按真实枚举核实）；GlobalTransactionMgr/
 DatabaseTransactionMgr 的分库管理；mermaid 状态图；
 tricky 点：COMMITTED≠可见——commit 与 publish/visible 的间隙就是
 part2 曾提过的"导入成功但查不到"窗口，讲透两个时间点的语义；
 易错点：Label 复用的行为（已成功/进行中/已失败的 Label 重投分别怎样）、
 label_keep_max_second 过期后幂等失效）

## 1.4 双模式对比（本章重点段）
（事务管理器双实现：GlobalTransactionMgr（bdbje 持久化+内存状态）vs
 CloudGlobalTransactionMgr（薄客户端，状态在 MetaService/FDB）；
 引用 part1 4.4 的对照，本章深入差异的语义后果：FE 重启后事务恢复来源、
 事务高可用边界；2PC 接口（precommit/commit）在两模式的落点）

## 1.5 动手实验
（核心点：用 SHOW PROC '/transactions' 或对应命令观察一次导入的
 状态流转（核实真实命令）；易错点：同一 Label 重复提交三种时机各试一次，
 观察报错/幂等行为差异）

## 1.6 排查清单
（症状→路径：导入卡在 COMMITTED 不 VISIBLE / Label already used 误判 /
 事务数超限报错的定位入口）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/01-load-overview-and-txn.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/01-load-overview-and-txn.md
git commit -m "[docs] doris-internals part3: ch1 load overview and transaction model

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 2 章《Stream Load 全路径》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/02-stream-load-path.md`

**Interfaces:**
- Consumes: 第 1 章事务模型；part2 第 5/6 章的 fragment 下发与 pipeline 框架。
- Produces: HTTP 接入→计划→Sink 分发的链路，第 3 章从 DeltaWriter 接力。

- [ ] **Step 1: 核实素材**

```bash
grep -n "stream_load" be/src/service/http_service.cpp | head -5
ls be/src/service/http/action/ | grep -i stream
grep -n "class StreamLoadAction\|handle" be/src/service/http/action/stream_load.h | head -5
ls be/src/load/stream_load/
grep -n "class StreamLoadExecutor\|begin_txn\|commit_txn" be/src/load/stream_load/stream_load_executor.cpp | head -8
grep -rn "streamLoadPut\|TStreamLoadPutRequest" gensrc/thrift/FrontendService.thrift | head -3
ls be/src/load/channel/
grep -n "class LoadChannel\b\|class TabletsChannel" be/src/load/channel/*.h | head -4
grep -rn "class VOlapTableSink\|OlapTableSink" be/src/exec/sink/*.h 2>/dev/null | head -4
grep -rn "node_channel\|NodeChannel" be/src/exec/sink/*.h 2>/dev/null | head -3
```

- [ ] **Step 2: 写作**

创建 `02-stream-load-path.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：Stream Load 全路径 —— 从 HTTP 请求到数据分发

## 2.1 问题：客户端一条 HTTP 流怎么变成分布在多台 BE 上的写入
（三连问：接入点选 FE 还是 BE（FE 转发瓶颈 vs BE 直收+重定向）；
 数据只有一份流、目标却是多 tablet 多副本——谁来切分与分发；
 Doris 的 BE 直收 + 内部起导入计划的设计）

## 2.2 源码走读：HTTP 接入与事务开启
（http_service 注册 → StreamLoadAction 处理请求头（label/format/columns）→
 StreamLoadExecutor 向 FE begin txn + 请求导入计划（streamLoadPut）；
 mermaid 时序图：client→BE(http)→FE(txn+plan)→BE(执行)；
 tricky 点：301 重定向与认证头透传——curl 不带 --location-trusted 的
 经典失败；易错点：格式/列映射错误在哪一层报出来）

## 2.3 源码走读：导入计划与 Sink 分发
（导入本质是一个只有 scan(http流)→sink 的计划，复用 part2 的执行框架；
 OlapTableSink 按 tablet 分桶规则切分行、经 NodeChannel 发往各 BE 的
 LoadChannel/TabletsChannel（be/src/load/channel/）；
 tricky 点：一行数据到达"错误 BE"是不存在的——sink 端按分桶计算目标，
 但 tablet 版本/schema 不一致时的失败路径；
 易错点：单行超限/字符集/空值造成的整批失败与容错参数（max_filter_ratio））

## 2.4 双模式对比
（接入与计划两模式一致；分发目标 BE 的选择在分离模式跟随计算组
 （引用 part2 5.4 的 CloudReplica 映射）；云侧 StreamLoadExecutor 子类
 （cloud_stream_load_executor.cpp）改了哪一步（commit 走 MetaService））

## 2.5 动手实验
（核心点：curl 发起一次 Stream Load，打开 FE/BE debug 日志对照 2.2 时序图
 找到各环节日志；易错点：不带 --location-trusted 重定向丢认证、
 故意构造超 max_filter_ratio 的脏数据看报错与 error url）

## 2.6 排查清单
（症状→路径：-235 反压 / 卡在数据传输 vs 卡在 plan / error url 怎么读）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/02-stream-load-path.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/02-stream-load-path.md
git commit -m "[docs] doris-internals part3: ch2 stream load full path

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 3 章《Tablet 写入细节：MemTable 到 Segment》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/03-tablet-write-path.md`

**Interfaces:**
- Consumes: 第 2 章的 TabletsChannel 落点；part1 第 3 章 Rowset/Segment 层级。
- Produces: DeltaWriter/MemTable/Flush/Delete Bitmap 细节，第 4 章讲提交、第 6 章讲 Compaction 时引用。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/load/delta_writer/ be/src/load/memtable/
grep -n "class DeltaWriter\|write\|close" be/src/load/delta_writer/delta_writer.h | head -8
grep -n "class MemTable\b\|insert\|flush" be/src/load/memtable/memtable.h | head -8
grep -n "class MemTableWriter\|_flush_memtable" be/src/load/memtable/memtable_writer.h | head -5
grep -n "class MemtableFlushExecutor\|class FlushToken" be/src/load/memtable/memtable_flush_executor.h | head -4
grep -rn "write_buffer_size" be/src/common/config.cpp | head -3
grep -n "class MemTableMemoryLimiter" be/src/load/memtable/memtable_memory_limiter.h
ls be/src/storage/delete/
grep -n "class DeleteBitmapCalculator\|class CalcDeleteBitmapExecutor" be/src/storage/delete/*.h | head -3
grep -rn "class CloudDeltaWriter" be/src/cloud/cloud_delta_writer.h | head -2
grep -n "segment_writer" be/src/storage/rowset/beta_rowset_writer.cpp | head -5
```

- [ ] **Step 2: 写作**

创建 `03-tablet-write-path.md`（约 6000-8000 字），结构：

```markdown
# 第 3 章：Tablet 写入细节 —— MemTable、Flush 与 Delete Bitmap

## 3.1 问题：高频小批写怎么变成列存大文件
（三连问：来一批写一个文件（小文件爆炸）vs WAL+内存表攒批（LSM 经典）
 vs 直接改写已有文件（列存代价不可接受）；MemTable 攒批+Flush 成
 Segment 的选择；与 RocksDB 类 LSM 的异同（Doris 没有 WAL？——核实
 group commit WAL 的存在与角色再落笔））

## 3.2 源码走读：DeltaWriter→MemTable→Flush
（TabletsChannel 每 tablet 一个 DeltaWriter；MemTable 的内存组织
 （按 schema 攒列、聚合模型在内存先聚合——核实实现）；写满
 write_buffer_size 触发 flush，MemtableFlushExecutor 异步刷成 segment
 （经 RowsetWriter）；mermaid 流程图；
 tricky 点：MemTableMemoryLimiter 全局内存水位反压——导入全局变慢
 的常见根因；易错点：flush 线程池打满时的堆积表现）

## 3.3 源码走读：主键模型的 Delete Bitmap
（MoW 写入为什么要算 delete bitmap（新数据到来时标记旧行删除）；
 DeleteBitmapCalculator/CalcDeleteBitmapExecutor 的计算时机
 （写入时+publish 时，按真实代码核实分工）；
 tricky 点：bitmap 计算依赖的"可见版本集合"与并发导入的相互影响；
 易错点：主键表大批量随机 upsert 的写放大来源）

## 3.4 双模式对比
（CloudDeltaWriter 与 DeltaWriter 的差异：rowset 落对象存储、
 元数据经 MetaService；delete bitmap 在分离模式的存放与锁
 （DeleteBitmapUpdateLockContext，核实后落笔））

## 3.5 动手实验
（核心点：小 write_buffer_size 下导入，观察 flush 日志与 segment 文件
 生成（对照 part1 3.3 的目录规则）；易错点：并发导入把
 memtable 内存打到 limiter 水位，观察反压日志与导入变慢）

## 3.6 排查清单
（症状→路径：导入慢先分清 sink 慢/flush 慢/提交慢 / 主键表导入
 越来越慢 / MEM_LIMIT_EXCEEDED 的读法）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/03-tablet-write-path.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/03-tablet-write-path.md
git commit -m "[docs] doris-internals part3: ch3 tablet write path memtable flush

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 4 章《事务提交与可见性》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/04-commit-and-visibility.md`

**Interfaces:**
- Consumes: 第 1 章状态机、第 3 章写完的 rowset。
- Produces: Publish/可见性机制，part6 导入故障篇与本部分第 6 章引用。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class PublishVersionDaemon\|publishVersion" fe/fe-core/src/main/java/org/apache/doris/transaction/PublishVersionDaemon.java | head -5
grep -n "commitTransaction" fe/fe-core/src/main/java/org/apache/doris/transaction/DatabaseTransactionMgr.java | head -5
grep -n "class EnginePublishVersionTask" be/src/storage/task/engine_publish_version_task.h | head -2
grep -rn "publish_version" be/src/agent/task_worker_pool.cpp 2>/dev/null | head -3
grep -n "commitTxn\|getVisibleVersion" fe/fe-core/src/main/java/org/apache/doris/cloud/transaction/CloudGlobalTransactionMgr.java | head -8
grep -rn "commit_txn" cloud/src/meta-service/*.cpp | head -5
grep -rn "visible_version\|partition_version" cloud/src/meta-service/meta_service_txn.cpp 2>/dev/null | head -5
```

- [ ] **Step 2: 写作**

创建 `04-commit-and-visibility.md`（约 6000-8000 字），结构：

```markdown
# 第 4 章：事务提交与可见性 —— Publish Version 与 MetaService 提交

## 4.1 问题：多副本多 tablet 的"同时可见"怎么做
（三连问：每副本自行可见（读到不一致版本）vs 全局锁（吞吐死）vs
 版本号推进——commit 定版本、publish 让各副本追平、读走可见版本；
 quorum 提交（多数副本成功即 commit）的取舍与副本落后的修复责任）

## 4.2 源码走读：存算一体的两段——commit 与 publish
（DatabaseTransactionMgr.commitTransaction 定版本+写编辑日志；
 PublishVersionDaemon 周期驱动，向 BE 发 publish 任务
 （EnginePublishVersionTask 把 rowset 标为可见版本）；
 mermaid 时序图：commit→publish→VISIBLE 的完整链路；
 tricky 点：publish 是异步尽力而为——失败重试与"卡在 COMMITTED"
 的所有成因（副本宕机/-235/schema change 冲突）；
 易错点：visibleVersion 与 nextVersion 的差值就是积压深度，
 怎么用命令看（核实 SHOW PROC 路径））

## 4.3 源码走读：存算分离的提交（本章重点段）
（CloudGlobalTransactionMgr.commitTxn → MetaService 的 commit_txn RPC
 （cloud/src/meta-service/，核实具体文件与关键步骤：FDB 事务里改
 tablet 元数据+分区版本）；没有 per-BE publish——版本推进在 MetaService
 一处完成，BE 读时拉取；对照 4.2 逐点比较：提交延迟构成、
 失败模式、"卡 COMMITTED"在分离模式对应什么；
 tricky 点：FDB 事务的冲突重试对高频提交的影响）

## 4.4 动手实验
（核心点：导入后立刻循环查 SHOW PROC 事务状态，抓到 COMMITTED→VISIBLE
 的窗口；易错点：停掉一个副本所在 BE 再导入（三副本表），
 观察 quorum 提交成功但 publish 有 tablet 落后，再看副本修复补齐）

## 4.5 排查清单
（症状→路径：查询读不到刚导入的数据 / publish 积压怎么看深度与卡点 /
 分离模式 commit 冲突重试的日志特征）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/04-commit-and-visibility.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/04-commit-and-visibility.md
git commit -m "[docs] doris-internals part3: ch4 commit and visibility

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 5 章《其他导入方式：Broker / Routine / Insert Into / Group Commit》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/05-other-load-paths.md`

**Interfaces:**
- Consumes: 第 2 章 Stream Load 基线路径、第 1 章分类表。
- Produces: 各导入方式与基线路径的"差异点"地图。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class BrokerLoadJob" fe/fe-core/src/main/java/org/apache/doris/load/loadv2/BrokerLoadJob.java | head -2
grep -n "class LoadJob\|enum JobState" fe/fe-core/src/main/java/org/apache/doris/load/loadv2/LoadJob.java fe/fe-core/src/main/java/org/apache/doris/load/loadv2/JobState.java | head -4
grep -n "class RoutineLoadJob" fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java | head -2
ls fe/fe-core/src/main/java/org/apache/doris/load/routineload/kafka/
grep -rn "class KafkaRoutineLoadJob" fe/fe-core/src/main/java/org/apache/doris/load/routineload/ | head -2
grep -n "class InsertLoadJob" fe/fe-core/src/main/java/org/apache/doris/load/loadv2/InsertLoadJob.java | head -2
ls be/src/load/group_commit/
grep -rn "group_commit" be/src/load/group_commit/*.h | head -5
```

- [ ] **Step 2: 写作**

创建 `05-other-load-paths.md`（约 5000-7000 字），结构：

```markdown
# 第 5 章：其他导入方式 —— 与 Stream Load 基线的差异

## 5.1 问题：一条写入内核，多少种喂数据的姿势
（三连问：为每种数据源做专用链路 vs 全部转成一种内部形式复用一条链路；
 Doris 的答案：万变不离"起事务→起计划→sink→commit"，差异只在
 触发方与数据源接入层；本章按"差异点"讲而不是重复全路径）

## 5.2 Broker Load：FE 编排的异步批作业
（LoadJob/JobState 状态机（PENDING→LOADING→COMMITTED→FINISHED）；
 BrokerLoadJob 拆分与调度；与 Stream Load 差异：作业持久化、
 失败重试语义、进度可查；易错点：作业状态与事务状态两层状态机
 的对应关系，排障时看哪个）

## 5.3 Routine Load：常驻消费作业
（RoutineLoadJob/KafkaRoutineLoadJob：分 task、offset 管理、
 自动重试；tricky 点：offset 提交与 Doris 事务的绑定——
 exactly-once 的实现边界；易错点：消费积压/小事务风暴（task 划分
 参数）→ -235 的因果链）

## 5.4 Insert Into 与 Group Commit
（Insert Into（查询计划当数据源，InsertLoadJob）；高频小写的答案
 Group Commit（be/src/load/group_commit/：攒批共享一个事务，
 核实 WAL 的角色）；tricky 点：group commit 的可见性延迟语义；
 易错点：误用逐条 insert 不开 group commit 的事务风暴）

## 5.5 双模式对比
（各方式的差异集中在 FE 编排层，两模式一致；commit 落点差异
 已由第 4 章覆盖，一句引用）

## 5.6 动手实验
（核心点：起一个 Routine Load 消费本地 Kafka（或注明依赖），
 观察 task 与事务的对应；易错点：把 max_batch_interval 调小制造
 小事务风暴，观察版本数增长与 -235 逼近）

## 5.7 排查清单
（症状→路径：Broker Load 卡 LOADING / Routine Load 停消费 PAUSED 原因 /
 insert 频繁失败）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/05-other-load-paths.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/05-other-load-paths.md
git commit -m "[docs] doris-internals part3: ch5 other load paths

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 6 章《Compaction：版本合并的艺术》

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/06-compaction.md`

**Interfaces:**
- Consumes: 第 3 章的 rowset 产出、第 4 章的版本推进、part1 3.3 版本区间。
- Produces: Compaction 机制，part5 存储引擎与 part6 故障篇引用。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/storage/compaction/
grep -n "class CumulativeCompaction\b\|class BaseCompaction\b\|class FullCompaction" be/src/storage/compaction/*.h | head -4
grep -n "class CumulativeCompactionPolicy" be/src/storage/compaction/cumulative_compaction_policy.h | head -3
grep -rn "compaction_score\|_calc_compaction_score" be/src/storage/tablet/tablet.cpp 2>/dev/null | head -5
grep -rn "compaction" be/src/storage/storage_engine.cpp | grep -i "thread\|submit" | head -5
grep -rn "cumulative_compaction_min_deltas\|compaction_task_num" be/src/common/config.cpp | head -5
ls be/src/cloud/ | grep -i compaction
grep -n "class CloudBaseCompaction\|class CloudCumulativeCompaction" be/src/cloud/cloud_base_compaction.h be/src/cloud/cloud_cumulative_compaction.h 2>/dev/null | head -3
grep -rn "start_tablet_job\|finish_tablet_job" cloud/src/meta-service/*.cpp | head -3
```

- [ ] **Step 2: 写作**

创建 `06-compaction.md`（约 6000-8000 字），结构：

```markdown
# 第 6 章：Compaction —— 版本合并的艺术

## 6.1 问题：LSM 类系统绕不开的账
（三连问：小 rowset 越积越多——不合并（读放大爆炸+版本超限）vs
 每次导入即时合并（写放大爆炸）vs 分层合并摊销；
 cumulative/base 两层的设计（新数据快合、老数据少动）；
 与 RocksDB leveled/tiered 的对照；-235 是"合并跟不上导入"的告警）

## 6.2 源码走读：触发与选择
（compaction score 的计算（版本数驱动）；生产者线程周期挑 tablet
 （storage_engine 里的调度线程，核实线程名与提交路径）；
 CumulativeCompactionPolicy 挑选 input rowsets 的规则；
 tricky 点：cumulative point 的推进语义——它决定"哪些归 cumulative
 哪些归 base"，理解错会看不懂挑选行为；
 易错点：把 compaction 并发/内存参数调过头挤占导入与查询）

## 6.3 源码走读：执行与版本替换
（合并读（复用读路径的聚合/去重语义）→写新 rowset→
 元数据原子替换（stale rowsets 的保留窗口与回收）；
 tricky 点：合并期间新导入照常追加——版本区间不重叠如何保证；
 主键表 compaction 与 delete bitmap 的交互（第 3 章伏笔回收））

## 6.4 双模式对比（本章重点段）
（执行位置：一体=每 BE 管自己的 tablet；分离=计算组内 BE 执行但要
 经 MetaService 的 tablet job 锁（start_tablet_job/finish_tablet_job，
 part1 2.4 伏笔回收）防止多计算组重复合并；
 合并产物：本地盘替换 vs 对象存储新文件+元数据切换+旧文件回收
 （Recycler 伏笔，纯文本指向 part4）；
 费用视角：分离模式 compaction 消耗对象存储请求费）

## 6.5 动手实验
（核心点：高频小批导入把版本数推高，用 SHOW PROC/compaction action
 （核实 BE http 接口路径）观察 score 上升与合并触发、版本数回落；
 易错点：把 cumulative 触发阈值调大复现 -235，再调回观察恢复）

## 6.6 排查清单
（症状→路径：-235/-238 处置顺序（先限导入频率还是先催合并）/
 compaction 不动了（线程池/失败重试/锁）/ 合并风暴打满 IO）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part3-load-lifecycle/06-compaction.md）。预期无 MISSING；人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/06-compaction.md
git commit -m "[docs] doris-internals part3: ch6 compaction

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 7: 部分目录页 + 系列 README 翻转

**Files:**
- Create: `docs/doris-internals/part3-load-lifecycle/README.md`
- Modify: `docs/doris-internals/README.md`（第三部分状态翻转+章节改链接，仿第一/二部分完成格式）
- Modify: `docs/doris-internals/part2-query-lifecycle/README.md`（下一部分预告若为脆弱锚点则改为直链 ../part3-load-lifecycle/README.md，核实现状再动）

**Interfaces:**
- Consumes: Task 1-6 产出的 6 个章节文件。
- Produces: 完整可导航的第三部分。

- [ ] **Step 1: 核实现状**

```bash
ls docs/doris-internals/part3-load-lifecycle/
grep -n "第三部分" docs/doris-internals/README.md
grep -n "第三部分\|part3" docs/doris-internals/part2-query-lifecycle/README.md
```

- [ ] **Step 2: 写作与修改**

part3 README（约 500 字，仿 part1/part2 目录页：导语 2-3 句 + 6 行表格 + 下一部分预告纯文本或稳健链接）；系列 README 最小化修改（状态翻转 + 6 行改链接）；part2 README 预告链接改稳健。

- [ ] **Step 3: 全量校验**

三个文件引用校验 + 全树死链检查（脚本同前，预期无输出）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part3-load-lifecycle/README.md \
        docs/doris-internals/README.md \
        docs/doris-internals/part2-query-lifecycle/README.md
git commit -m "[docs] doris-internals part3: part index, series index flip to complete

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：最终整分支审查（fable 模型，含台账 Minor triage 与 fix-later 清单），修复确认后 push，向作者简报第三部分完成情况，继续第四部分计划制定。
