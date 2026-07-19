# 《Doris 内核透视》第一批交付物实施计划（总 README + 第一部分）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成教程系列总目录 `docs/doris-internals/README.md` 与第一部分（全局架构与设计哲学）全部 5 章及部分目录页。

**Architecture:** 纯文档写作项目。每章按设计文档（`docs/superpowers/specs/2026-07-19-doris-internals-tutorial-design.md`）第 5 节的五段式骨架写作；每个任务 = 一个章节文件，流程固定为"核实素材 → 写作 → 引用校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid 图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束来自设计文档，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（严禁泛泛而谈）：遇到了什么问题？→ 有哪些候选方案、各有什么优劣？→ Doris 最终怎么考量和解决的？
- **源码走读分清主次**：显而易见的高度概括；不易理解的详细逐段解释；重点挖掘易错点/tricky 点并解释"为什么这么写、错写会怎样"。
- **动手实验双目的**：验证核心点 + 主动踩一遍易错点。
- 双模式并重：每章交代存算一体与存算分离的路径差异（个别章节可裁剪，需在章内注明或在计划中说明理由）。
- 所有源码引用格式为 `路径:行号` 或 `路径`，必须在当前 master 真实存在；**先核实再落笔，禁止凭记忆写引用**。
- 已核实的关键锚点（写作时直接可用，行号需现场核实）：
  - FE 入口：`fe/fe-core/src/main/java/org/apache/doris/DorisFE.java`；FE 全局单例：`fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java`
  - BE 入口：`be/src/service/doris_main.cpp`；存储引擎：`be/src/storage/storage_engine.cpp`
  - MetaService 入口：`cloud/src/main.cpp`（`int main` 在约 170 行，需复核）
  - 数据模型枚举：`gensrc/proto/olap_file.proto:331-333`（`DUP_KEYS`/`UNIQUE_KEYS`/`AGG_KEYS`）
  - 本 master BE 目录已重构：存储在 `be/src/storage/`（无 `be/src/olap/`）、Pipeline 在 `be/src/exec/pipeline/`、Block/Column/DataType 在 `be/src/core/`、Rowset 在 `be/src/storage/rowset/`、Segment 读写在 `be/src/storage/segment/`、BE 侧云逻辑在 `be/src/cloud/`
  - MetaService 子模块：`cloud/src/meta-service/`、`cloud/src/meta-store/`、`cloud/src/recycler/`、`cloud/src/resource-manager/`、`cloud/src/snapshot/`
- 构建/测试命令必须与根 `AGENTS.md` 一致：`./build.sh --be --fe`（默认 ASAN）、`run-be-ut.sh`、`run-fe-ut.sh`、`run-regression-test.sh`（regression 用 `-d 父目录 -s 用例名`）。
- 每完成一个文件即提交，提交信息前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行（见任务内命令）。
- 每个写作任务完成后执行统一的**引用校验步骤**：

```bash
# 在仓库根执行；FILE 为本任务产出的 md 文件
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

预期输出为空（无 MISSING 行）。出现 MISSING 必须修正后重跑，直至为空再提交。

---

### Task 1: 系列总目录 `docs/doris-internals/README.md`

**Files:**
- Create: `docs/doris-internals/README.md`

**Interfaces:**
- Produces: 七个部分目录的相对链接约定（`part1-architecture/` … `part7-case-studies/`），后续所有部分目录页需与此一致；第一部分各章文件名约定（见 Task 2-6 的 Create 路径），总 README 中的第一部分章节链接必须与之完全一致。

- [ ] **Step 1: 核实素材**

阅读以下内容获取写作素材（只读，不改）：

```bash
head -80 AGENTS.md          # 构建/测试规范，写"如何使用本教程"时引用
ls docs/doris-internals/ 2>/dev/null   # 确认目录尚不存在或为空
```

并重读设计文档 `docs/superpowers/specs/2026-07-19-doris-internals-tutorial-design.md` 第 1、2、4、5、6 节。

- [ ] **Step 2: 写作**

创建 `docs/doris-internals/README.md`，内容结构（H2 级标题，总篇幅约 2000-3000 字）：

```markdown
# Doris 内核透视：从请求路径到源码实现

（引言：本教程是什么、不是什么；目标读者=有大数据经验的工程师；
 读完的能力标准 —— 复述设计文档第 1 节的四条目标）

