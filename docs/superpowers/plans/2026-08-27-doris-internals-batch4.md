# 《Doris 内核透视》第四批交付物实施计划（第四部分：元数据与 FE 内核）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第四部分（元数据与 FE 内核）全部 6 章及部分目录页，并将系列 README 的第四部分状态翻转为"已完成"。

**Architecture:** 纯文档写作项目。每章按设计文档第 5 节五段式骨架写作；主线是"一条元数据从诞生到高可用再到回收的一生"：内存结构（ch1）→持久化（ch2）→高可用（ch3）→存算分离元数据（ch4）→调度体系（ch5）→联邦元数据（ch6）。每个任务 = 一个章节文件，流程固定为"核实素材 → 写作 → 引用校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid 图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束沿用前三批并补充第四部分锚点，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（严禁泛泛而谈）：遇到了什么问题？→ 有哪些候选方案、各有什么优劣？→ Doris 最终怎么考量和解决的？
- **源码走读分清主次**：显而易见的高度概括；不易理解的详细逐段解释；重点挖掘易错点/tricky 点并解释"为什么这么写、错写会怎样"。
- **动手实验双目的**：验证核心点 + 主动踩一遍易错点。环境说明引用 part1 第 5 章，不重复。
- 双模式并重：本部分 ch2/ch3 以存算一体为主体（bdbje/Checkpoint/选主），须在章内注明分离模式对应物在 ch4；ch4 是分离模式专章；ch5 双模式并重（Tablet 均衡/修复 vs 计算组管理）；ch1/ch6 差异较小按需交代。
- 所有源码引用格式为 `路径:行号` 或 `路径`，必须在当前 master 真实存在；**先核实再落笔，禁止凭记忆写引用**。
- **裸类名陷阱**：引用校验脚本只校验带后缀路径；正文提到的类名/方法名必须逐一 `grep -rn "class Xxx"` 核实。
- **配置可变性陷阱（批 3 教训）**：实验中凡声称可热更的 BE 配置必须先核实 `DEFINE_mXxx`；FE 配置核实 `@ConfField(mutableAndMasterOnly/mutable)` 注解。
- **跨部分引用惯例（批 3 裁决）**：引用已成文部分的章节，首次提及用相对 md 链接（如 `[part1 第 2 章](../part1-architecture/02-three-components.md)` + 节号纯文本），重复提及可纯文本；**未成文部分（part5/6/7）一律纯文本**。**严禁凭印象写"partX 提过/已证"——回引前必须 grep 目标文件确认该内容真实存在（批 3 教训：4 处虚假回引）。**
- 已核实的第四部分关键锚点（目录/文件级已核实，行号需现场核实）：
  - 内存结构：`fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java`（`loadImage` 约 :2366、`saveImage` 约 :2753、tabletScheduler 创建约 :815）、`Database.java`、`OlapTable.java`；part1 第 2/3 章已建立 Env 单例与 Table→Tablet 层级，本部分引用不重复
  - Catalog 联邦：`fe/fe-core/src/main/java/org/apache/doris/datasource/`（`CatalogMgr.java`、`CatalogIf.java`、`ExternalCatalog.java`、`InternalCatalog.java`、`hive/`、`iceberg/`、`CatalogFactory.java`）
  - 持久化：`fe/fe-core/src/main/java/org/apache/doris/persist/EditLog.java`、`persist/meta/`（`FeMetaFormat.java`、`MetaHeader.java`、`MetaFooter.java`）、`journal/bdbje/`（`BDBJEJournal.java`、`BDBEnvironment.java`、`BDBJournalCursor.java`）、`master/Checkpoint.java`、`persist/MetaCleaner.java`
  - 高可用：`fe/fe-core/src/main/java/org/apache/doris/ha/`（`BDBHA.java`、`BDBStateChangeListener.java`、`FrontendNodeType.java`、`MasterInfo.java`）；part1 第 2 章已核实 FrontendNodeType 枚举（含 REPLICA）
  - 调度：`fe/fe-core/src/main/java/org/apache/doris/clone/`（`TabletScheduler.java`、`TabletChecker.java`、`TabletSchedCtx.java`、`BeLoadRebalancer.java`、`DiskRebalancer.java`、`PartitionRebalancer.java`、`ColocateTableCheckerAndBalancer.java`）
  - 云侧 FE：`fe/fe-core/src/main/java/org/apache/doris/cloud/catalog/`（`CloudEnv.java`、`CloudClusterChecker.java`、`CloudSystemInfoService.java`、`ComputeGroup.java`——注意 `resource/computegroup/ComputeGroup.java` 是另一个同名类，写作时必须区分）
  - MetaService：`cloud/src/meta-service/`（`meta_service.cpp/h`、`meta_service_txn.cpp`、`meta_service_job.cpp`、`meta_service_partition.cpp`、`meta_server.cpp`）、`cloud/src/meta-store/`（**`keys.h` 定义全部 FDB key 编码**：`instance_key`/`txn_label_key`/`partition_version_key` 等约 :344 起、`codec.h`、`txn_kv.h`）、`cloud/src/recycler/`（`recycler.cpp/h`、`checker.cpp`）、`cloud/src/resource-manager/resource_manager.cpp`
  - part3 已核实可复用结论：MetaService commit_txn 七步（part3 第 4 章）、tablet job lock（part3 第 6 章）、Recycler 回收 rowset 的 `RecycleRowsetPB::COMPACT` 入口
