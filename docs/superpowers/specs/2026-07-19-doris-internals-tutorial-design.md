# 《Doris 内核透视》教程设计文档

日期：2026-07-19
状态：已与作者确认的设计决策，作为整个系列写作的基准。

## 1. 目标与读者

**读者画像**：10 年大数据领域经验的工程师，部署过 Doris、排查过简单问题，希望成为能解决绝大部分 Doris 问题的内核级专家。

**教程目标**：读完后读者应能——

1. 独立画出任意重要请求（查询、导入、DDL、副本修复等）的端到端执行路径；
2. 读懂并能修改 FE/BE/MetaService 的核心源码，理解每处设计的"所以然"；
3. 面对线上疑难问题（慢查询、导入失败、副本异常、选主故障、Compaction 积压等）有系统的定位方法论；
4. 理解存算一体与存算分离两种架构在每个关键环节的实现差异。

**语言**：中文。代码注释、类名、日志保留英文原文。

## 2. 已确认的四项关键决策

| 决策项 | 结论 |
|---|---|
| 主线组织 | 请求生命周期主线（"一条 SQL 的一生"式叙事），模块知识沿路径展开 |
| 存算分离覆盖 | 双模式并重：每个主线章节同时讲存算一体与存算分离两条实现路径 |
| 实操形态 | 源码编译调试为主：本地 build、断点/日志追踪、UT、场景构造复现 |
| 交付节奏 | 按部分顺序交付：先 README + 第一部分，作者审阅风格后继续后续部分 |

## 3. 内容锚定原则

- 所有代码解析基于**本仓库当前 master** 的真实代码，引用格式 `文件路径:行号`（可点击跳转）。
- 目录引用必须与本仓库实际结构一致。本 master 的 BE 已重构，注意：
  - 执行引擎（Pipeline）：`be/src/exec/`（`pipeline/`、`operator/`、`scan/`、`sink/`、`exchange/`、`runtime_filter/`、`spill/`）
  - 存储引擎：`be/src/storage/`（旧版资料中的 `be/src/olap/` 已更名）
  - 向量化内存结构（Block/Column/DataType）：`be/src/core/`（旧版资料中的 `be/src/vec/` 已重构）
  - BE 侧存算分离逻辑：`be/src/cloud/`
  - FE 核心：`fe/fe-core/src/main/java/org/apache/doris/`（`nereids/` 优化器、`catalog/`、`transaction/` 等）
  - 存算分离元数据服务：`cloud/src/`（`meta-service/`、`meta-store/`、`recycler/`、`resource-manager/`、`snapshot/`）
- 案例集（第七部分）从本仓库 git 历史中挑选真实的重要 PR / bugfix 深挖，注明 PR 编号。
- 编译、运行、测试的实操步骤必须遵循仓库根 AGENTS.md 的规范（`build.sh`、`run-be-ut.sh`、`run-fe-ut.sh`、`run-regression-test.sh`）。
- 写作时对每个引用的类名/函数名/路径先在代码中核实再落笔，禁止凭记忆书写。

## 4. 目录结构

```
docs/doris-internals/
├── README.md                    # 总目录、阅读地图、源码环境搭建（编译/调试/UT 环境）
├── part1-architecture/          # 第一部分：全局架构与设计哲学
├── part2-query-lifecycle/       # 第二部分：一条查询 SQL 的一生
├── part3-load-lifecycle/        # 第三部分：一次导入的一生
├── part4-fe-internals/          # 第四部分：元数据与 FE 内核
├── part5-storage-engine/        # 第五部分：存储引擎深潜
├── part6-operations/            # 第六部分：集群运维与故障排查
└── part7-case-studies/          # 第七部分：经典 feature/bug 案例集
```

每部分目录内为若干编号章节文件（`01-xxx.md`、`02-xxx.md`……）加一个部分内 `README.md` 目录页。

## 5. 每章固定骨架

每个章节按以下五段式组织（个别章节可裁剪，但默认齐全）：

1. **原理**——该机制解决什么问题、为什么这样设计、有哪些替代方案被放弃（知其所以然）。
2. **源码走读**——沿执行路径逐段解析真实代码，关键函数给出调用链图；代码块配逐段解释。
3. **双模式对比**——存算一体与存算分离在该环节的路径差异；涉及 MetaService 的深入 `cloud/src/`。
4. **动手实验**——源码级实操：编译目标、断点/日志位置、UT 运行命令、如何构造场景观察行为。
5. **排查清单**——本章知识对应的典型故障：症状 → 定位路径 → 关键日志/工具。

## 6. 各部分章节规划（写作时可微调，增删章节需回到本设计文档更新）

