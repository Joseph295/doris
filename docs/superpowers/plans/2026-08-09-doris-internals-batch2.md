# 《Doris 内核透视》第二批交付物实施计划（第二部分：一条查询 SQL 的一生）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第二部分（一条查询 SQL 的一生）全部 9 章及部分目录页，并将系列 README 的第二部分状态翻转为"已完成"。

**Architecture:** 纯文档写作项目。每章按设计文档（`docs/superpowers/specs/2026-07-19-doris-internals-tutorial-design.md`）第 5 节的五段式骨架写作；主线是一条 `SELECT` 从 MySQL 客户端发出到结果返回的完整路径，9 章按路径顺序衔接（连接→解析→RBO→CBO→分发→Pipeline→Scan→算子→回传）。每个任务 = 一个章节文件，流程固定为"核实素材 → 写作 → 引用校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid 图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束来自设计文档与第一批的经验教训，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（严禁泛泛而谈）：遇到了什么问题？→ 有哪些候选方案、各有什么优劣？→ Doris 最终怎么考量和解决的？
- **源码走读分清主次**：显而易见的高度概括；不易理解的详细逐段解释；重点挖掘易错点/tricky 点并解释"为什么这么写、错写会怎样"。
- **动手实验双目的**：验证核心点 + 主动踩一遍易错点。第 1 部分第 5 章（`docs/doris-internals/part1-architecture/05-source-map-and-dev-env.md`）已给出编译/单机部署/UT/回归/日志调整的标准流程，实验中直接引用，不再重复环境说明。
- 双模式并重：每章交代存算一体与存算分离的路径差异。本部分第 1-4、6、8、9 章的 FE 优化路径与 BE 算子逻辑两模式基本一致，允许压缩为一小节甚至一句注明"本环节两模式一致，差异见第 X 章"；第 5、7 章是本部分双模式差异的主战场，必须完整展开。
- 所有源码引用格式为 `路径:行号` 或 `路径`，必须在当前 master 真实存在；**先核实再落笔，禁止凭记忆写引用**。
- **裸类名陷阱（第一批教训）**：引用校验脚本只校验带后缀（`.java`/`.cpp` 等）的路径，正文中的裸类名（如 `StmtExecutor`）会绕过脚本。凡在正文提到的类名/方法名，写作时必须逐一 `grep -rn "class Xxx"` 核实存在；自查与审阅都要人工核对裸类名。
- 已核实的关键锚点（目录/文件级已核实，行号需现场核实）：
  - 连接与协议：`fe/fe-core/src/main/java/org/apache/doris/mysql/`（`AcceptListener.java`、`MysqlChannel.java`、握手/认证包）；`fe/fe-core/src/main/java/org/apache/doris/qe/`（`ConnectContext.java`、`ConnectScheduler.java`、`ConnectPoolMgr.java`、`ConnectProcessor.java`、`MysqlConnectProcessor.java`、`StmtExecutor.java`）
  - 解析：语法文件在 `fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4` 与 `DorisLexer.g4`（注意不在 fe-core）；`fe/fe-core/src/main/java/org/apache/doris/nereids/parser/NereidsParser.java`、`LogicalPlanBuilder.java`；查询分析入口 `fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/executor/Analyzer.java`（旧 `analysis/` 包的 Stmt AST 已整体删除，见第 1 部分第 5 章）
  - 优化器：`fe/fe-core/src/main/java/org/apache/doris/nereids/`（`NereidsPlanner.java`、`CascadesContext.java`、`memo/`、`jobs/rewrite/`、`jobs/cascades/`、`rules/rewrite/`、`rules/exploration/`、`rules/implementation/`、`cost/`（`CostModel.java`、`CostCalculator.java`）、`stats/`）；统计信息 `fe/fe-core/src/main/java/org/apache/doris/statistics/`（`AnalysisManager.java` 等）
  - 分发：`fe/fe-core/src/main/java/org/apache/doris/planner/`（`PlanFragment.java`、`ExchangeNode.java`、`OlapScanNode.java`）；`fe/fe-core/src/main/java/org/apache/doris/qe/`（`Coordinator.java`、`NereidsCoordinator.java`、`CoordinatorContext.java`、`SimpleScheduler.java`）；BE 接收入口 `be/src/service/internal_service.cpp`、`be/src/runtime/fragment_mgr.cpp`
  - BE 执行：**Pipeline 框架在 `be/src/exec/pipeline/`（仅调度设施：`pipeline_fragment_context`、`pipeline_task`、`task_scheduler`、`dependency`），算子实现在 `be/src/exec/operator/`（平级目录，不在 pipeline/ 下）**；`be/src/exec/` 下还有平级的 `scan/`、`exchange/`、`sink/`、`runtime_filter/`、`spill/`、`sort/`
  - Scan 与缓存：`be/src/exec/scan/`（`scanner_context.cpp`、`olap_scanner.cpp`）；File Cache 在 `be/src/io/cache/`（`BlockFileCache`、`CachedRemoteFileReader`）
  - 结果与 Profile：`be/src/runtime/result_buffer_mgr.cpp`、`be/src/runtime/result_block_buffer.cpp`；**`RuntimeProfile` 在 `be/src/runtime/runtime_profile.h`（不在 util/）**；FE 侧 `fe/fe-core/src/main/java/org/apache/doris/common/profile/`（`Profile.java`、`ExecutionProfile.java`）
- 构建/测试命令必须与根 `AGENTS.md` 一致：`./build.sh --be --fe`（默认 ASAN）、`run-be-ut.sh`、`run-fe-ut.sh`、`run-regression-test.sh`（regression 用 `-d 父目录 -s 用例名`）。
- 章内交叉引用：第一部分各章用相对链接 `../part1-architecture/0X-xxx.md`；本部分章间用同目录相对链接。每章开头保留与第一部分相同的基线说明格式：「基于写作时核实所用的 HEAD（`XXXX`，源码树与系列基线 `7bc98f696f` 一致）」，SHA 用写作时 `git rev-parse --short HEAD` 的实际值。
- 每完成一个文件即提交，提交信息前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行（见任务内命令）。
- 每个写作任务完成后执行统一的**引用校验步骤**：