- 构建/测试命令与根 `AGENTS.md` 一致。章内基线注格式：「基于写作时核实所用的 HEAD（`XXXX`，源码树与系列基线 `7bc98f696f` 一致）」。
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

### Task 1: 第 1 章《Catalog 体系与元数据内存结构》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/01-catalog-and-memory.md`

**Interfaces:**
- Consumes: part1 第 2 章 Env 单例、part1 第 3 章逻辑层级。
- Produces: 元数据内存组织与锁模型认知，ch2 持久化与 ch5 调度在其上展开。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class Env\b" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java
grep -n "getInternalCatalog\|getCatalogMgr" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java | head -4
grep -n "class InternalCatalog" fe/fe-core/src/main/java/org/apache/doris/datasource/InternalCatalog.java
grep -n "idToDb\|fullNameToDb" fe/fe-core/src/main/java/org/apache/doris/datasource/InternalCatalog.java | head -4
grep -n "class Database\b" fe/fe-core/src/main/java/org/apache/doris/catalog/Database.java
grep -n "readLock\|writeLock\|MonitoredReentrant" fe/fe-core/src/main/java/org/apache/doris/catalog/Database.java | head -6
grep -n "class OlapTable" fe/fe-core/src/main/java/org/apache/doris/catalog/OlapTable.java
grep -rn "class TabletInvertedIndex" fe/fe-core/src/main/java/org/apache/doris/catalog/TabletInvertedIndex.java | head -2
grep -rn "getMaxJournalId\|replayedJournalId" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java | head -3
grep -rn "metadata_mem" fe/fe-core/src/main/java/org/apache/doris/common/Config.java fe/fe-common/src/main/java/org/apache/doris/common/Config.java 2>/dev/null | head -3
```

- [ ] **Step 2: 写作**

创建 `01-catalog-and-memory.md`（约 6000-8000 字），结构：

```markdown
# 第 1 章：Catalog 体系与元数据内存结构

## 1.1 问题：元数据放哪、怎么组织才能又快又稳
（三连问：元数据服务的三种架构——外置元数据库（HMS 式，网络往返+单点）
 vs 分布式 KV（每次访问都有 RPC）vs 全内存镜像+日志复制（快，但受限于
 单机内存与回放时长）；Doris 存算一体选全内存的考量与代价——
 这个代价正是 part1 第 4 章讲过的存算分离动机之一，交代呼应）

## 1.2 源码走读：从 Env 到 Tablet 的内存对象树
（Env→CatalogMgr→InternalCatalog（idToDb/fullNameToDb 双索引）→
 Database→OlapTable→…（层级本身 part1 第 3 章讲过，引用不重复；
 本章重点是**内存组织方式**：并发容器选型、id 与 name 双索引为什么都要）；
 TabletInvertedIndex 倒排：tablet→BE 的反向查找为什么单独建；
 mermaid 对象关系图（突出索引与锁，不重复层级图）；
 tricky 点：内存元数据的"权威时刻"——只有 Master 的内存是权威，
 非 Master 靠回放追，读到旧值的窗口；
 易错点：tablet 元数据放大（part1 第 3 章伏笔）在 FE 内存的真实占用，
 checkpoint 时的翻倍峰值）