## 这套教程怎么读（阅读地图）
（三种读法：顺序精读 / 按请求路径跳读 / 按故障场景查阅；
 用 mermaid flowchart 画出七个部分的依赖关系：
 part1 是所有部分的基础；part2/part3 依赖 part1；
 part4/part5 深化 part2/part3；part6 依赖前五部分；part7 独立可插叙）

## 全系列目录
（七个部分的表格：部分名 | 一句话主题 | 状态（已完成/写作中/规划中）；
 第一部分列出 5 章的相对链接，指向 part1-architecture/ 下各文件；
 第 2~7 部分标注"规划中"并给出规划章节名列表 —— 从设计文档第 6 节复制）

## 环境准备（速览）
（一段说明：本教程实操以源码编译调试为主；
 给出最小命令集：git clone、./build.sh --be --fe、BUILD_TYPE=ASAN 说明、
 三个 UT/回归脚本名字；
 注明详细环境搭建在第一部分第 5 章，给相对链接 part1-architecture/05-source-map-and-dev-env.md）

## 约定
（代码引用格式 `路径:行号` 基于 master 某 commit（写明当前 HEAD 短 hash，
 用 git rev-parse --short HEAD 获取）；
 术语约定：FE/BE/MS(MetaService)、存算一体(shared-nothing)/存算分离(shared-storage/cloud mode)；
 图例约定：mermaid 时序图中参与者命名规则）
```

硬性要求：目录中第一部分 5 章链接的文件名必须与 Task 2-6 的 Create 路径逐字一致；"状态"列如实标注（此时 5 章尚未写完则标"写作中"，在 Task 6 完成后回改为"已完成"）。

- [ ] **Step 3: 引用校验**

运行 Global Constraints 中的引用校验脚本（FILE=docs/doris-internals/README.md）。预期：无 MISSING。
另外校验相对链接：

```bash
cd docs/doris-internals && grep -oE '\]\(([^)#]+)' README.md | tr -d '](' \
  | grep -v '^http' | sort -u | while read -r p; do [ -e "$p" ] || echo "DEAD LINK: $p"; done
```

此时 part1 各章尚未创建，允许出现指向 part1-architecture/ 的 DEAD LINK（Task 6 末尾会复检清零）；其余 DEAD LINK 必须修复。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/README.md
git commit -m "[docs] doris-internals: add series index and reading map

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 1 章《Doris 是什么样的系统》

**Files:**
- Create: `docs/doris-internals/part1-architecture/01-positioning.md`

**Interfaces:**
- Produces: 术语首次定义（MPP、列存、实时分析、明细/聚合场景），后续章节直接使用不再解释。

- [ ] **Step 1: 核实素材**

```bash
head -40 README.md                      # 官方定位表述
grep -rn "MPP" README.md | head -3
ls be/src/exec/pipeline | head          # 确认 pipeline 执行引擎存在，佐证 MPP 并行执行
ls fe/fe-core/src/main/java/org/apache/doris/nereids | head
```

- [ ] **Step 2: 写作**

创建 `01-positioning.md`（约 4000-6000 字），结构：

```markdown
# 第 1 章：Doris 是什么样的系统

## 1.1 问题：实时分析场景到底难在哪
（三连问之一。具体化问题：高并发点查+大宽表聚合+实时写入可见+更新需求
 同时存在时，传统方案的矛盾 —— 用具体数字描述典型场景约束）

## 1.2 候选方案与权衡
（三连问之二。至少对比四类方案的优劣：
 a. Hadoop 系离线数仓（Hive/Spark）：吞吐大、延迟分钟级、无点查能力
 b. 预计算系（Druid/Kylin）：查询快但灵活性差、维度爆炸
 c. ClickHouse 单机极致列存：单表快、join 弱（当年）、运维分片手工
 d. MPP 数仓（Greenplum 类）：SQL 完整、实时写入弱
 每类给出"它解决了什么、代价是什么"）

## 1.3 Doris 的取舍
（三连问之三。Doris 的答案：MPP 架构 + 列存 + 自研存储引擎 + MySQL 协议
 + 前后端分离的两进程模型；每个选择对应 1.2 中哪个痛点；
 也要讲放弃了什么：例如不做事务型 OLTP、单行事务能力有限等）

