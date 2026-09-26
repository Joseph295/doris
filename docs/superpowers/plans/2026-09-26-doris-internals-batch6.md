# 《Doris 内核透视》第六批交付物实施计划（第六部分：集群运维与故障排查）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第六部分（集群运维与故障排查）全部 6 章及部分目录页，并将系列 README 的第六部分状态翻转为"已完成"。

**Architecture:** 纯文档写作项目。本部分性质与前五部分不同：**是站在前五部分机制之上的综合运维篇**。前五部分每章末尾的"排查清单"给了零散的症状→路径对；本部分把它们组织成系统性方法论（ch1 工具箱 + ch2-5 按故障类别的决策树）并补上前部未展开的工具级深度（debug point、内省表、MemTracker 内核）。**写作纪律：机制一律链接前部对应章节，本部分只写"怎么查、怎么断、怎么治"；复述机制超过两句即算重复。** 五段式在本部分做统一裁剪（在各章内注明）：原理三连问保留但聚焦"排查体系设计"类问题；"源码走读"聚焦工具实现与日志/指标的产生点；"动手实验"变为**故障演练**（主动制造故障→用本章方法定位）。

**Tech Stack:** Markdown（GFM）、mermaid 决策树/流程图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束沿用前五批并补充第六部分锚点，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（聚焦排查体系设计类问题，严禁泛泛而谈）。
- **源码走读分清主次**：本部分聚焦工具实现、日志与指标的产生点；机制细节一律链接前部。
- **故障演练双目的**：主动制造该类故障 + 用本章方法完整定位一遍。环境引用 part1 第 5 章。
- 双模式并重：ch4 双模式各占半章（一体=副本/均衡故障、分离=缓存/计算组故障）；其余章节按症状分叉处交代。
- 所有源码引用 `路径:行号` 或 `路径`，先核实再落笔；**裸类名逐一 grep**；**反引号内禁写"类名.方法名"**（分拆式）；BE 配置核实 `DEFINE_mXxx`、FE 配置核实 `@ConfField(mutable)`。
- **跨部分引用惯例**：已成文部分首次提及用相对 md 链接+节号纯文本；part7 纯文本；**回引前必须 grep 目标文件确认内容真实存在**。本部分回引密度远高于前部——每章"素材盘点"步骤必须先 grep 收集前部相关排查清单原文再动笔。
- **目录页严格 ≤700 字**（批 5 新规）。
- 已核实的第六部分关键锚点（目录/文件级已核实，行号需现场核实）：
  - Debug point：BE `be/src/util/debug_points.cpp/h` + `be/src/service/http/action/debug_point_action.cpp`；FE 侧现场核实（grep DebugPointUtil）
  - 内存：`be/src/runtime/memory/`（`mem_tracker.h`、`mem_tracker_limiter.cpp/h`、`thread_mem_tracker_mgr.h`、`global_memory_arbitrator.cpp/h`、`cache_manager.cpp`、`jemalloc_control.cpp`、`heap_profiler.cpp`）
  - Metrics：`be/src/common/metrics/doris_metrics.cpp/h`；workload group 指标 `be/src/runtime/workload_group/workload_group_metrics.cpp`
  - 内省表：`fe/fe-core/src/main/java/org/apache/doris/tablefunction/`（`BackendsTableValuedFunction.java`、`MetadataGenerator.java`）、`fe/fe-core/src/main/java/org/apache/doris/datasource/systable/`（`SysTable.java` 族）；information_schema 现场核实（grep SchemaTable）
  - 审计日志：`fe/fe-core/src/main/java/org/apache/doris/qe/AuditLogHelper.java`、`AuditEventProcessor.java`
  - 前部已交付的排查资产（每章素材盘点必读）：part2 ch9 §9.4 profile 精读方法论与指纹表；part1-5 各章排查清单（grep "排查清单" 收集）；part2 ch9 慢查询四步漏斗；part3 ch4 publish 积压、ch1 事务状态；part4 ch3 选主/ch2 image、ch5 副本调度；part5 ch6 缓存排查