## 1.3 源码走读：锁模型
（Database 级读写锁（核实真实锁类型与获取顺序约定）；
 表级锁与库级锁的层级、锁顺序约定在哪注释/约定；
 tricky 点：大 DDL 持库写锁阻塞查询 plan 的连锁；
 易错点：跨库操作的锁顺序——死锁的经典构造与 Doris 的规避约定）

## 1.4 双模式对比
（分离模式内存里还有什么：CloudEnv 增补的对象、哪些元数据变成了
 MetaService 的缓存视图（版本、tablet 位置）；一句话指向 ch4）

## 1.5 动手实验
（核心点：用 MemoryViewer/SHOW PROC '/mem' 类接口（核实真实入口）
 观察 FE 元数据内存构成；建 1000 分区表前后对比；
 易错点：构造一个高分区数+高分桶数表，观察 FE 内存增长与
 checkpoint 耗时变化——把 part1 的"放大效应"在 FE 侧量化）

## 1.6 排查清单
（症状→路径：FE 内存持续增长 / SHOW 命令变慢（锁等待）/
 FE OOM 后如何用更大堆重启）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/01-catalog-and-memory.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/01-catalog-and-memory.md
git commit -m "[docs] doris-internals part4: ch1 catalog and memory structure

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 2 章《元数据持久化：EditLog、bdbje 与 Checkpoint》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/02-editlog-and-checkpoint.md`

**Interfaces:**
- Consumes: ch1 的内存对象树。
- Produces: 日志+镜像持久化模型，ch3 高可用（日志复制即选主基础）与 part6 元数据故障篇引用。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class EditLog" fe/fe-core/src/main/java/org/apache/doris/persist/EditLog.java
grep -n "logEdit\|logCreateTable" fe/fe-core/src/main/java/org/apache/doris/persist/EditLog.java | head -5
grep -n "loadJournal" fe/fe-core/src/main/java/org/apache/doris/persist/EditLog.java | head -2
grep -n "class BDBJEJournal\|write\|read" fe/fe-core/src/main/java/org/apache/doris/journal/bdbje/BDBJEJournal.java | head -6
grep -n "class BDBEnvironment" fe/fe-core/src/main/java/org/apache/doris/journal/bdbje/BDBEnvironment.java
grep -n "class Checkpoint\|doCheckpoint\|runAfterCatalogReady" fe/fe-core/src/main/java/org/apache/doris/master/Checkpoint.java | head -5
grep -n "loadImage\|saveImage" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java | head -4
ls fe/fe-core/src/main/java/org/apache/doris/persist/meta/
grep -rn "class MetaCleaner" fe/fe-core/src/main/java/org/apache/doris/persist/MetaCleaner.java | head -2
grep -rn "edit_log_roll_num\|checkpoint" fe/fe-common/src/main/java/org/apache/doris/common/Config.java | head -6
grep -rn "class JournalEntity" fe/fe-core/src/main/java/org/apache/doris/journal/JournalEntity.java | head -2
```

- [ ] **Step 2: 写作**