## 1.4 从取舍到代码：顶层结构印证
（源码走读-概览级：用 3 个证据把上述取舍落到仓库：
 fe/ 是 Java 的分析/元数据层（指出 nereids/ 与 catalog/ 目录）、
 be/ 是 C++ 向量化执行+存储（指出 exec/pipeline/ 与 storage/）、
 gensrc/ 是两者之间的 thrift/proto 契约（指出 gensrc/proto/、gensrc/thrift/）；
 本章不深入代码，只建立"目录-职责"映射）

## 1.5 本章小结与自查问题
（3-5 个自测问题，如"为什么 Doris 选择 FE Java + BE C++ 的双语言结构"）
```

裁剪说明（写入章内脚注）：本章为定位章，无独立"动手实验/排查清单/双模式对比"段；双模式差异在第 4 章专章展开。

- [ ] **Step 3: 引用校验**

运行引用校验脚本（FILE=docs/doris-internals/part1-architecture/01-positioning.md）。预期：无 MISSING。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part1-architecture/01-positioning.md
git commit -m "[docs] doris-internals part1: ch1 positioning and trade-offs

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 2 章《三大件解剖：FE / BE / MetaService》

**Files:**
- Create: `docs/doris-internals/part1-architecture/02-three-components.md`

**Interfaces:**
- Consumes: 第 1 章的术语定义。
- Produces: 进程模型图与"FE=Env、BE=ExecEnv+StorageEngine、MS=MetaServiceImpl"的核心对象锚点，供 part2/part3 直接引用。

- [ ] **Step 1: 核实素材**

逐个打开并读懂以下入口（记录准确行号用于引用）：

```bash
# FE 启动主线
grep -n "public static void main\|start(" fe/fe-core/src/main/java/org/apache/doris/DorisFE.java | head
# Env 单例及其初始化
grep -n "class Env\|private static class SingletonHolder\|public void initialize" fe/fe-core/src/main/java/org/apache/doris/catalog/Env.java | head
# BE 启动主线
grep -n "int main" be/src/service/doris_main.cpp
grep -n "class ExecEnv" be/src/runtime/exec_env.h | head
grep -n "class StorageEngine" be/src/storage/storage_engine.h | head
# MetaService 启动与服务注册
grep -n "int main" cloud/src/main.cpp
ls cloud/src/meta-service/ | head -20
# FE-BE 之间的 RPC 契约
ls gensrc/thrift/ | head -20; ls gensrc/proto/ | head -20
# 心跳机制
grep -rn "HeartbeatFlags\|heartbeat" be/src/agent/ --include=*.h -l | head -3
find fe/fe-core/src/main/java/org/apache/doris/system -name "*.java" | head
```

- [ ] **Step 2: 写作**

创建 `02-three-components.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：三大件解剖 —— FE、BE 与 MetaService

## 2.1 问题：一个分布式数据库的职责应该怎么切
（三连问：元数据管理/查询规划/执行/存储四类职责如何划分进程？
 候选：单进程一体（CK 早期）/ 计算存储同进程+独立元数据（Doris 存算一体）/
 全部微服务化（BigQuery 类）；各方案在部署复杂度、故障域、扩展性上的优劣；
 Doris 的选择及理由：FE 管元数据+规划（Java 生态成熟），BE 管执行+存储（C++ 性能），
 存算分离下再拆出 MetaService）

## 2.2 FE 进程解剖
（源码走读：从 DorisFE.java 的 main 出发，概括启动流程 —— 加载配置、
 初始化 Env 单例、启动三类服务端口（MySQL 协议/thrift RPC/HTTP）；
 Env 是理解 FE 的钥匙：贴 Env 中最关键的十来个成员（catalog、editLog、
 各类 Mgr）片段并逐一说明职责；
 tricky 点：Env 单例的初始化顺序问题、FE 角色（Master/Follower/Observer）
 对可写性的影响在代码里如何体现）

## 2.3 BE 进程解剖
（源码走读：doris_main.cpp 启动流程概括 —— 端口、ExecEnv 初始化、
 StorageEngine 打开、心跳服务；
 ExecEnv 与 StorageEngine 的职责边界：执行时资源 vs 存储生命周期；
 tricky 点：BE 无自主元数据，一切听 FE 调度 —— 从心跳（master 信息下发）
 与 agent task（be/src/agent/）机制说明"FE 是大脑、BE 是手脚"的实现方式）