- 构建/测试命令与根 `AGENTS.md` 一致。章内基线注格式照旧。
- 每完成一个文件即提交，前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行。
- 统一**引用校验步骤**（含 `.g4`），预期输出为空：

```bash
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md|g4)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

---

### Task 1: 第 1 章《排查方法论与工具箱》

**Files:**
- Create: `docs/doris-internals/part6-operations/01-toolbox.md`

**Interfaces:**
- Consumes: part2 ch9 profile 方法论、part1 ch5 日志基础。
- Produces: 五件工具（日志/内省表/Profile/Metrics/debug point）的系统盘点，ch2-6 的排查步骤直接引用工具而不再解释。

- [ ] **Step 1: 核实素材**

```bash
grep -rn "class DebugPoints\|DBUG_EXECUTE_IF" be/src/util/debug_points.h | head -4
grep -rn "enable_debug_points" be/src/common/config.cpp fe/fe-common/src/main/java/org/apache/doris/common/Config.java | head -3
grep -rln "DebugPointUtil" fe/fe-core/src/main/java/org/apache/doris/common/util/ | head -1
grep -n "class.*Action" be/src/service/http/action/debug_point_action.h 2>/dev/null | head -2; ls be/src/service/http/action/ | grep -i debug
grep -rn "ADMIN SET.*CONFIG\|SET_CONFIG" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -2
ls fe/fe-core/src/main/java/org/apache/doris/tablefunction/ | head -15
grep -rn "active_queries\|backend_active_tasks" fe/fe-core/src/main/java/org/apache/doris/tablefunction/MetadataGenerator.java | head -4
grep -n "class AuditLogHelper" fe/fe-core/src/main/java/org/apache/doris/qe/AuditLogHelper.java
grep -rn "fe.audit.log" fe/fe-core/src/main/java/org/apache/doris/ -r --include=*.java -l | head -2
grep -n "DorisMetrics" be/src/common/metrics/doris_metrics.h | head -2
grep -rn "^sys_log_level\|sys_log_verbose_modules" fe/fe-common/src/main/java/org/apache/doris/common/Config.java be/src/common/config.cpp 2>/dev/null | head -4
grep -rn "排查清单" docs/doris-internals/part*/0*.md -l | wc -l   # 前部资产盘点
```

- [ ] **Step 2: 写作**

创建 `01-toolbox.md`（约 6000-8000 字），结构：

```markdown
# 第 1 章：排查方法论与工具箱

## 1.1 问题：故障时你有 60 秒决定往哪查
（三连问：排查体系的三种组织方式——工具堆砌（会用但不知道先用哪个）
 vs 按组件划分（FE/BE 谁的问题先猜）vs 按可观测层次分层
 （现象→指标→日志→内核态）；Doris 五件工具在层次里的位置；
 本部分与前五部分排查清单的关系：前部给"点"、本部分连"线"）

## 1.2 工具一/二：日志体系与审计日志
（FE log4j 分文件（fe.log/fe.warn.log/fe.audit.log——审计日志的字段
 与产生点 AuditLogHelper）、BE glog 分级；动态调级（回引 part1 ch5
 不重复，补：sys_log_verbose_modules 模块级 verbose 的用法与代价）；
 tricky 点：audit log 是排查慢查询的第一入口——queryId 贯穿
 FE/BE 日志与 profile 的关联键；
 易错点：verbose 全开打爆磁盘/性能）

## 1.3 工具三：内省表与命令
（information_schema 与 TVF 内省（backends()/active_queries 等，
 按 MetadataGenerator/tablefunction 真实清单归纳成表）；
 SHOW PROC 树的全景导航（综合前部用过的 /transactions、/cluster_health、
 /cluster_balance、/mem 等，给一张"哪类问题查哪棵子树"的地图）；
 易错点：内省表是 Master 视角还是本 FE 视角（回引 part4 ch1 权威语义））