```bash
# 在仓库根执行；FILE 为本任务产出的 md 文件
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md|g4)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

预期输出为空（无 MISSING 行）。出现 MISSING 必须修正后重跑，直至为空再提交。注意本批新增 `.g4` 后缀。

---

### Task 1: 第 1 章《连接与协议》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/01-connection-and-protocol.md`

**Interfaces:**
- Consumes: 第 1 部分第 2 章的 FE 线程/端口模型（`query_port` 9030）。
- Produces: `ConnectContext`/`StmtExecutor` 两个贯穿本部分的核心对象锚点；后续各章从 `StmtExecutor` 接力。

- [ ] **Step 1: 核实素材**

```bash
ls fe/fe-core/src/main/java/org/apache/doris/mysql/ | head -30
grep -n "class AcceptListener" fe/fe-core/src/main/java/org/apache/doris/mysql/AcceptListener.java
grep -n "handshake\|auth" fe/fe-core/src/main/java/org/apache/doris/qe/ConnectProcessor.java | head
grep -n "class ConnectScheduler\|submit\|registerConnection" fe/fe-core/src/main/java/org/apache/doris/qe/ConnectScheduler.java | head
grep -n "qe_max_connection" fe/fe-core/src/main/java/org/apache/doris/common/Config.java
grep -n "COM_QUERY\|dispatch\|handleQuery" fe/fe-core/src/main/java/org/apache/doris/qe/MysqlConnectProcessor.java | head
grep -n "forward\|MasterOp" fe/fe-core/src/main/java/org/apache/doris/qe/StmtExecutor.java | head -10
ls fe/fe-core/src/main/java/org/apache/doris/mysql/authenticate/ | head
```

- [ ] **Step 2: 写作**

创建 `01-connection-and-protocol.md`（约 5000-7000 字），结构：

```markdown
# 第 1 章：连接与协议 —— 查询的入口

## 1.1 问题：分析型数据库拿什么协议见客户端
（三连问：自定义协议生态从零建 vs HTTP/JDBC 专属 vs 兼容 MySQL 线协议；
 兼容 MySQL 的收益（驱动/BI 工具全免费）与代价（握手/认证/编码细节全要照抄，
 协议演进被上游锁死）；Doris 选 MySQL 协议 + 另开 HTTP（Stream Load 等）
 的分工考量）

## 1.2 源码走读：从 accept 到 ConnectContext
（NIO accept（AcceptListener）→ 握手包/认证 → ConnectContext 的创建与归属
 （ConnectScheduler/ConnectPoolMgr 管理连接池与配额）；
 mermaid 时序图：client ↔ FE 的握手/认证/COM_QUERY 序列；
 tricky 点：ConnectContext 是线程绑定的会话状态容器，session variable
 的读写为什么必须通过它、跨线程误用会怎样；
 易错点：qe_max_connection 与单用户配额的两层限流，打满后新连接
 的现象（handshake 后立刻断）与误判方向）

## 1.3 源码走读：COM_QUERY 分发与 StmtExecutor
（MysqlConnectProcessor 命令分发 → StmtExecutor 的 execute 主线（概览级，
 后续各章展开）；
 tricky 点：非 Master FE 收到写语句时的 forward to master 机制——
 查询在本地执行、DDL/DML 转发，读代码时两条路径容易混）

## 1.4 双模式对比
（连接与协议层两模式完全一致；一句注明 + 指出存算分离下多计算组时
 连接接入仍在 FE、与计算组无关，真正分叉从第 5 章计划分发开始）

## 1.5 动手实验
（核心点：`mysql --host ... -P 9030` 连接后用第 1 部分第 5 章的日志方法
 打开 FE debug 日志，观察一条 SELECT 的 accept→auth→dispatch 日志链；
 易错点：把 qe_max_connection 调小（如 5）重启 FE，用多个客户端打满，
 亲眼看新连接被拒的报错样貌，学会与网络故障区分）

## 1.6 排查清单
（症状→路径：连不上（端口/白名单/连接数）/ 连接频繁断（超时参数）/
 权限报错的定位入口）
```

- [ ] **Step 3: 引用校验**