## 2.4 MetaService 进程解剖（存算分离）
（源码走读：cloud/src/main.cpp 启动流程；meta-service/ 的 brpc 服务结构；
 meta-store/ 对 FoundationDB 的封装层次；recycler 的角色；
 双模式对比总起：存算一体下"FE 就是元数据服务"，存算分离下元数据外移的动机
 —— FE 内存元数据+bdbje 的容量/弹性瓶颈）

## 2.5 三者如何对话：RPC 契约
（gensrc/thrift/（FE↔BE 控制面）与 gensrc/proto/（数据面+MS 接口）的分工；
 挑 FrontendService.thrift、BackendService.thrift、cloud.proto 各 1-2 个
 代表性接口说明；
 易错点：thrift/proto 改动的兼容性纪律 —— 为什么字段只加不改号）

## 2.6 动手实验
（核心点实验：单机拉起 FE+BE（按 AGENTS.md 规范 build + 启动 + add backend），
 用 SHOW FRONTENDS / SHOW BACKENDS 观察心跳；在 fe.log/be.INFO 里找到
 一次心跳往返的日志证据；
 易错点实验：故意把 BE conf 的 priority_networks 配错网段，观察
 add backend 后心跳失败的日志形态 —— 这是真实部署中最高频的坑之一）

## 2.7 排查清单
（症状→路径：BE 加不进集群 / 心跳丢失 / FE 启动卡住 各自的定位步骤与关键日志）
```

- [ ] **Step 3: 引用校验**

运行引用校验脚本（FILE=docs/doris-internals/part1-architecture/02-three-components.md）。预期：无 MISSING。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part1-architecture/02-three-components.md
git commit -m "[docs] doris-internals part1: ch2 FE/BE/MetaService anatomy

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 3 章《数据模型：从 Table 到 Segment》

**Files:**
- Create: `docs/doris-internals/part1-architecture/03-data-model.md`

**Interfaces:**
- Consumes: 第 2 章的核心对象锚点。
- Produces: Table→Partition→Tablet→Rowset→Segment 层级图与术语，part3/part5 直接引用。

- [ ] **Step 1: 核实素材**

```bash
# 层级对应的 FE 类
ls fe/fe-core/src/main/java/org/apache/doris/catalog/ | grep -i "OlapTable\|Partition\|Tablet\|MaterializedIndex"
# 层级对应的 BE 类与元数据 proto
grep -n "message TabletMetaPB\|message RowsetMetaPB" gensrc/proto/olap_file.proto
sed -n '325,340p' gensrc/proto/olap_file.proto        # KeysType 枚举
ls be/src/storage/tablet/ | head; ls be/src/storage/rowset/ | head
ls be/src/storage/segment/ | head
# 分桶与分区
grep -rn "class HashDistributionInfo\|class RandomDistributionInfo" fe/fe-core/src/main/java/org/apache/doris/catalog/ | head
grep -rn "class PartitionInfo" fe/fe-core/src/main/java/org/apache/doris/catalog/PartitionInfo.java | head -2
```

- [ ] **Step 2: 写作**

创建 `03-data-model.md`（约 6000-8000 字），结构：

```markdown
# 第 3 章：数据模型 —— 从 Table 到 Segment

## 3.1 问题：海量数据怎么切才能又好写又好查
（三连问：切分要同时服务于分布式并行、副本容错、导入原子性、查询裁剪；
 候选：按行 range 切（HBase region）/一致性哈希/固定分片（ES shard）/
 两级切分（分区+分桶）；各自在均衡、裁剪、扩容上的优劣；
 Doris 选两级切分的理由：分区管生命周期与裁剪、分桶管并行与均衡）

## 3.2 逻辑层级：Table→Partition→MaterializedIndex→Tablet
（源码走读：FE catalog 中 OlapTable/Partition/MaterializedIndex/Tablet 类
 的嵌套关系，配 mermaid 类图；
 tricky 点：MaterializedIndex 这一层为什么存在 —— 同步物化视图/rollup 与
 base 表共享 Partition 的设计，初学者最容易忽略的一层；
 易错点：Tablet 数量 = 分区数×分桶数×副本数 的放大效应，tablet 过多
 打爆 FE 内存与调度的真实事故模式）