创建 `02-editlog-and-checkpoint.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：元数据持久化 —— EditLog、bdbje 与 Checkpoint

## 2.1 问题：全内存元数据怎么不丢
（三连问：每次变更全量落盘（写放大不可接受）vs 仅 WAL（重启回放无限长）
 vs WAL+定期快照（经典（如 HDFS NN 的 editlog+fsimage）；
 日志存本地文件 vs 存复制状态机（bdbje 的 Replicated Environment）——
 选 bdbje 一石二鸟：既是持久化又是复制/选主基础；代价：黑盒依赖）

## 2.2 源码走读：一次 DDL 的日志之旅
（logEdit 的序列化（JournalEntity/OperationType）→ BDBJEJournal.write →
 bdbje 复制多数派确认；回放侧 loadJournal 的 switch 巨表；
 mermaid 时序：DDL→内存变更→logEdit→复制→非 Master 回放；
 tricky 点：**先改内存还是先写日志**——Doris 的顺序（核实真实顺序）
 与 crash 一致性语义，错序会怎样；
 易错点：OperationType 新增字段的兼容规则（呼应 part1 第 2 章
 gensrc 契约兼容点，FE 侧的对应物））

## 2.3 源码走读：Checkpoint 与镜像
（Checkpoint 线程的触发条件（edit_log_roll_num 等，核实）；
 为什么在独立"影子 Env"里回放到指定 journalId 再 saveImage
 （核实 Checkpoint.java 的真实实现方式）；image 文件格式
 （persist/meta/ 的 Header/Footer/Index）；MetaCleaner 清理旧 image/日志；
 tricky 点：checkpoint 内存翻倍峰值（ch1 伏笔回收）与 OOM 风险参数；
 易错点：image 损坏/版本不兼容时的启动失败样貌，
 元数据兼容性开关（核实 metadata_failure_recovery 等真实配置名））

## 2.4 双模式对比
（分离模式下这一整套还在吗——在（FE 自身状态仍需 bdbje），但"重"的
 tablet/版本元数据已外移 MetaService，image 变小、checkpoint 变轻；
 指向 ch4 的 FDB 持久化）

## 2.5 动手实验
（核心点：建表后用 BDBTool/BDBDebugger（核实真实工具类与启用方式）
 或日志观察 journal id 推进；触发一次 checkpoint（调小 edit_log_roll_num，
 核实 FE 配置可变性）观察 image 生成与旧日志清理；
 易错点：故意 kill -9 Master FE 于两次 checkpoint 之间，观察重启回放
 日志条数与耗时——理解"回放窗口=宕机恢复时长"）

## 2.6 排查清单
（症状→路径：FE 启动慢在回放 / image 目录异常大 / bdbje 磁盘满
 的处置顺序）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/02-editlog-and-checkpoint.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/02-editlog-and-checkpoint.md
git commit -m "[docs] doris-internals part4: ch2 editlog bdbje checkpoint

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 3 章《FE 高可用：选主、角色与故障切换》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/03-fe-ha.md`

**Interfaces:**
- Consumes: ch2 的 bdbje 复制模型。
- Produces: FE HA 机制，part6 FE 故障篇的基础。

- [ ] **Step 1: 核实素材**

```bash
ls fe/fe-core/src/main/java/org/apache/doris/ha/
grep -n "class BDBHA\|fencing\|getLeader" fe/fe-core/src/main/java/org/apache/doris/ha/BDBHA.java | head -5
grep -n "class BDBStateChangeListener\|stateChange" fe/fe-core/src/main/java/org/apache/doris/ha/BDBStateChangeListener.java | head -4
sed -n '20,30p' fe/fe-core/src/main/java/org/apache/doris/ha/FrontendNodeType.java
grep -n "transferToMaster\|transferToNonMaster" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java | head -4
grep -rn "MasterOpExecutor\|forward" fe/fe-core/src/main/java/org/apache/doris/qe/MasterOpExecutor.java | head -3
grep -rn "helper node\|helperNode" fe/fe-core/src/main/java/org/apache/doris/ -r --include=Env.java | head -3
grep -rn "metadata_failure_recovery\|master_sync_policy\|replica_sync_policy" fe/fe-common/src/main/java/org/apache/doris/common/Config.java | head -5
```

- [ ] **Step 2: 写作**

创建 `03-fe-ha.md`（约 6000-8000 字），结构：