### 第一部分：全局架构与设计哲学
1. Doris 是什么样的系统：MPP、列存、实时分析的定位与取舍
2. 三大件解剖：FE / BE / MetaService 的职责边界与进程结构
3. 数据模型：Table→Partition→Tablet→Rowset→Segment 层级与三种数据模型（Duplicate/Unique/Aggregate）
4. 两种架构形态：存算一体 vs 存算分离的本质差异
5. 源码地图与开发环境：仓库结构导览、编译（ASAN）、调试器接入、三类测试框架

### 第二部分：一条查询 SQL 的一生
1. 连接与协议：MySQL 协议接入、ConnectContext、查询入口
2. 解析与合法化：词法语法解析到 LogicalPlan
3. Nereids 优化器（上）：RBO 规则重写体系
4. Nereids 优化器（下）：CBO、统计信息与代价模型
5. 计划分发：Fragment 切分、Coordinator 调度与两种模式下的副本选择
6. BE Pipeline 执行引擎：算子、依赖驱动调度、并行度
7. Scan 路径：存算一体本地读 vs 存算分离 File Cache + 远端存储读
8. Join / 聚合 / 排序算子与 Runtime Filter、Spill
9. 结果回传与 Profile 精读：从 Profile 反推执行瓶颈

### 第三部分：一次导入的一生
1. 导入方式总览与事务模型：2PC、Label 机制、事务状态机
2. Stream Load 全路径：HTTP 接入→计划→Sink→MemTable→Flush
3. Tablet 写入细节：MemTable、Segment 生成、主键模型 Delete Bitmap
4. 事务提交与可见性：存算一体 Publish Version vs 存算分离 MetaService 提交
5. 其他导入方式：Broker/Routine/Insert Into 的路径差异
6. Compaction：为什么需要、调度策略、两种模式下的执行位置

### 第四部分：元数据与 FE 内核
1. Catalog 体系与元数据内存结构
2. 元数据持久化：EditLog、bdbje、Checkpoint（存算一体）
3. FE 高可用：选主、角色（Master/Follower/Observer）、故障切换
4. 存算分离元数据：MetaService 架构、FoundationDB 数据布局、Recycler
5. 调度体系：Tablet 均衡、副本修复（存算一体）与集群/计算组管理（存算分离）
6. 外部数据源：Catalog 联邦查询架构概览

### 第五部分：存储引擎深潜
1. Rowset 与 Segment 文件格式：编码、压缩、页结构
2. 索引体系：前缀索引、ZoneMap、BloomFilter、倒排索引
3. 读路径内核：谓词下推、延迟物化、聚合/去重读逻辑
4. 主键模型内核：MoW 的 Delete Bitmap 全生命周期
5. 数据管理：Delete、Schema Change、分区生命周期
6. 存算分离存储：File Cache 内核、对象存储交互、数据回收

### 第六部分：集群运维与故障排查
1. 排查方法论与工具箱：日志体系、内省表、Profile、Metrics、debug point
2. 查询类故障：慢查询、OOM、超时的系统定位法
3. 导入类故障：失败、积压、事务卡住
4. 副本与均衡类故障（存算一体）／缓存与计算组故障（存算分离）
5. FE 故障：选主异常、元数据损坏恢复、image 回滚
6. 内存管理：BE 内存模型、MemTracker、常见内存问题定位

### 第七部分：经典 feature/bug 案例集
从 git 历史挑选 8~12 个真实案例，每案例讲：问题背景→根因分析→修复思路→源码对照→经验教训。候选方向（写作时从 git log 落实具体 PR）：优化器错误结果类、主键模型正确性类、内存/性能回退类、存算分离一致性类、并发竞争类。

## 7. 写作风格

- 面向资深工程师：不解释大数据常识，聚焦 Doris 特有设计。
- "所以然"优先：每个机制先讲动机与权衡，再看实现。
- 图辅助叙事：执行路径用 mermaid 时序图/流程图表达。
- 代码块保持精简：截取关键片段（≤40 行/段）+ 逐段讲解，完整逻辑用 `文件:行号` 指路。
- 每章可独立阅读，跨章引用用相对链接。

## 8. 交付与验收

- 交付顺序：README + 第一部分 → 作者审阅 → 依次 2~7 部分，每部分完成后简报进度。
- 单章验收标准：五段式齐全（或注明裁剪原因）；所有代码引用在当前 master 可定位；实操命令与 AGENTS.md 规范一致；双模式差异有明确交代。
- 本教程为仓库内文档，不涉及编译产物，不影响构建；提交在专用分支 `doris-internals-tutorial` 上进行。

## 9. 不在范围内（YAGNI）

- 不写英文版；不做网站化/发布流水线。
- 不覆盖 fs_brokers、fe_plugins、sdk、extension 等外围模块（案例涉及时局部带过）。
- 不承诺覆盖所有 SQL 功能点，聚焦执行路径与内核机制。