## 3.3 物理层级：Tablet→Rowset→Segment
（源码走读：TabletMetaPB/RowsetMetaPB 关键字段逐段讲解（版本区间、
 rowset 状态）；BE 侧 tablet/ 与 rowset/ 目录的类对应；
 Segment 文件在磁盘上的组织（路径规则）；
 版本机制引入：rowset 的 [start_version, end_version] —— 这是理解
 导入可见性与 compaction 的地基，务必讲透"为什么用版本区间而不是单版本号"）

## 3.4 三种数据模型：Duplicate / Unique / Aggregate
（三连问：同一份存储引擎如何同时支持明细、可更新、预聚合三类需求？
 候选：三套引擎（如某些系统）vs 一套 LSM 变体+读时合并语义参数化；
 源码走读：olap_file.proto 的 KeysType 枚举（DUP_KEYS/UNIQUE_KEYS/AGG_KEYS，
 gensrc/proto/olap_file.proto:331-333）如何一路影响读写路径（概览级，
 详细读写路径留给 part3/part5 并给出前向链接）；
 易错点：Unique 模型 MoR 与 MoW 的语义差异、Aggregate 模型
 count(*) 结果与导入行数不一致的经典困惑）

## 3.5 双模式对比
（层级结构在两种模式下完全一致，但 Tablet 的"归属"不同：存算一体
 tablet 数据在 BE 本地盘、元数据在 TabletMeta（本地 RocksDB）；
 存算分离 tablet 数据在对象存储、元数据在 MetaService(FDB)，BE 只有 cache；
 指出 be/src/cloud/ 下 CloudTablet 相关类与 be/src/storage/tablet/ 的对应）

## 3.6 动手实验
（核心点：建一张 2 分区×2 分桶×1 副本表，通过 SHOW TABLETS / 
 information_schema 观察 tablet 分布；导入一批数据后在 BE 数据目录里
 找到对应 segment 文件，对照 3.3 的路径规则；
 易错点：建一张分桶数过大的表，观察 tablet 元数据量；用 ADMIN SHOW 
 命令观察 tablet 健康状态）

## 3.7 排查清单
（症状→路径：tablet 过多导致 FE 慢 / 查询只命中部分分区（裁剪失效）/ 
 版本数过多报错 -235 的定位入口）
```

- [ ] **Step 3: 引用校验**

运行引用校验脚本（FILE=docs/doris-internals/part1-architecture/03-data-model.md）。预期：无 MISSING。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part1-architecture/03-data-model.md
git commit -m "[docs] doris-internals part1: ch3 data model hierarchy

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 4 章《两种架构形态：存算一体 vs 存算分离》

**Files:**
- Create: `docs/doris-internals/part1-architecture/04-two-architectures.md`

**Interfaces:**
- Consumes: 第 2 章进程模型、第 3 章数据层级。
- Produces: 双模式差异总表（后续每章"双模式对比"段的总纲，各章从此表展开细节）。

- [ ] **Step 1: 核实素材**

```bash
ls cloud/src/ ; ls be/src/cloud/ | head -30
ls fe/fe-core/src/main/java/org/apache/doris/cloud/ | head -20
# 模式开关
grep -rn "isCloudMode\|deploy_mode" fe/fe-core/src/main/java/org/apache/doris/common/Config.java | head -5
grep -rn "is_cloud_mode\|cloud_unique_id" be/src/common/config.h be/src/common/config.cpp 2>/dev/null | head -5
# File Cache
find be/src -type d -name "*file_cache*" -o -type d -name "*cache*" | head -5
# 计算组/集群概念
grep -rln "ComputeGroup\|cluster" fe/fe-core/src/main/java/org/apache/doris/cloud/ | head -5
```

- [ ] **Step 2: 写作**

创建 `04-two-architectures.md`（约 6000-8000 字），结构：

```markdown
# 第 4 章：两种架构形态 —— 存算一体与存算分离

## 4.1 问题：本地盘架构的天花板
（三连问：存算一体在弹性伸缩、成本、多负载隔离上的具体瓶颈 —— 
 扩容要迁数据、冷数据占贵盘、大查询打满 IO 影响导入；
 用具体场景量化痛点）

## 4.2 候选方案与权衡
（候选：a. 冷热分层（数据仍归 BE 管）；b. 彻底存算分离（数据归对象存储、
 元数据外移、计算无状态化）；c. 折中的共享存储多写；
 各自的改造代价、一致性难度、性能天花板；
 Doris 两条都做过 —— 冷热分层（cooldown）与存算分离的关系与取舍）