```markdown
# 第 3 章：FE 高可用 —— 选主、角色与故障切换

## 3.1 问题：谁说了算，挂了怎么办
（三连问：外置协调器（ZK 式，多一个系统）vs 自带共识（Raft 自研或嵌入库）
 vs 复用日志复制层的选主（bdbje Paxos 系）；Doris 借 bdbje 选主的
 一体化收益与"选主逻辑黑盒"的代价；FOLLOWER 奇数多数派与
 OBSERVER 只读扩展的角色设计——为什么两种角色而不是一种）

## 3.2 源码走读：选主与状态迁移
（BDBHA/BDBStateChangeListener：bdbje 状态回调→ transferToMaster /
 transferToNonMaster（核实 Env 中两条链路做了什么：回放追平、
 启动 Master 独有 daemon、editlog 角色切换）；
 mermaid 状态机：UNKNOWN/MASTER/FOLLOWER/OBSERVER 迁移；
 tricky 点：**新 Master 必须先回放完 backlog 才对外服务**——
 切换窗口内写请求的表现，"切了主但没恢复"的误判；
 易错点：sync_policy 参数（master/replica）对掉电丢日志的影响，
 默认值的取舍（核实默认值））

## 3.3 源码走读：请求转发与读写路径
（非 Master 的写转发（part2 第 1 章 MasterOpExecutor 已提，引用）；
 Observer 的读一致性：回放延迟下读到旧元数据的语义；
 tricky 点：FOLLOWER 数量为什么必须奇数、加减 FOLLOWER 的
 多数派变化——运维中加错节点类型的后果）

## 3.4 双模式对比
（分离模式 FE 仍用 bdbje 做自身 HA，但事务/版本状态在 MetaService，
 切主影响面大幅缩小（part3 第 1 章 1.4 已证，链接引用）；
 多 FE 一致性由"都问 MetaService"兜底）

## 3.5 动手实验
（核心点：3 FE（1 Master+2 Follower）本地拉起（引用 part1 第 5 章环境），
 SHOW FRONTENDS 观察角色；kill Master 观察选主与恢复时长，
 对照 3.2 的状态迁移日志；
 易错点：把 Observer 当 Follower 加进集群，观察多数派不变的事实；
 或 kill 到只剩 1 个 Follower，观察集群只读/不可写的表现）

## 3.6 排查清单
（症状→路径：选不出主（多数派丢失）/ 新主起了但不服务（回放中）/
 元数据两边不一致怀疑脑裂的排查与 metadata_failure_recovery 的
 使用边界（危险操作警示））
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/03-fe-ha.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/03-fe-ha.md
git commit -m "[docs] doris-internals part4: ch3 fe high availability

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 4 章《存算分离元数据：MetaService、FDB 布局与 Recycler》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/04-metaservice-fdb.md`

**Interfaces:**
- Consumes: part1 第 2 章 MetaService 进程结构、part3 第 4/6 章的 commit/job-lock 结论。
- Produces: 分离模式元数据全景，part5 第 6 章与 part6 引用。

- [ ] **Step 1: 核实素材**

```bash
ls cloud/src/meta-service/ cloud/src/meta-store/ cloud/src/recycler/ cloud/src/resource-manager/
grep -n "instance_key\|meta_tablet_key\|partition_version_key\|txn_label_key" cloud/src/meta-store/keys.h | head -10
sed -n '1,80p' cloud/src/meta-store/keys.h   # key 空间注释块（若有）
grep -n "class TxnKv\|class FdbTxnKv" cloud/src/meta-store/txn_kv.h | head -3
grep -n "class Recycler\|recycle_instance\|recycle_tablet\|recycle_rowsets" cloud/src/recycler/recycler.h cloud/src/recycler/recycler.cpp | head -8
grep -n "class Checker\|inverted check" cloud/src/recycler/checker.h | head -3
grep -n "class ResourceManager" cloud/src/resource-manager/resource_manager.h | head -2
grep -rn "class CloudClusterChecker" fe/fe-core/src/main/java/org/apache/doris/cloud/catalog/CloudClusterChecker.java | head -2
grep -rn "getCurrentClusterNames\|addCluster" fe/fe-core/src/main/java/org/apache/doris/cloud/system/CloudSystemInfoService.java | head -4
grep -rn "fdb" cloud/script/*.sh 2>/dev/null | head -3; ls cloud/script/ 2>/dev/null | head -5
```

- [ ] **Step 2: 写作**

创建 `04-metaservice-fdb.md`（约 6000-8000 字），结构：