## 1.4 工具四/五：Metrics 与 debug point
（Prometheus 指标体系（doris_metrics 产生点、FE/BE /metrics 端口）、
 该采哪些核心指标（结合前部各章提过的指标归纳）；
 debug point：测试注入机制（DBUG_EXECUTE_IF 宏、http 开关、
 enable_debug_points 配置）——排障时"复现难"问题的利器；
 tricky 点：debug point 是全局生效的——生产误开的后果；
 易错点：指标名在版本间漂移，写告警规则的兼容姿势）

## 1.5 方法论：从症状到子系统的决策树
（mermaid 决策树：慢/错/挂/涨 四大症状类 → 先看什么工具 →
 分流到 ch2-5 哪一章；每个分支注明对应前部机制章节链接；
 这是全部分的导航页）

## 1.6 故障演练
（核心点：不制造故障，先做"工具热身"——对一条正常查询走完
 audit log→queryId→profile→BE 日志关联全链路；
 易错点：开一个无害 debug point（选真实存在的点，核实）观察行为变化，
 体会"忘了关"的风险后关闭）

## 1.7 排查清单（元清单）
（工具选择常见错误：上来就翻 BE 日志/不留 queryId/重启大法毁现场）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/01-toolbox.md）。预期无 MISSING；裸类名与回引逐一核实。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/01-toolbox.md
git commit -m "[docs] doris-internals part6: ch1 troubleshooting toolbox

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 2 章《查询类故障：慢查询、OOM 与超时》

**Files:**
- Create: `docs/doris-internals/part6-operations/02-query-issues.md`

**Interfaces:**
- Consumes: ch1 工具箱；part2 全部（尤其 ch9 §9.4）；part5 ch3。
- Produces: 查询故障决策树与处置手册。

- [ ] **Step 1: 核实素材**

```bash
grep -rn "排查清单" docs/doris-internals/part2-query-lifecycle/*.md | head -12   # 收集查询侧资产
grep -rn "query_timeout\|execution_timeout" fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java | head -4
grep -rn "MEM_LIMIT_EXCEEDED\|memory exceed" be/src/runtime/memory/mem_tracker_limiter.cpp | head -4
grep -rn "cancel.*timeout\|TIMEOUT" fe/fe-core/src/main/java/org/apache/doris/qe/ConnectContext.java | head -4
grep -rn "active_queries" fe/fe-core/src/main/java/org/apache/doris/tablefunction/MetadataGenerator.java | head -2
grep -rn "SHOW PROCESSLIST\|processlist" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -2
grep -rn "kill query\|KILL" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -3
```

- [ ] **Step 2: 写作**