## 4.3 存算分离的总体设计
（架构图（mermaid）：FE + 多计算组 BE + MetaService(FDB) + 对象存储；
 四个关键改变逐一讲动机：元数据从 FE 内存+bdbje 移到 FDB；数据从本地盘
 移到对象存储+File Cache；副本从 3 副本变 1 逻辑副本（对象存储自带冗余）；
 BE 变成近似无状态、按计算组隔离负载）

## 4.4 代码层面的双模式共存
（源码走读，重点讲"一份代码两种模式"的组织方式：
 FE：fe/fe-core/.../cloud/ 包下的 Cloud* 子类覆盖存算一体基类的关键行为，
 挑 1-2 对类做对照（如事务管理的 cloud 实现 vs 本地实现）；
 BE：be/src/cloud/ 的 CloudStorageEngine/CloudTablet 与 
 be/src/storage/ 的 StorageEngine/Tablet 的继承/平行关系；
 tricky 点：模式判断散布在代码中的形式（Config.isCloudMode 等）、
 读代码时如何快速判断"这段逻辑在哪种模式下生效" —— 给出可操作的判别法；
 易错点：只看基类逻辑就下结论，而 cloud 子类悄悄覆盖了行为）

## 4.5 双模式差异总表
（一张大表：维度 × 两模式 —— 元数据存储/事务提交点/副本机制/数据文件位置/
 缓存层/扩缩容方式/Compaction 执行者/典型故障形态；
 每行注明"详见第 X 章"，作为全书双模式对比段的索引）

## 4.6 动手实验
（核心点：读 FE 启动代码，跟踪 isCloudMode 的判定链路（Config→Env 初始化
 分支），在 IDE/grep 中数一数 Cloud* 类的数量感受改造面；
 说明：本章实验为纯代码考察，存算分离集群的完整拉起（需 FDB+对象存储）
 成本高，放到 part4 第 4 章的 MetaService 实验中再做，此处注明）

## 4.7 排查清单
（先判模式再排障：拿到一个陌生集群如何 30 秒判断它是哪种模式
 （SHOW FRONTENDS 的字段/fe.conf 的 deploy_mode/进程列表有无 meta-service）；
 两种模式下同一症状（如导入慢）的排查路径分叉点）
```

- [ ] **Step 3: 引用校验**

运行引用校验脚本（FILE=docs/doris-internals/part1-architecture/04-two-architectures.md）。预期：无 MISSING。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part1-architecture/04-two-architectures.md
git commit -m "[docs] doris-internals part1: ch4 shared-nothing vs shared-storage

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 5 章《源码地图与开发环境》+ 部分目录页 + 总 README 收尾

**Files:**
- Create: `docs/doris-internals/part1-architecture/05-source-map-and-dev-env.md`
- Create: `docs/doris-internals/part1-architecture/README.md`
- Modify: `docs/doris-internals/README.md`（把第一部分状态改为"已完成"，复检链接）

**Interfaces:**
- Consumes: Task 2-5 产出的四个章节文件（目录页链接它们）。
- Produces: 编译/调试/UT 的标准操作流程，后续所有部分的"动手实验"段直接引用本章而不再重复环境说明。

- [ ] **Step 1: 核实素材**

```bash
head -100 AGENTS.md                     # 构建/测试规范全文
sed -n '1,60p' build.sh                 # 参数说明
cat run-be-ut.sh | head -40; cat run-fe-ut.sh | head -40
cat run-regression-test.sh | head -40
ls regression-test/conf/ | head
ls be/test | head; ls fe/fe-core/src/test | head
grep -rn "BUILD_TYPE" env.sh custom_env.sh.tpl 2>/dev/null | head -5
ls bin/                                  # start_fe.sh / start_be.sh
```

- [ ] **Step 2: 写作章节**

创建 `05-source-map-and-dev-env.md`（约 5000-7000 字），结构：

```markdown
# 第 5 章：源码地图与开发环境

## 5.1 仓库总地图
（顶层目录逐个一句话：be/ fe/ cloud/ gensrc/ regression-test/ thirdparty/
 docker/ tools/ 等；配一张 mermaid 图按"控制面/数据面/契约/测试"分组）