```markdown
# 第 4 章：存算分离元数据 —— MetaService、FDB 布局与 Recycler

## 4.1 问题：把元数据搬出 FE 之后放哪、怎么放
（三连问：自研元数据存储 vs MySQL 系（事务但难横向扩）vs
 分布式事务 KV（FDB：严格串行化、水平扩展，但运维一个新系统）；
 为什么是 FDB 而不是 etcd/ZK（数据量与事务模型）；
 MetaService 作为无状态转换层的定位——FE/BE 不直接说 FDB 话）

## 4.2 源码走读：FDB key 空间布局（本章重点段）
（meta-store/keys.h 的 key 编码体系：前缀分类（instance/txn/version/
 meta/stats/job/recycle…，按真实注释与函数清单归纳成表）；
 codec 的有序编码为什么重要（范围扫描）；
 一次 get_version/commit_txn 触碰哪些 key（用 part3 第 4 章七步
 对照，链接引用不重复）；
 tricky 点：FDB 事务限制（10s/10MB，核实 MetaService 里对应的
 规避代码如分批/lazy commit——part3 第 4 章 TXN_BYTES_TOO_LARGE
 伏笔回收）；
 易错点：直接改 FDB 数据的危险性；版本号 key 的单调性依赖）

## 4.3 源码走读：Recycler 与 Checker
（为什么需要独立回收：对象存储没有"引用计数"，删除=先标记
 （recycle key）后异步清理；recycler.cpp 的回收循环与分类
 （索引/分区/tablet/rowset/事务标记，按真实函数）；
 checker.cpp 的正逆向校验（数据↔元数据一致性）；
 tricky 点：回收窗口参数与"误删可恢复窗口"的关系；
 易错点：Recycler 停摆的后果——对象存储费用只涨不跌，
 怎么监控它在干活（核实指标/日志））

## 4.4 源码走读：计算组与节点管理
（ResourceManager（MS 侧）与 CloudClusterChecker/CloudSystemInfoService
 （FE 侧缓存视图）的同步机制；加减 BE/计算组的元数据流；
 tricky 点：FE 视图滞后于 MS 的窗口，对 part2 第 5 章 5.4 副本
 选择的影响（链接引用））

## 4.5 动手实验
（核心点：若有 FDB 环境（引用 cloud/script 的本地拉起方式，核实脚本名），
 用 MetaService 的 http 接口（core 里的 http_encode_key/
 meta_service_http，核实真实路径）查一个 tablet 的 meta key；
 无环境则给"纸上实验"：给定建表参数手推 key 布局；
 易错点：观察 recycle key 的产生与清理（drop 一张表后跟踪），
 或核实回收窗口配置并解释误删恢复操作顺序）

## 4.6 排查清单
（症状→路径：MS 报 KV_TXN_CONFLICT 高频（part3 第 4 章伏笔）/
 对象存储用量与表大小对不上（Recycler 检查）/ FE 看到的计算组
 与实际不符（缓存滞后 vs MS 权威））
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/04-metaservice-fdb.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/04-metaservice-fdb.md
git commit -m "[docs] doris-internals part4: ch4 metaservice fdb recycler

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 5 章《调度体系：Tablet 均衡、副本修复与计算组管理》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/05-scheduling.md`