创建 `02-query-issues.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：查询类故障 —— 慢、爆、超时

## 2.1 问题：查询故障的分诊逻辑
（三连问：按报错文本查（同一报错多种根因）vs 按组件猜 vs
 按"查询生命周期阶段"分诊（part2 的九章路径就是分诊图——
 卡在哪一段，工具证据是什么）；本章把 part2 各章排查清单
 组织成一棵完整决策树）

## 2.2 慢查询：四步漏斗的完整版
（part2 ch9 §9.4 四步漏斗回引为骨架，本章补"漏斗前"与"漏斗后"：
 前=怎么发现慢（audit log 阈值/active_queries/SHOW PROCESSLIST）、
 拿到 queryId 与 profile；后=六类指纹（回引 ch9 指纹表）各自的
 处置动作清单（改 SQL/建索引/调并行度/触发 compaction……每类
 链接机制章节）；
 tricky 点：偶发慢 vs 持续慢的分野——版本数/缓存/调度三个
 时变因素（链接 part1 3.3/part5 ch6/part2 ch5））

## 2.3 内存超限：读懂报错与三层限额
（MEM_LIMIT_EXCEEDED 报错文本逐段解读（真实报错格式，核实
 mem_tracker_limiter 的报错拼装）；查询级/负载组/进程级三层
 限额的关系（详细机制留 ch6，本章给"看到哪层报错查哪里"）；
 易错点：把进程级内存紧张误当查询问题（ch6 伏笔））

## 2.4 超时与取消：谁在数秒表
（query_timeout/execution_timeout 的真实语义与优先级（核实）；
 超时后的取消链路回引 part2 ch9 9.2；KILL 语法；
 tricky 点："客户端超时但查询还在跑"三种成因（客户端/FE/BE
 各自的表，回引 part2 ch9 cancel 传播））

## 2.5 双模式对比
（查询故障分诊两模式基本一致；分叉点：scan 慢在分离模式多一个
 cache 维度（链接 part5 ch6 排查清单）、副本坏在一体模式
 （链接 part4 ch5）——决策树上标注分叉）

## 2.6 故障演练
（核心点：构造一条慢查询（大表无索引点查），从 audit log 发现→
 profile 指纹→定位→建前缀索引解决，全链路走一遍；
 易错点：把 exec_mem_limit 调小制造 MEM_LIMIT_EXCEEDED，
 读懂报错的每一段后恢复）

## 2.7 排查清单（决策树浓缩版）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/02-query-issues.md）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/02-query-issues.md
git commit -m "[docs] doris-internals part6: ch2 query issues

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 3 章《导入类故障：失败、积压与事务卡住》

**Files:**
- Create: `docs/doris-internals/part6-operations/03-load-issues.md`

**Interfaces:**
- Consumes: ch1 工具箱；part3 全部排查清单。
- Produces: 导入故障决策树与处置手册。

- [ ] **Step 1: 核实素材**

```bash
grep -rn "排查清单" docs/doris-internals/part3-load-lifecycle/*.md | head -12
grep -rn "error url\|ErrorURL" be/src/load/stream_load/stream_load_executor.cpp | head -2
grep -rn "SHOW LOAD\|SHOW ROUTINE LOAD\|SHOW STREAM LOAD" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -4
grep -rn "SHOW TRANSACTION" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -2
grep -rn "PAUSED" fe/fe-core/src/main/java/org/apache/doris/load/routineload/RoutineLoadJob.java | head -3
```

- [ ] **Step 2: 写作**

创建 `03-load-issues.md`（约 5000-7000 字），结构：

```markdown
# 第 3 章：导入类故障 —— 失败、积压与事务卡住

## 3.1 问题：导入故障先分"哪一段"
（三连问回顾式：part3 的链路（接入→写入→提交→可见）天然是分诊图；
 失败=快反馈（读报错）、慢/积压=看堆积点、卡住=看事务状态机；
 本章按这三类组织 part3 各章清单）

## 3.2 快失败类：报错的读法
（error url（part3 ch2）、-235（part3 各章的完整故事线汇总成
 一页处置卡）、Label 冲突（part3 ch1 三时机表）、格式/权限/超限
 各类报错的一句话分流；每类链接机制章节）

## 3.3 积压与变慢类
（memtable 反压（part3 ch3）、publish 积压深度（part3 ch4 命令）、
 compaction 跟不上（part3 ch6 score 观察）、Routine Load 消费
 积压与 PAUSED 原因（part3 ch5）；
 tricky 点：积压的因果链常反直觉——查询慢可能是导入的锅
 （版本堆积），导入慢可能是查询的锅（IO 争抢），给判别方法）

## 3.4 卡住类：事务状态机定位
（SHOW TRANSACTION/PROC 定位卡在哪个状态（part3 ch1/ch4），
 各状态的"卡因清单"与处置（含分离模式 commit 冲突）；
 易错点：手动 abort 事务的边界与风险）

## 3.5 双模式对比
（分叉点标注：publish 类问题一体专属、MS commit 冲突分离专属，
 决策树分叉处链接）

## 3.6 故障演练
（核心点：复用 part3 ch6 实验的 -235 复现，但这次从"用户报导入失败"
 出发用 ch1 工具反向定位到 compaction 落后；
 易错点：停 BE 制造 publish 积压，观察 SHOW PROC 深度指标并恢复）

## 3.7 排查清单（决策树浓缩版）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/03-load-issues.md）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/03-load-issues.md
git commit -m "[docs] doris-internals part6: ch3 load issues

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 4 章《副本与均衡故障（一体）／缓存与计算组故障（分离）》

**Files:**
- Create: `docs/doris-internals/part6-operations/04-replica-and-cache-issues.md`

**Interfaces:**
- Consumes: ch1 工具箱；part4 ch5、part5 ch6、part2 ch5。
- Produces: 存储层故障双模式手册。

- [ ] **Step 1: 核实素材**

```bash
grep -rn "排查清单" docs/doris-internals/part4-fe-internals/05-scheduling.md docs/doris-internals/part5-storage-engine/06-cloud-storage.md | head -8
grep -rn "ADMIN REPAIR\|admin_repair" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -2
grep -rn "ADMIN SET REPLICA\|SET_REPLICA" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -3
grep -rn "tablet_health" fe/fe-core/src/main/java/org/apache/doris/common/proc/ -l | head -2
grep -rn "file_cache" be/src/service/http_service.cpp | head -3
```

- [ ] **Step 2: 写作**

创建 `04-replica-and-cache-issues.md`（约 5000-7000 字），结构：

```markdown
# 第 4 章：副本与均衡故障（一体）／缓存与计算组故障（分离）

## 4.1 问题：同一层故障、两套物理现实
（三连问：一体的"数据安全"故障（副本丢/版本落后/colocate 乱）与
 分离的"性能退化"故障（cache 失效/计算组不均）本质差异——
 前者威胁正确性、后者威胁 SLA；排查心态与工具的相应差别）

## 4.2 一体模式：副本故障手册
（part4 ch5 排查清单展开成处置手册：tablet 不健康状态速查
 （12 状态表回引）、修复不动的六因（配额/黑名单/盘满/版本落后/
 colocate 约束/调度积压）、ADMIN REPAIR/SET REPLICA 命令的
 使用边界与危险操作警示（核实语法）；
 tricky 点：误用 SET REPLICA STATUS 把好副本标坏的事故模式）

## 4.3 一体模式：均衡故障
（均衡不动/均衡风暴两个方向（part4 ch5 5.3 回引）；
 易错点：扩容后期望"立刻均衡"的误解——观察节奏与限流参数）

## 4.4 分离模式：缓存与计算组故障
（命中率突降三查（part5 ch6 回引展开）、预热失败/慢（part4 ch5
 5.4 warmup 链路）、计算组倾斜（映射重算时机）、对象存储
 429/费用异常（part5 ch6）；
 tricky 点：加减 BE 后的 cache 冷启动窗口是预期而非故障——
 与真故障的区分）

## 4.5 故障演练
（核心点（一体）：三副本表 kill BE 复现修复全流程，用 ch1 工具
 观察每一步（回引 part4 ch5 实验但从排障视角重走）；
 易错点（分离，无环境则纸上）：清 cache 观察命中率恢复曲线，
 区分"冷启动"与"容量不足"的曲线形态）

## 4.6 排查清单（双模式决策树）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/04-replica-and-cache-issues.md）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/04-replica-and-cache-issues.md
git commit -m "[docs] doris-internals part6: ch4 replica and cache issues

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 5 章《FE 故障：选主、元数据与恢复》

**Files:**
- Create: `docs/doris-internals/part6-operations/05-fe-issues.md`

**Interfaces:**
- Consumes: ch1 工具箱；part4 ch1-3。
- Produces: FE 故障手册（含高危恢复操作规程）。

- [ ] **Step 1: 核实素材**

```bash
grep -rn "排查清单" docs/doris-internals/part4-fe-internals/0[123]*.md | head -10
grep -rn "metadata_failure_recovery" bin/start_fe.sh | head -2
grep -rn "class BDBTool\|BDBDebugger" fe/fe-core/src/main/java/org/apache/doris/journal/bdbje/*.java | head -3
grep -rn "helper" bin/start_fe.sh | head -2
ls fe/fe-core/src/main/java/org/apache/doris/persist/meta/
grep -rn "SHOW FRONTENDS" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -1
```

- [ ] **Step 2: 写作**

创建 `05-fe-issues.md`（约 5000-7000 字），结构：

```markdown
# 第 5 章：FE 故障 —— 选主、元数据与恢复

## 5.1 问题：FE 故障的特殊性
（三连问：FE 故障为什么"小症状大风险"（元数据是全局单点资产）；
 排查 FE 的第一原则：先保元数据再恢复服务 vs 先恢复服务——
 什么情况用哪个；本章把 part4 ch2/ch3 排查清单展开成规程）

## 5.2 选主与角色异常
（选不出主（多数派核对表，part4 ch3 回引）、新主不服务
 （回放窗口判断：日志关键字与进度估算）、角色显示异常
 （SHOW FRONTENDS 各列的诊断读法，核实列含义）；
 tricky 点：加错节点类型的事后补救路径）

## 5.3 元数据损坏与恢复规程（高危操作章）
（image 损坏/回放失败的启动报错分类；恢复决策树：
 有健康 Follower→正常重加入；全挂→metadata_failure_recovery
 单点恢复规程（-r 参数，part4 ch2 已证是启动项）逐步走+
 每步风险注记；BDBTool/BDBDebugger 的查看用法（只读操作优先）；
 易错点：对多个 FE 同时 -r 的灾难、恢复后忘记去掉参数）

## 5.4 双模式对比
（分离模式 FE 故障影响面缩小（part3 ch1 1.4/part4 ch3 3.4 回引），
 但 MS 依赖使"FE 起不来"多一类根因（连不上 MS）——判别方法）

## 5.5 故障演练
（核心点：3FE 集群 kill Master 观察切换（part4 ch3 实验的排障
 视角重走：这次只用 ch1 工具判断进度）；
 易错点（谨慎，实验环境）：单 FE 环境演练 -r 恢复全流程一次，
 体会每步的检查点；结束必须恢复正常启动方式）

## 5.6 排查清单（含高危操作 checklist）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/05-fe-issues.md）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/05-fe-issues.md
git commit -m "[docs] doris-internals part6: ch5 fe issues and recovery

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 6 章《内存管理：BE 内存模型与 MemTracker》

**Files:**
- Create: `docs/doris-internals/part6-operations/06-memory.md`

**Interfaces:**
- Consumes: ch2 的三层限额伏笔；part2 ch8 spill、part3 ch3 limiter。
- Produces: BE 内存内核与定位方法（本部分唯一"机制深潜"章，因前部未覆盖）。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/runtime/memory/
grep -n "class MemTracker\b" be/src/runtime/memory/mem_tracker.h | head -2
grep -n "class MemTrackerLimiter" be/src/runtime/memory/mem_tracker_limiter.h | head -2
grep -n "enum class Type" be/src/runtime/memory/mem_tracker_limiter.h | head -2; sed -n '/enum class Type/,/};/p' be/src/runtime/memory/mem_tracker_limiter.h | head -15
grep -n "class ThreadMemTrackerMgr" be/src/runtime/memory/thread_mem_tracker_mgr.h | head -2
grep -n "class GlobalMemoryArbitrator" be/src/runtime/memory/global_memory_arbitrator.h | head -2
grep -rn "mem_limit\|soft_mem_limit" be/src/common/config.cpp | head -5
grep -rn "process memory\|sys_mem_available" be/src/runtime/memory/global_memory_arbitrator.h | head -4
grep -rn "jemalloc" be/src/runtime/memory/jemalloc_control.cpp | head -3
grep -rn "/mem_tracker\|mem_tracker" be/src/service/http_service.cpp | head -3
grep -rn "class CacheManager" be/src/runtime/memory/cache_manager.h | head -2
```

- [ ] **Step 2: 写作**

创建 `06-memory.md`（约 6000-8000 字），结构：

```markdown
# 第 6 章：内存管理 —— BE 内存模型与 MemTracker

## 6.1 问题：C++ 进程里给几百个查询记账
（三连问：不记账（OOM killer 裁决）vs 全局配额（谁超杀谁不知道）
 vs 层级记账+线程本地挂账（MemTracker 树+ThreadMemTrackerMgr）；
 记账的两难：精确（每次分配都记，慢）vs 近似（攒批记，有误差）——
 Doris 的取舍（核实攒批/flush 机制）；jemalloc 与记账的关系
 （记的是逻辑分配、RSS 是物理占用——两者差异就是碎片/缓存））

## 6.2 源码走读：MemTracker 树与三层限额
（MemTrackerLimiter 的 Type 枚举（真实类型清单：GLOBAL/QUERY/
 LOAD/COMPACTION…核实）；查询级 exec_mem_limit→负载组→进程级
 mem_limit 的三层判定顺序（ch2 伏笔回收，核实真实判定链）；
 ThreadMemTrackerMgr 线程挂账切换（attach 机制）；
 tricky 点：挂账挂错树——异步线程/线程池场景的 tracker 传递，
 记账漂移的成因；
 易错点：看 metrics 的 tracker 值 vs RSS 差距大就喊泄漏——
 jemalloc 缓存/碎片的正常范围）

## 6.3 源码走读：全局仲裁与自保
（GlobalMemoryArbitrator：soft/hard 水位、进程级自保动作序列
 （挑谁牺牲：cache 收缩→spill→取消查询的顺序，核实真实策略）；
 CacheManager 的统一收缩入口；
 与 part2 ch8 spill 触发、part3 ch3 limiter 的关系图
 （三个水位体系一张图讲清，回引）；
 tricky 点：内存紧张时的"雪崩链"——cancel 风暴的形成与参数）

## 6.4 定位方法：从报错/指标到根因
（MEM_LIMIT_EXCEEDED 报错的完整解剖（ch2 给了读法，这里给
 产生点与字段含义源码级确认）；/mem_tracker http 端点与
 内存 profile（heap_profiler，核实启用方式）的使用；
 内存问题四分类：单查询大/并发高/cache 占比高/真泄漏——
 各自的证据形态）

## 6.5 双模式对比
（内存模型两模式一致；分离模式 file cache 占比是新大户——
 其内存/磁盘账本位置（回引 part5 ch6））

## 6.6 故障演练
（核心点：跑大查询用 /mem_tracker 端点观察树的层级值变化；
 易错点：调小 mem_limit 触发进程级自保，观察牺牲顺序
 （日志证据），恢复；对比 tracker 总值与 RSS 理解差距）

## 6.7 排查清单
（OOM 三查/泄漏疑云的证伪流程/cache 占比调优入口）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part6-operations/06-memory.md）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/06-memory.md
git commit -m "[docs] doris-internals part6: ch6 memory management memtracker

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 7: 部分目录页 + 系列 README 翻转

**Files:**
- Create: `docs/doris-internals/part6-operations/README.md`
- Modify: `docs/doris-internals/README.md`（第六部分翻转+链接，免责声明下移到 part6 之后仅辖 part7）
- Modify: `docs/doris-internals/part5-storage-engine/README.md`（预告改直链 ../part6-operations/README.md）

**Interfaces:**
- Consumes: Task 1-6 产出。
- Produces: 完整可导航的第六部分。

- [ ] **Step 1: 核实现状**

```bash
ls docs/doris-internals/part6-operations/
grep -n "第六部分" docs/doris-internals/README.md
grep -n "第六部分\|part6" docs/doris-internals/part5-storage-engine/README.md
```

- [ ] **Step 2: 写作与修改**

part6 README（**≤700 字**：导语 2-3 句（综合运维篇定位、ch1 工具箱+决策树导航、ch6 唯一机制深潜章）+ 6 行表格（关键收获=一行钩子）+ part7 预告纯文本）；系列 README 最小化修改；part5 README 预告改直链。

- [ ] **Step 3: 全量校验**

三文件引用校验 + 全树死链检查（预期无输出）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part6-operations/README.md \
        docs/doris-internals/README.md \
        docs/doris-internals/part5-storage-engine/README.md
git commit -m "[docs] doris-internals part6: part index, series index flip to complete

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：最终整分支审查（fable 模型，含台账 triage、回引真实性专项（本部分回引密度最高，重点查）、主线/决策树交叉一致性、"机制复述超两句"重复度检查、fix-later 清单处置），修复确认后 push，向作者简报第六部分完成情况，继续第七部分（案例集，需先从 git log 落实真实 PR 案例再定计划）的计划制定。
