# 第二部分：一条查询 SQL 的一生

第一部分给了你一张"系统长什么样"的静态地图；这一部分换成纵向视角，跟着**一条 SELECT** 从客户端敲下回车走到结果集回到屏幕，看它中途经过哪些对象、做了哪些决定。九章分两段：前五章在 FE，把 SQL 从字节一路打磨成可调度的 Fragment（连接接入 → 解析绑定 → RBO 重写 → CBO 选物理计划 → 切分分发）；后四章在 BE，讲这些 Fragment 怎么被 Pipeline 引擎调度、怎么扫数据、怎么跑重算子、怎么把结果汇回并留下 Profile。其中第 5 章（副本选择）与第 7 章（Scan 路径）是本部分双模式差异的主战场，必须完整展开；其余各章两模式路径基本一致，只在差异处点到。

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-connection-and-protocol.md) | 连接与协议 | 为何兼容 MySQL 线协议；握手/认证/限流各归谁管；`COM_QUERY` 如何分发到 `StmtExecutor` |
| [第 2 章](02-parse-and-analyze.md) | 解析与合法化 | ANTLR4 解析、AST 到未绑定/已绑定 `LogicalPlan` 的形态变换与绑定期检查 |
| [第 3 章](03-nereids-rbo.md) | Nereids 优化器（上） | RBO 等价重写、规则批的隐含依赖与收敛、谓词下推在 outer join 的陷阱 |
| [第 4 章](04-nereids-cbo.md) | Nereids 优化器（下） | Memo 搜索空间、统计与代价模型、分布属性与 Exchange 插入决策 |
| [第 5 章](05-plan-distribution.md) | 计划分发 | Fragment 在 Exchange 处切分、并行实例展开、两模式副本选择分叉 |
| [第 6 章](06-pipeline-engine.md) | BE Pipeline 执行引擎 | 阻塞点切分 pipeline、`Dependency` 依赖驱动调度、禁止同步阻塞 IO |
| [第 7 章](07-scan-path.md) | Scan 路径 | 异步 scanner + 背压、本地直读 vs File Cache 远端读、命中率与容量规划 |
| [第 8 章](08-operators-rf-spill.md) | 核心算子 | Join/聚合/排序的两态拆分、Runtime Filter 与 Spill 的 tricky 语义 |
| [第 9 章](09-result-and-profile.md) | 结果与 Profile | 结果 pull 链与取消传播、Profile 生成、按指纹定位瓶颈的方法论 |

读完这九章，你应当能画出一条查询从连接到结果的端到端路径，判断任意一段执行代码在哪种模式下生效，并能读一份慢查询 Profile 反推瓶颈——这套 Profile 精读方法是第六部分查询故障排查的直接基础。

**下一部分预告**：第三部分《一次导入的一生》——从事务模型与 Label 机制，沿 Stream Load 的 HTTP 接入 → 计划 → Sink → MemTable → Flush 走完一次写入的完整生命周期，直到 Compaction。规划详见[系列总目录](../README.md)。