**Interfaces:**
- Consumes: ch1 的 TabletInvertedIndex、part1 第 3 章副本概念。
- Produces: 调度/修复机制，part6 副本故障篇的基础。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class TabletChecker\|runAfterCatalogReady" fe/fe-core/src/main/java/org/apache/doris/clone/TabletChecker.java | head -3
grep -n "class TabletScheduler\|schedulePendingTablets\|handleTabletByTypeAndStatus" fe/fe-core/src/main/java/org/apache/doris/clone/TabletScheduler.java | head -5
grep -n "class TabletSchedCtx\|Priority" fe/fe-core/src/main/java/org/apache/doris/clone/TabletSchedCtx.java | head -4
grep -n "class BeLoadRebalancer\|class DiskRebalancer\|class PartitionRebalancer" fe/fe-core/src/main/java/org/apache/doris/clone/*.java | head -4
grep -rn "TabletStatus" fe/fe-core/src/main/java/org/apache/doris/catalog/Tablet.java | head -5
grep -rn "clone task" fe/fe-core/src/main/java/org/apache/doris/clone/TabletSchedCtx.java | head -3
grep -rn "max_scheduling_tablets\|balance_load_score_threshold\|disable_balance" fe/fe-common/src/main/java/org/apache/doris/common/Config.java | head -5
grep -rn "class CloudTabletRebalancer" fe/fe-core/src/main/java/org/apache/doris/cloud/ -r | head -2
ls fe/fe-core/src/main/java/org/apache/doris/cloud/catalog/ | grep -i "rebalanc\|warmup"
```

- [ ] **Step 2: 写作**

创建 `05-scheduling.md`（约 6000-8000 字），结构：

```markdown
# 第 5 章：调度体系 —— Tablet 均衡、副本修复与计算组管理

## 5.1 问题：几十万 tablet 谁来看护
（三连问：不管（坏一个少一个）vs 全量周期巡检（扫不过来）vs
 事件驱动+分级队列巡检（TabletChecker 发现 + TabletScheduler 调度，
 优先级+限流防打爆）；修复与均衡共用一条调度管道的设计取舍）

## 5.2 源码走读：发现与调度
（TabletChecker 巡检→健康状态判定（Tablet 的 TabletStatus 枚举，
 核实真实状态集）→TabletSchedCtx 入队（Priority 分级）→
 TabletScheduler 的调度循环（配额、去重、超时）；
 mermaid 流程图：从"副本缺失"到"clone 任务下发到 BE"；
 tricky 点：调度限流参数（max_scheduling_tablets 等）——调太小
 修复慢、调太大打爆集群的两难；
 易错点：REPLICA_MISSING vs VERSION_INCOMPLETE 等状态的
 处置差异，看错状态修错方向）

## 5.3 源码走读：三种均衡器
（BeLoadRebalancer（BE 间负载）/DiskRebalancer（盘间）/
 PartitionRebalancer（分区打散）的分工与触发条件；
 均衡与修复抢配额的协调；
 tricky 点：均衡的"移动代价"——大 tablet 搬迁对集群的冲击，
 disable_balance 的使用场景；
 易错点：colocate 表的均衡约束（ColocateTableCheckerAndBalancer），
 违反 colocate 的搬迁为什么被禁止）

## 5.4 双模式对比（本章重点段）
（分离模式没有副本修复（对象存储兜底）也没有数据搬迁式均衡，
 对应物是：CloudTabletRebalancer 的 tablet→BE 映射再均衡
 （只动 cache 亲和性映射不动数据，核实真实类与机制）、
 计算组加减节点的 warmup（核实 CloudWarmUpJob 类）；
 一张表对照：故障形态/均衡对象/搬的是什么/代价）

## 5.5 动手实验
（核心点：3 BE 集群建三副本表，kill 一个 BE 超过
 tablet_repair_delay_factor（核实参数）观察修复任务产生、
 SHOW PROC '/cluster_health' 或 tablet health 相关 proc（核实路径）；
 易错点：故意把调度配额调小再制造批量副本缺失，观察修复排队
 与优先级；恢复后清理）

## 5.6 排查清单
（症状→路径：副本长期不修复（配额/黑名单/磁盘满）/ 均衡风暴
 打满网络 / colocate 表 unstable 的定位）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/05-scheduling.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/05-scheduling.md
git commit -m "[docs] doris-internals part4: ch5 tablet scheduling and balancing

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 6 章《外部数据源：Catalog 联邦查询架构概览》

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/06-external-catalog.md`

**Interfaces:**
- Consumes: ch1 的 CatalogMgr/CatalogIf 骨架。
- Produces: 联邦元数据认知（概览级，执行细节不展开——设计文档定位为"概览"）。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class CatalogMgr\|createCatalog" fe/fe-core/src/main/java/org/apache/doris/datasource/CatalogMgr.java | head -4
grep -n "interface CatalogIf" fe/fe-core/src/main/java/org/apache/doris/datasource/CatalogIf.java
grep -n "class ExternalCatalog" fe/fe-core/src/main/java/org/apache/doris/datasource/ExternalCatalog.java
grep -n "class CatalogFactory" fe/fe-core/src/main/java/org/apache/doris/datasource/CatalogFactory.java
ls fe/fe-core/src/main/java/org/apache/doris/datasource/hive/ | head -8
grep -rn "class HMSExternalCatalog" fe/fe-core/src/main/java/org/apache/doris/datasource/hive/ | head -2
ls fe/fe-core/src/main/java/org/apache/doris/datasource/iceberg/ | head -5
grep -rn "metadata cache\|MetaCache\|invalidate" fe/fe-core/src/main/java/org/apache/doris/datasource/ExternalCatalog.java | head -4
grep -rln "class ExternalMetaCacheMgr" fe/fe-core/src/main/java/org/apache/doris/datasource/ | head -2
grep -rn "REFRESH CATALOG" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -2
```

- [ ] **Step 2: 写作**

创建 `06-external-catalog.md`（约 5000-7000 字，概览定位），结构：

```markdown
# 第 6 章：外部数据源 —— Catalog 联邦查询架构概览

## 6.1 问题：别人的元数据怎么为我所用
（三连问：ETL 搬进来（延迟+存储翻倍）vs 专用连接器逐个实现
 （N 种源×M 个入口的组合爆炸）vs 统一 Catalog 抽象+按源实现
 （CatalogIf 接口族）；元数据拿过来之后的第二个问题：每次查询
 现拉（慢）vs 缓存（一致性）——缓存+失效策略的取舍）

## 6.2 源码走读：Catalog 抽象与注册
（CatalogIf/ExternalCatalog/CatalogFactory 的类型注册；
 InternalCatalog 与 External 的统一入口（ch1 呼应）；
 CREATE CATALOG 的落库与 editlog（ch2 呼应）；
 挑 HMSExternalCatalog 做代表走一遍：连接→库表懒加载；
 tricky 点：外部表对象的懒加载与占位——第一次访问的抖动）

## 6.3 源码走读：元数据缓存与失效
（ExternalMetaCacheMgr（核实真实类名）的缓存层级（catalog/db/table/
 partition/file 各级，按真实实现归纳）；REFRESH CATALOG/TABLE 的
 失效粒度；自动刷新参数；
 tricky 点：缓存不一致的典型表象——外部新增分区查不到、
 schema 变了报错，与失效粒度的对应；
 易错点：大库全量 REFRESH 的代价）

## 6.4 双模式对比
（联邦元数据层两模式一致（都在 FE），一句注明；file cache 对外表
 数据读的作用引用 part2 第 7 章）

## 6.5 动手实验
（核心点：起本地 HMS 或用 JDBC catalog（择可行者，核实 JDBC catalog
 类存在性）建外部 catalog，观察懒加载与缓存命中日志；
 易错点：外部侧新增分区/改 schema，不 REFRESH 观察旧数据，
 REFRESH 后对比——亲手踩缓存一致性坑）

## 6.6 排查清单
（症状→路径：外部表查不到新数据（缓存）/ 连接失败（凭证/网络分层
 排查）/ 外部权限变更不生效）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part4-fe-internals/06-external-catalog.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/06-external-catalog.md
git commit -m "[docs] doris-internals part4: ch6 external catalog federation

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 7: 部分目录页 + 系列 README 翻转

**Files:**
- Create: `docs/doris-internals/part4-fe-internals/README.md`
- Modify: `docs/doris-internals/README.md`（第四部分状态翻转+章节改链接，免责声明下移到 part4 之后）
- Modify: `docs/doris-internals/part3-load-lifecycle/README.md`（下一部分预告改为直链 ../part4-fe-internals/README.md）

**Interfaces:**
- Consumes: Task 1-6 产出的 6 个章节文件。
- Produces: 完整可导航的第四部分。

- [ ] **Step 1: 核实现状**

```bash
ls docs/doris-internals/part4-fe-internals/
grep -n "第四部分" docs/doris-internals/README.md
grep -n "第四部分\|part4" docs/doris-internals/part3-load-lifecycle/README.md
```

- [ ] **Step 2: 写作与修改**

part4 README（约 500 字，仿 part2/part3 目录页：导语 + 6 行表格（章节/主题/关键收获，关键收获必须反映各章真实内容且不得与章内修正结论矛盾）+ 下一部分预告）；系列 README 最小化修改；part3 README 预告改直链。

- [ ] **Step 3: 全量校验**

三个文件引用校验 + 全树死链检查（脚本同前，预期无输出）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part4-fe-internals/README.md \
        docs/doris-internals/README.md \
        docs/doris-internals/part3-load-lifecycle/README.md
git commit -m "[docs] doris-internals part4: part index, series index flip to complete

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：最终整分支审查（fable 模型，含台账 Minor triage、回引真实性专项 grep、fix-later 清单处置），修复确认后 push，向作者简报第四部分完成情况，继续第五部分计划制定。