## 5.2 FE 源码地图
（fe/fe-core/src/main/java/org/apache/doris/ 下重要包逐个：nereids/ catalog/
 analysis/ planner/ qe/ transaction/ load/ journal/ persist/ system/ cloud/ 等，
 每个包一句话职责 + 指向后续深入讲解它的部分/章节；
 tricky 点：analysis/ 与 nereids/ 并存的历史原因（旧优化器遗留），
 读代码时如何避免误入已废弃路径）

## 5.3 BE 源码地图
（be/src/ 下重要目录：service/ runtime/ exec/ core/ storage/ io/ load/
 agent/ cloud/ common/ util/，每个一句话 + 前向链接；
 重点提示本 master 与旧资料的目录差异（storage↔olap、core↔vec、
 exec/pipeline 位置），帮读者校准网上旧文章）

## 5.4 编译与运行
（严格按 AGENTS.md：./build.sh --be --fe；BUILD_TYPE=ASAN 的意义与
 何时用 RELEASE；产物在 output/；单机部署步骤：改 conf 端口与
 priority_networks → start_fe.sh --daemon → start_be.sh --daemon →
 ADD BACKEND；启动慢的预期与日志确认法；
 易错点：ASAN 版本性能与内存开销、priority_networks 配错、端口冲突）

## 5.5 调试手段矩阵
（四种手段的适用场景与操作步骤：
 a. 日志：FE log4j 与 BE glog 的级别调整（含运行时动态调整入口）；
 b. 调试器：lldb/gdb attach BE、IDE 远程调试 FE 的要点；
 c. UT：run-be-ut.sh --run --filter / run-fe-ut.sh 的用法（从脚本
 head 中核实准确参数后写入）；
 d. 回归测试：run-regression-test.sh -d 目录 -s 用例 的规范用法；
 每种手段给一个真实可跑的最小例子）

## 5.6 动手实验
（核心点：完整走一遍 build → 拉起单机集群 → 跑通一个回归用例；
 易错点：故意在一个简单 BE UT 里改断一行断言，观察 ASAN 构建下
 UT 失败输出的读法（定位到行、看堆栈），再改回来 —— 
 这是后续所有章节"踩易错点"实验的基本功）

## 5.7 排查清单
（编译失败常见类：thirdparty 缺失/子模块未初始化/内存不足；
 启动失败常见类：日志在哪、先看什么）
```

- [ ] **Step 3: 写作部分目录页**

创建 `part1-architecture/README.md`（约 500 字）：

```markdown
# 第一部分：全局架构与设计哲学

（两三句导语：本部分建立全局心智模型，是后续所有部分的地基）

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-positioning.md) | Doris 是什么样的系统 | 定位与取舍 |
| [第 2 章](02-three-components.md) | 三大件解剖 | FE/BE/MS 职责边界 |
| [第 3 章](03-data-model.md) | 数据模型 | Table→Segment 层级 |
| [第 4 章](04-two-architectures.md) | 两种架构形态 | 双模式差异总表 |
| [第 5 章](05-source-map-and-dev-env.md) | 源码地图与开发环境 | 编译调试基本功 |

（下一部分预告：第二部分《一条查询 SQL 的一生》）
```

- [ ] **Step 4: 总 README 收尾**

修改 `docs/doris-internals/README.md`：第一部分状态改为"已完成"；确认 5 章链接与实际文件名一致。

- [ ] **Step 5: 全量引用与链接校验**

对本任务三个文件运行引用校验脚本；再对整个目录跑死链检查：

```bash
cd docs/doris-internals
find . -name "*.md" | while read -r f; do
  d=$(dirname "$f")
  grep -oE '\]\(([^)#]+)' "$f" | tr -d '](' | grep -v '^http' | while read -r p; do
    [ -e "$d/$p" ] || echo "DEAD LINK in $f: $p"
  done
done
```

预期：无输出。此时 Task 1 遗留的 part1 DEAD LINK 必须全部清零。

- [ ] **Step 6: 提交**

```bash
git add docs/doris-internals/part1-architecture/05-source-map-and-dev-env.md \
        docs/doris-internals/part1-architecture/README.md \
        docs/doris-internals/README.md
git commit -m "[docs] doris-internals part1: ch5 source map, part index, series index finalize

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：向作者汇报第一部分完成情况，请作者审阅风格与深度（这是设计文档约定的审阅关口），根据反馈决定是否调整骨架再进入第二部分的计划制定。