运行 Global Constraints 中的引用校验脚本（FILE=docs/doris-internals/part2-query-lifecycle/01-connection-and-protocol.md）。预期：无 MISSING。人工核对正文全部裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/01-connection-and-protocol.md
git commit -m "[docs] doris-internals part2: ch1 connection and mysql protocol

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 2 章《解析与合法化》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/02-parse-and-analyze.md`

**Interfaces:**
- Consumes: 第 1 章的 `StmtExecutor` 入口；第 1 部分第 5 章"旧 analysis/ AST 已删除"的结论。
- Produces: `LogicalPlan`/`UnboundRelation`/bound plan 概念，第 3、4 章直接使用。

- [ ] **Step 1: 核实素材**

```bash
ls fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/
grep -n "selectClause\|queryTerm\|fromClause" fe/fe-sql-parser/src/main/antlr4/org/apache/doris/nereids/DorisParser.g4 | head -5
grep -n "class NereidsParser\|parseSQL\|parseSingle" fe/fe-core/src/main/java/org/apache/doris/nereids/parser/NereidsParser.java | head
grep -n "class LogicalPlanBuilder" fe/fe-core/src/main/java/org/apache/doris/nereids/parser/LogicalPlanBuilder.java
grep -n "class Analyzer" fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/executor/Analyzer.java
ls fe/fe-core/src/main/java/org/apache/doris/nereids/rules/analysis/ | head -20
grep -rn "class UnboundRelation" fe/fe-core/src/main/java/org/apache/doris/nereids/ --include=*.java | head -2
grep -rn "class BindExpression\|class BindRelation" fe/fe-core/src/main/java/org/apache/doris/nereids/rules/analysis/*.java | head -4
```

- [ ] **Step 2: 写作**

创建 `02-parse-and-analyze.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：解析与合法化 —— 从字符串到 LogicalPlan

## 2.1 问题：SQL 文本怎么变成可优化的树
（三连问：手写递归下降（性能/灵活但维护贵）vs yacc/bison 系
 vs ANTLR 生成（语法即文档，但性能与错误信息要调）；
 Doris 用 ANTLR4 + visitor 构建 LogicalPlan 的考量；
 一段简短历史：旧优化器的手写 AST（SelectStmt 一族）已整体删除，
 引用第 1 部分第 5 章，提醒读者旧博客失效）

## 2.2 源码走读：词法语法与 LogicalPlanBuilder
（DorisLexer.g4/DorisParser.g4 的组织（语法文件在 fe-sql-parser 模块，
 不在 fe-core——本 master 的模块拆分点要点明）；
 NereidsParser 入口 → LogicalPlanBuilder（visitor）把 parse tree 变成
 未绑定的 LogicalPlan（UnboundRelation/UnboundSlot）；
 mermaid 流程图：SQL → token → parse tree → unbound plan → bound plan；
 tricky 点：dialect 兼容（parser/Dialect.java）、反引号与大小写规则；
 易错点：改语法文件后忘了重新生成/编译 fe-sql-parser 模块，
 现象是"语法明明改了却不生效"）

## 2.3 源码走读：Analyze——绑定与合法化
（nereids/jobs/executor/Analyzer.java 驱动 rules/analysis/ 规则批：
 BindRelation（表名→Catalog 对象）、BindExpression（列名→Slot）、
 类型强转、聚合合法性检查等，挑 2-3 个规则详讲；
 tricky 点：列名解析的作用域顺序（同名列在 join 两侧/子查询/别名时
 绑定到谁），错写 SQL 时报错信息怎么读；
 易错点：GROUP BY 里用 SELECT 别名/序号的方言行为差异）

## 2.4 双模式对比
（解析与绑定两模式一致，一句注明；BindRelation 拿到的表对象在
 分离模式下是 CloudTable 系（见第 1 部分 4.4 的判别法），
 但对本环节逻辑无影响）

## 2.5 动手实验
（核心点：EXPLAIN PARSED PLAN / EXPLAIN ANALYZED PLAN 对比同一条 SQL
 的 unbound/bound 形态，对照 2.2/2.3 的类名；
 易错点：构造一条歧义列名 SQL（join 两侧同名列不带前缀）看报错，
 再构造 GROUP BY 别名的例子验证方言行为）

## 2.6 排查清单
（症状→路径：语法报错但 SQL 在 MySQL 能跑（方言差异定位）/
 "Unknown column" 但列明明存在（作用域/大小写）/ 视图嵌套解析报错）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/02-parse-and-analyze.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/02-parse-and-analyze.md
git commit -m "[docs] doris-internals part2: ch2 parse and analyze

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 3 章《Nereids 优化器（上）：RBO 规则重写体系》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/03-nereids-rbo.md`

**Interfaces:**
- Consumes: 第 2 章的 bound LogicalPlan。
- Produces: 规则/Job/fixpoint 概念与 `CascadesContext`，第 4 章在其上讲 memo 与代价搜索。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class NereidsPlanner" fe/fe-core/src/main/java/org/apache/doris/nereids/NereidsPlanner.java
ls fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/rewrite/
grep -rn "class Rewriter" fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/executor/Rewriter.java | head -2
ls fe/fe-core/src/main/java/org/apache/doris/nereids/rules/rewrite/ | head -40
ls fe/fe-core/src/main/java/org/apache/doris/nereids/rules/rewrite/ | wc -l
grep -rn "class PushDownFilter\|PredicatePushDown" fe/fe-core/src/main/java/org/apache/doris/nereids/rules/rewrite/*.java | head -3
grep -rn "class ColumnPruning" fe/fe-core/src/main/java/org/apache/doris/nereids/rules/rewrite/ColumnPruning.java | head -2
grep -n "disable_nereids_rules" fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java | head -3
grep -rn "class RuleType" fe/fe-core/src/main/java/org/apache/doris/nereids/rules/RuleType.java | head -2
```

- [ ] **Step 2: 写作**

创建 `03-nereids-rbo.md`（约 6000-8000 字），结构：

```markdown
# 第 3 章：Nereids 优化器（上）—— RBO 规则重写体系

## 3.1 问题：哪些优化不需要代价就能做
（三连问：优化器为什么分 RBO/CBO 两层——哪些变换"永远不亏"
 （谓词下推、列裁剪、常量折叠、子查询去关联）哪些必须比代价
 （join 顺序、分布方式）；候选：全部进代价搜索（搜索空间爆炸）vs
 启发式一把梭（复杂查询选错计划）vs 分层——先规则重写收敛再代价搜索；
 Doris/Nereids 的分层与业界（Calcite HepPlanner+VolcanoPlanner、
 Cascades 论文）的对应关系）

## 3.2 源码走读：规则框架
（Rule/RuleType/pattern 匹配的骨架；Rewriter 组织规则批（topDown/
 bottomUp/fixedPoint），CascadesContext 承载重写状态；
 挑一条真实规则（如 ColumnPruning 或某个谓词下推规则）逐段读；
 tricky 点：规则批的顺序是精心编排的隐含依赖——某些规则必须在
 另一些之后跑，乱序会怎样（举一个真实的顺序依赖例子）；
 易错点：fixedPoint 规则写得不收敛（每次都"改写成功"）会无限循环，
 框架的迭代上限保护在哪）

## 3.3 源码走读：三类经典重写规则
（谓词下推/列裁剪/子查询去关联各挑核心片段：
 谓词能推过哪些算子、推不过哪些（窗口/limit 边界）；
 子查询去关联为什么难（关联子查询→join 的变换条件）；
 每类给"错写会怎样"：如把不该下推的谓词推过 outer join 导致错误结果——
 这正是历史 bug 高发区，前向链接 part7 案例集）

## 3.4 双模式对比
（RBO 与模式无关，一句注明）

## 3.5 动手实验
（核心点：用 EXPLAIN REWRITTEN PLAN 对比 ANALYZED PLAN，找出谓词下推
 与列裁剪的效果；
 易错点：SET disable_nereids_rules 关闭某条规则（从 RuleType 枚举里
 挑一个安全的），对比计划与执行时间，体会单条规则的价值；
 再用一条 outer join + where 的 SQL 观察谓词被正确地推到哪一侧）

## 3.6 排查清单
（症状→路径：查询突然变慢先看 REWRITTEN PLAN 是否形态异常 /
 升级后计划变化的规则级 bisect 法（disable_nereids_rules）/
 结果错误怀疑重写规则时如何最小化复现）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/03-nereids-rbo.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/03-nereids-rbo.md
git commit -m "[docs] doris-internals part2: ch3 nereids rewrite rules

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 4 章《Nereids 优化器（下）：CBO、统计信息与代价模型》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/04-nereids-cbo.md`

**Interfaces:**
- Consumes: 第 3 章的 CascadesContext 与重写后的 LogicalPlan。
- Produces: 物理计划（PhysicalPlan）与分布属性概念，第 5 章据此讲 Fragment 切分。

- [ ] **Step 1: 核实素材**

```bash
ls fe/fe-core/src/main/java/org/apache/doris/nereids/memo/
grep -n "class Memo\b" fe/fe-core/src/main/java/org/apache/doris/nereids/memo/Memo.java
ls fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/cascades/
ls fe/fe-core/src/main/java/org/apache/doris/nereids/jobs/joinorder/ 2>/dev/null | head
grep -n "class CostModel\|class CostCalculator" fe/fe-core/src/main/java/org/apache/doris/nereids/cost/CostModel.java fe/fe-core/src/main/java/org/apache/doris/nereids/cost/CostCalculator.java
ls fe/fe-core/src/main/java/org/apache/doris/nereids/stats/ | head
grep -rn "class StatsCalculator" fe/fe-core/src/main/java/org/apache/doris/nereids/stats/StatsCalculator.java | head -2
grep -n "class AnalysisManager" fe/fe-core/src/main/java/org/apache/doris/statistics/AnalysisManager.java
grep -rn "enable_auto_analyze" fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java fe/fe-core/src/main/java/org/apache/doris/common/Config.java 2>/dev/null | head -3
ls fe/fe-core/src/main/java/org/apache/doris/nereids/properties/ | grep -i "distribut\|physical" | head
```

- [ ] **Step 2: 写作**

创建 `04-nereids-cbo.md`（约 6000-8000 字），结构：

```markdown
# 第 4 章：Nereids 优化器（下）—— CBO、统计信息与代价模型

## 4.1 问题：join 顺序与分布方式只能比出来
（三连问：n 表 join 的搜索空间（卡特兰数）怎么办——穷举 DP（System R，
 表多爆炸）vs 贪心（快但易错）vs Cascades memo（分组去重+剪枝+
 属性 enforcer 一体化）；分布式还多一维：broadcast vs shuffle vs
 bucket shuffle vs colocate 的选择也要进代价；Nereids 选 Cascades 的考量
 与大表数场景的兜底（join reorder 算法分层））

## 4.2 源码走读：Memo 与 Cascades 搜索
（Memo/Group/GroupExpression 结构（mermaid 图示一条 SQL 的 memo 形态）；
 jobs/cascades/ 的任务分解（OptimizeGroup→OptimizeExpression→
 ApplyRule→DeriveStats）；rules/exploration/（变换）与
 rules/implementation/（物理化）的分工；
 tricky 点：required property / enforcer 机制——分布属性不满足时
 插 Exchange 的决策点，读代码最绕的一块，详细拆解；
 易错点：把 exploration 规则误写成 implementation（或反之）会怎样）

## 4.3 源码走读：统计信息从哪来、怎么算
（statistics/ 的 ANALYZE 链路与自动收集开关；StatsCalculator 沿计划
 自底向上推导行数/NDV；FilterEstimation 的选择率估计；
 tricky 点：估计误差沿 join 树指数放大——为什么"错一层、歪全局"；
 易错点：统计缺失/过期时优化器的默认值行为，broadcast 选错
 直接把小表广播成大表的事故模式）

## 4.4 双模式对比
（CBO 逻辑两模式一致，一句注明；统计信息收集作业的执行位置差异
 简要交代）

## 4.5 动手实验
（核心点：建两张表 join，先不 ANALYZE 看 EXPLAIN 的分布方式与行数估计，
 ANALYZE 后再看变化，对照 4.3 的推导链；
 易错点：DROP STATS 后构造大小表 join，观察 broadcast/shuffle 选择
 是否翻转，把"统计缺失→计划劣化"亲手踩一遍）

## 4.6 排查清单
（症状→路径：join 顺序明显不对（先查统计再查 hint）/
 广播了大表（内存报警伴随）/ ANALYZE 不生效或过期的确认方法）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/04-nereids-cbo.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/04-nereids-cbo.md
git commit -m "[docs] doris-internals part2: ch4 nereids cbo and statistics

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 5 章《计划分发：Fragment 切分与 Coordinator 调度》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/05-plan-distribution.md`

**Interfaces:**
- Consumes: 第 4 章的物理计划与分布属性。
- Produces: Fragment/instance/调度概念，第 6 章从 BE 收到 fragment 后接力。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class PlanFragment\b" fe/fe-core/src/main/java/org/apache/doris/planner/PlanFragment.java
grep -n "class ExchangeNode" fe/fe-core/src/main/java/org/apache/doris/planner/ExchangeNode.java
grep -n "class Coordinator\b\|class NereidsCoordinator" fe/fe-core/src/main/java/org/apache/doris/qe/Coordinator.java fe/fe-core/src/main/java/org/apache/doris/qe/NereidsCoordinator.java
grep -rn "selectBackend\|getHost\|replica" fe/fe-core/src/main/java/org/apache/doris/qe/SimpleScheduler.java | head -5
grep -rln "class CloudReplica" fe/fe-core/src/main/java/org/apache/doris/cloud/ | head -2
grep -n "exec_plan_fragment" be/src/service/internal_service.cpp | head -3
grep -n "exec_plan_fragment" be/src/runtime/fragment_mgr.cpp | head -3
grep -rn "colocate\|bucket" fe/fe-core/src/main/java/org/apache/doris/planner/OlapScanNode.java | head -5
grep -rn "PExecPlanFragment" gensrc/proto/internal_service.proto | head -3
```

- [ ] **Step 2: 写作**

创建 `05-plan-distribution.md`（约 6000-8000 字），结构：

```markdown
# 第 5 章：计划分发 —— Fragment 切分与 Coordinator 调度

## 5.1 问题：一棵计划树怎么摊到一群机器上
（三连问：单点执行（放弃并行）vs 全对称 SPMD（每台跑全计划，
 shuffle 全靠运行时）vs 按 Exchange 边界切 Fragment、每段独立设并行度；
 MPP 系的通行选择与 Doris 的落法；实例数怎么定（并行度×机器数）；
 谁当协调者——FE 直接协调 vs 派驻 BE，Doris 让 FE 的 Coordinator
 承担的取舍（省一跳 vs FE 负载））

## 5.2 源码走读：Fragment 切分与实例展开
（物理计划→PlanFragment 树（切分点=Exchange）；每个 fragment 的
 并行实例展开与参数（mermaid 图：一条两表 join SQL 的 fragment 树
 及实例分布）；
 tricky 点：bucket shuffle join / colocate join 对调度的约束——
 实例必须跟着 bucket 走，这是"计划正确但调度也必须配合"的典型；
 易错点：并行度参数的多个来源（session/表属性/机器数）谁生效）

## 5.3 源码走读：副本选择与下发
（Coordinator/NereidsCoordinator 的执行状态机（概览）；
 SimpleScheduler 选副本：本地性、黑名单、负载；
 brpc 下发 exec_plan_fragment → BE FragmentMgr 接收（交棒第 6 章）；
 tricky 点：副本选择失败/超时的重试语义，哪些错误会自动换副本、
 哪些直接失败）

## 5.4 双模式对比（本章重点段）
（存算一体：3 副本任选，本地性优先，坏副本进黑名单；
 存算分离：tablet→计算组内 BE 的映射由 CloudReplica 计算（一致性哈希类
 机制，核实后落笔），无"3 副本"概念但有 cache 亲和性——同一 tablet
 尽量固定到同一 BE 以命中 File Cache；换计算组=换一批 BE=cache 冷启动；
 对照读 fe/.../cloud/ 下的副本/调度类与一体模式类）

## 5.5 动手实验
（核心点：EXPLAIN 看 fragment 边界，开 profile 跑一次查询，
 在 profile 里数 fragment/instance 数量并与 EXPLAIN 对上；
 易错点：单副本表 stop 掉其所在 BE（或 SET 会话变量禁用某 BE），
 观察查询报错样貌；三副本表做同样动作观察自动换副本——
 把"哪些故障能容忍"亲手验证）

## 5.6 排查清单
（症状→路径：查询报 backend not alive / 实例倾斜（个别 instance 拖尾）/
 分离模式下同一查询忽快忽慢（cache 亲和性被打破）的定位入口）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/05-plan-distribution.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/05-plan-distribution.md
git commit -m "[docs] doris-internals part2: ch5 fragment distribution and scheduling

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 6 章《BE Pipeline 执行引擎》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/06-pipeline-engine.md`

**Interfaces:**
- Consumes: 第 5 章下发到 BE FragmentMgr 的 fragment。
- Produces: Pipeline/PipelineTask/Dependency/Operator 概念，第 7、8 章的算子讲解在此框架上进行。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/exec/pipeline/
grep -n "class PipelineFragmentContext" be/src/exec/pipeline/pipeline_fragment_context.h | head -2
grep -n "class PipelineTask" be/src/exec/pipeline/pipeline_task.h | head -2
grep -n "class TaskScheduler" be/src/exec/pipeline/task_scheduler.h | head -2
grep -n "class Dependency" be/src/exec/pipeline/dependency.h | head -2
ls be/src/exec/operator/ | head -20; ls be/src/exec/operator/ | wc -l
grep -rn "class OperatorBase\|class OperatorXBase" be/src/exec/operator/operator.h 2>/dev/null | head -3; ls be/src/exec/operator/ | grep -i "^operator"
grep -n "parallel_pipeline_task_num" be/src/common/config.cpp fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java 2>/dev/null | head -3
ls be/src/exec/exchange/ 2>/dev/null | head
grep -rn "local exchange" be/src/exec/pipeline/*.h | head -3
```

- [ ] **Step 2: 写作**

创建 `06-pipeline-engine.md`（约 6000-8000 字），结构：

```markdown
# 第 6 章：BE Pipeline 执行引擎

## 6.1 问题：一台机器上怎么跑几百个并发查询片段
（三连问：经典火山模型 + 每 instance 一线程（阻塞算子占着线程睡觉、
 线程数爆炸、cache 不友好）vs 协程 vs pipeline 化——把计划按阻塞点
 切成 pipeline、任务化调度到固定线程池；
 阻塞算子（hash build、sort、agg）为什么是切分点：source/sink 拆分；
 Doris 从火山到 Pipeline（含 PipelineX 演进合入）的动机与收益）

## 6.2 源码走读：从 fragment 到 pipeline
（PipelineFragmentContext 把 plan 树构建成 pipeline 集合：
 算子 XOperator 的 source/sink 两态；mermaid 图：一条 join SQL 的
 fragment→pipelines→tasks 分解；
 tricky 点：算子在 be/src/exec/operator/（平级目录），pipeline/ 只放
 调度设施——目录即架构，找代码别走错；
 易错点：pipeline 之间的依赖（build 完才能 probe）如何表达）

## 6.3 源码走读：依赖驱动调度
（PipelineTask 状态机与 Dependency：ready/blocked 的唤醒机制，
 TaskScheduler 的多队列（task_queue）；
 tricky 点：Dependency 的 set_ready/block 时序——唤醒丢失是这类
 调度器的经典 bug 形态，讲清楚 Doris 怎么避免（原子状态+重检查），
 错写会怎样（task 永久挂起=查询 hang）；
 易错点：在算子里做同步阻塞 IO 而不挂 Dependency，占死调度线程）

## 6.4 并行度与 local exchange
（parallel_pipeline_task_num 的生效链路；local exchange 在单机内
 重分布数据的场景（概览，详见第 8 章倾斜处理））

## 6.5 双模式对比
（执行引擎两模式一致，一句注明；分离模式差异集中在 Scan（第 7 章））

## 6.6 动手实验
（核心点：跑一条 join 查询开 profile，对照 6.2 的分解在 profile 里
 找到每条 pipeline 与 task 数；
 易错点：SET parallel_pipeline_task_num=1 与默认值对比同一查询的
 耗时与 profile 形态，理解并行度的真实作用位置）

## 6.7 排查清单
（症状→路径：查询 hang（先抓 BE 栈看 task 状态）/ CPU 打满但吞吐低
 （调度开销/倾斜）/ 单查询把 BE 打挂的定位入口）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/06-pipeline-engine.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/06-pipeline-engine.md
git commit -m "[docs] doris-internals part2: ch6 pipeline execution engine

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 7: 第 7 章《Scan 路径：本地读与 File Cache 远端读》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/07-scan-path.md`

**Interfaces:**
- Consumes: 第 6 章的 pipeline/scan 算子框架；第 1 部分第 3 章的 Rowset/Segment 层级。
- Produces: Scan 全链路认知；存储层细节（索引、谓词下推的段内执行）前向链接 part5。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/exec/scan/
grep -n "class ScannerContext" be/src/exec/scan/scanner_context.h | head -2
grep -n "class OlapScanner" be/src/exec/scan/olap_scanner.h | head -2
grep -rn "class ScannerScheduler" be/src/exec/scan/*.h | head -2
ls be/src/io/cache/
grep -n "class BlockFileCache\b" be/src/io/cache/block_file_cache.h | head -2
grep -n "class CachedRemoteFileReader" be/src/io/cache/cached_remote_file_reader.h | head -2
grep -rn "enable_file_cache" be/src/common/config.cpp | head -3
grep -rn "file_cache_path" be/src/common/config.cpp | head -2
ls be/src/io/fs/ | grep -i "s3\|remote" | head -5
grep -rn "warm.*up\|warmup" be/src/cloud/*.h 2>/dev/null | head -3
```

- [ ] **Step 2: 写作**

创建 `07-scan-path.md`（约 6000-8000 字），结构：

```markdown
# 第 7 章：Scan 路径 —— 本地读与 File Cache 远端读

## 7.1 问题：扫描为什么要从执行线程里拆出去
（三连问：scan 直接在 pipeline task 里做（IO 阻塞占调度线程）vs
 异步 scanner 线程池 + 生产者消费者（解耦 IO 与计算，但要背压）；
 scanner 并发怎么定、内存怎么控（背压机制）；
 Doris 的 ScannerScheduler/ScannerContext 方案）

## 7.2 源码走读：存算一体本地读
（scan 算子→ScannerContext→scanner 线程池→OlapScanner→存储层读接口
 （段内细节留给 part5，给前向链接）；块在生产者消费者队列中的流动；
 tricky 点：scanner 数量与队列内存的背压参数，配错的现象；
 易错点：tablet 多而小时 scanner 调度开销反超 IO 的反直觉场景）

## 7.3 源码走读：存算分离远端读与 File Cache（本章重点段）
（CachedRemoteFileReader 的读路径：cache 命中→本地块设备；
 miss→对象存储读并回填；BlockFileCache 的块管理/淘汰（TTL、
 索引/数据分区缓存的类型区分，核实后落笔）；
 mermaid 图：一次 miss 读的完整路径（BE→cache→S3→回填→返回）；
 tricky 点：cache 以 block 为粒度而非文件——大文件部分命中的行为；
 易错点：file_cache_path 容量配置与磁盘实际的关系、cache 满后的
 淘汰抖动，冷查询延迟为什么会周期性出现）

## 7.4 双模式对比
（本章即双模式对比主体：同一条 SQL 的 scan 在两模式的路径分叉图；
 性能特征对比：本地盘稳定低延迟 vs cache 命中相当/miss 慢一个数量级；
 运维含义：分离模式容量规划=算 cache 命中率）

## 7.5 动手实验
（核心点（一体模式即可做）：开 profile 看 scan 相关指标（scanner 数、
 读行数、谓词过滤行数），对照 7.2 链路；
 易错点（有分离环境则做，否则注明依赖 part4 实验环境）：清空/缩小
 File Cache 后跑同一查询，对比冷热两次的 profile 中远端读指标，
 亲眼看 miss 的代价）

## 7.6 排查清单
（症状→路径：scan 慢先分谓词过滤少还是 IO 慢 / 分离模式冷查询慢
 （cache 命中率指标在哪看）/ 对象存储限流报错的识别）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/07-scan-path.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/07-scan-path.md
git commit -m "[docs] doris-internals part2: ch7 scan path and file cache

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 8: 第 8 章《Join / 聚合 / 排序算子与 Runtime Filter、Spill》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/08-operators-rf-spill.md`

**Interfaces:**
- Consumes: 第 6 章的算子框架（source/sink 两态）。
- Produces: 核心算子与 RF/Spill 认知，part6 的查询故障排查直接引用。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/exec/operator/ | grep -iE "hash_join|aggregation|sort" | head -15
grep -n "class HashJoinBuildSinkOperatorX\|class HashJoinProbeOperatorX" be/src/exec/operator/hash_join_build_sink_operator.h be/src/exec/operator/hash_join_probe_operator.h 2>/dev/null | head -4
ls be/src/exec/runtime_filter/ | head -15
grep -rn "class RuntimeFilter\b\|IN_FILTER\|BLOOM_FILTER\|MINMAX" be/src/exec/runtime_filter/*.h | head -6
grep -rn "runtime_filter_wait_time_ms" be/src/common/config.cpp fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java 2>/dev/null | head -3
ls be/src/exec/spill/ | head
ls be/src/exec/operator/ | grep -i "partitioned" | head -8
grep -rn "revocable\|revoke" be/src/exec/pipeline/revokable_task.h 2>/dev/null | head -3
grep -rn "class WorkloadGroup" be/src/runtime/workload_group/*.h 2>/dev/null | head -2
```

- [ ] **Step 2: 写作**

创建 `08-operators-rf-spill.md`（约 6000-8000 字），结构：

```markdown
# 第 8 章：Join / 聚合 / 排序算子与 Runtime Filter、Spill

## 8.1 问题：内存不够 join 一张大表怎么办
（三连问贯穿三件事：
 a. hash join 在 pipeline 下的拆分：build sink + probe source 两条
 pipeline 的依赖关系；
 b. runtime filter：probe 侧扫描浪费的行能不能提前拦——候选：不拦/
 静态谓词/运行时从 build 侧生成过滤器回推 scan；in/bloom/minmax
 三种过滤器的适用与代价；
 c. 内存放不下：直接 OOM vs 全量预留 vs spill 落盘——partitioned
 hash join/agg/sort 的分区落盘设计）

## 8.2 源码走读：三大算子
（hash join build/probe 算子核心片段（哈希表构建、探测输出）；
 aggregation sink/source；sort 的全排/topn 分支；
 每个算子只讲"pipeline 两态怎么协作"与关键数据结构，
 详略：哈希表实现细节一笔带过；
 tricky 点：join 匹配语义（null-aware、outer join 的未匹配输出）
 在两态拆分下的实现位置）

## 8.3 源码走读：Runtime Filter 全链路
（build 侧生成→（本地/全局 merge）→下发到 probe 侧 scan；
 mermaid 时序图；
 tricky 点：RF 是"尽力而为"——wait time 超时后 scan 不等直接跑，
 所以 RF 失效表现为"慢而不是错"；merge 的全局协调角色；
 易错点：wait 时间配置与 join 左右表颠倒时 RF 完全无效的场景）

## 8.4 源码走读：Spill
（触发条件（内存水位/query limit）；partitioned 算子的分区落盘与
 回读（be/src/exec/spill/ 的文件管理）；
 tricky 点：spill 是按分区递归的——单分区仍放不下时的降级行为；
 易错点：spill 打开后性能断崖是预期行为，与"内存泄漏"区分）

## 8.5 双模式对比
（算子逻辑两模式一致，一句注明；spill 落盘位置在分离模式下
 仍是 BE 本地盘（核实后落笔）——这是"分离模式 BE 并非完全无状态"
 的又一例证，呼应第 1 部分 4.3）

## 8.6 动手实验
（核心点：大小表 join 开 profile，找到 RF 的生成/下发/过滤行数指标，
 再把 runtime filter 关掉（session 变量）对比 scan 输出行数；
 易错点：SET exec_mem_limit（或 query 级内存参数，核实当前变量名）
 调小强制触发 spill，对比 profile 中 spill 指标与耗时，
 学会识别"spill 导致的慢"）

## 8.7 排查清单
（症状→路径：join 慢先看 RF 是否生效（profile 指标名）/
 内存超限报错读法（哪个算子申请的）/ 倾斜（单 instance 拖尾）
 与 spill 的 profile 特征区分）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/08-operators-rf-spill.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/08-operators-rf-spill.md
git commit -m "[docs] doris-internals part2: ch8 operators runtime-filter spill

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 9: 第 9 章《结果回传与 Profile 精读》

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/09-result-and-profile.md`

**Interfaces:**
- Consumes: 前 8 章的完整执行路径。
- Produces: Profile 精读方法论，part6 查询故障篇的基础工具。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class ResultBufferMgr" be/src/runtime/result_buffer_mgr.h | head -2
grep -n "class ResultBlockBuffer" be/src/runtime/result_block_buffer.h | head -2
ls be/src/exec/sink/ 2>/dev/null | grep -i result | head -3
grep -rn "fetch_data" be/src/service/internal_service.cpp | head -3
grep -rn "class RuntimeProfile" be/src/runtime/runtime_profile.h | head -2
ls fe/fe-core/src/main/java/org/apache/doris/common/profile/
grep -n "class Profile\b\|class ExecutionProfile" fe/fe-core/src/main/java/org/apache/doris/common/profile/Profile.java fe/fe-core/src/main/java/org/apache/doris/common/profile/ExecutionProfile.java | head -4
grep -rn "enable_profile" fe/fe-core/src/main/java/org/apache/doris/qe/SessionVariable.java | head -3
grep -rn "SHOW QUERY PROFILE\|show_query_profile" fe/fe-core/src/main/java/org/apache/doris/ -r --include=*.java -l | head -3
```

- [ ] **Step 2: 写作**

创建 `09-result-and-profile.md`（约 6000-8000 字），结构：

```markdown
# 第 9 章：结果回传与 Profile 精读

## 9.1 问题：结果从一群 BE 回到一个客户端
（三连问：各 BE 直接回客户端（协议不允许，MySQL 连接在 FE）vs
 全部汇到 coordinator fragment 再经 FE 转发 vs BE 结果缓冲 + FE 拉取；
 Doris 的 ResultSink→ResultBlockBuffer→FE fetch→MySQL 协议编码路径；
 背压：客户端不取时 buffer 满了怎么办）

## 9.2 源码走读：回传链路
（BE 侧 result sink 与 ResultBufferMgr/ResultBlockBuffer；
 FE 经 brpc fetch_data 拉取并编码为 MySQL 包回写 channel（呼应第 1 章）；
 tricky 点：cancel 传播——客户端断开/查询取消时，从 FE 到所有 BE
 fragment 的取消链路，为什么有时"客户端已断查询还在跑"）

## 9.3 Profile 体系：结构与生成
（BE 每算子计数器→RuntimeProfile 树→上报 FE 合并（ExecutionProfile/
 Profile）；打开方式（enable_profile 及 web/命令入口）；
 tricky 点：profile 是采样合并的产物，个别 instance 缺失的含义）

## 9.4 Profile 精读方法论（本章重点段）
（给一份真实（脱敏/自造）的慢查询 profile，分段精读：
 从总耗时→找最长 fragment→找最长算子→读关键计数器；
 常见瓶颈模式对照表：scan IO 慢 / RF 未生效 / 倾斜 / spill /
 exchange 网络慢 / 结果回传慢——每种模式给出 profile 上的指纹特征；
 这一节是全章价值核心，篇幅向它倾斜）

## 9.5 双模式对比
（回传与 profile 两模式一致，一句注明；分离模式 profile 里多出的
 cache 相关指标呼应第 7 章）

## 9.6 动手实验
（核心点：构造一条含 join+agg 的查询，完整走一遍 9.4 的精读流程，
 写出自己的瓶颈结论；
 易错点：构造数据倾斜（一个 key 占 90%）跑 join，在 profile 里
 找到倾斜的指纹（instance 间 max/min 耗时差），与均匀数据对比）

## 9.7 排查清单
（症状→路径：客户端等很久无结果（区分执行慢/回传卡/客户端不取）/
 查询取消不掉 / profile 拿不到（开关/过期）的处理）
```

- [ ] **Step 3: 引用校验**

同 Global Constraints 脚本（FILE=docs/doris-internals/part2-query-lifecycle/09-result-and-profile.md）。预期：无 MISSING。人工核对裸类名。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/09-result-and-profile.md
git commit -m "[docs] doris-internals part2: ch9 result fetch and profile reading

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 10: 部分目录页 + 系列 README 翻转 + part1 锚点更新

**Files:**
- Create: `docs/doris-internals/part2-query-lifecycle/README.md`
- Modify: `docs/doris-internals/README.md`（第二部分状态"规划中"→"已完成"，章节列表改为链接）
- Modify: `docs/doris-internals/part1-architecture/README.md`（更新第 17 行附近的第二部分预告锚点——第一批遗留 fix-later 项：原锚点指向含"（规划中）"的标题，翻转后会失效）

**Interfaces:**
- Consumes: Task 1-9 产出的 9 个章节文件。
- Produces: 完整可导航的第二部分。

- [ ] **Step 1: 核实现状**

```bash
ls docs/doris-internals/part2-query-lifecycle/
grep -n "第二部分" docs/doris-internals/README.md
grep -n "第二部分" docs/doris-internals/part1-architecture/README.md
```

- [ ] **Step 2: 写作部分目录页**

创建 `part2-query-lifecycle/README.md`（约 500 字）：

```markdown
# 第二部分：一条查询 SQL 的一生

（两三句导语：沿一条 SELECT 的完整路径讲查询内核，
 前 5 章在 FE、后 4 章在 BE，第 5/7 章是双模式差异的主战场）

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-connection-and-protocol.md) | 连接与协议 | 查询入口与 ConnectContext |
| [第 2 章](02-parse-and-analyze.md) | 解析与合法化 | SQL→LogicalPlan |
| [第 3 章](03-nereids-rbo.md) | Nereids RBO | 规则重写体系 |
| [第 4 章](04-nereids-cbo.md) | Nereids CBO | Memo、统计与代价 |
| [第 5 章](05-plan-distribution.md) | 计划分发 | Fragment 与副本选择 |
| [第 6 章](06-pipeline-engine.md) | Pipeline 引擎 | 依赖驱动调度 |
| [第 7 章](07-scan-path.md) | Scan 路径 | 本地读 vs File Cache |
| [第 8 章](08-operators-rf-spill.md) | 核心算子 | Join/RF/Spill |
| [第 9 章](09-result-and-profile.md) | 结果与 Profile | 瓶颈定位方法论 |

（下一部分预告：第三部分《一次导入的一生》——注意锚点写法要在
 系列 README 当前标题上核实）
```

- [ ] **Step 3: 系列 README 翻转与 part1 锚点更新**

修改 `docs/doris-internals/README.md`：第二部分标题"（规划中）"→"（已完成）"，9 行章节列表改为 `[标题](part2-query-lifecycle/0X-xxx.md)——副题` 格式（对照第一部分已完成的写法）；修改 `part1-architecture/README.md` 的第二部分预告：优先直接链接 `../part2-query-lifecycle/README.md`（比标题锚点稳健），并核对无其他文件引用旧锚点：

```bash
grep -rn "第二部分一条查询" docs/doris-internals/ | grep -v "README.md:"
```

- [ ] **Step 4: 全量引用与链接校验**

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

预期：无输出。

- [ ] **Step 5: 提交**

```bash
git add docs/doris-internals/part2-query-lifecycle/README.md \
        docs/doris-internals/README.md \
        docs/doris-internals/part1-architecture/README.md
git commit -m "[docs] doris-internals part2: part index, series index flip to complete

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：最终整分支审查（含台账 Minor triage），然后向作者简报第二部分完成情况（设计文档第 8 节约定"每部分完成后简报进度"），继续第三部分计划制定无需等待审阅（第一部分的风格关口已过）。
