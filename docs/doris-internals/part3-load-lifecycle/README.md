# 第三部分：一次导入的一生

第二部分跟着一条 SELECT 走完了查询链路，那条链路的前提是"数据已经在库里"。这一部分换到写入侧，跟着**一次 Stream Load** 从客户端 `curl` 灌数据走到数据可见：FE 开事务、BE 攒 memtable、刷 rowset，再到 publish 点亮版本，直到后台 Compaction 把碎版本合并收口。六章分三段：第 1 章打事务地基（批级事务 + Label 幂等 + 2PC 的状态机），第 2、3 章走完 Stream Load 这条基线主干（HTTP 接入分发 → tablet 写入落盘），第 4、5、6 章处理提交可见、其他导入方式与后台合并。其中第 1、4、6 章是本部分双模式差异的主战场——事务语义、Publish vs MetaService 提交、Compaction 执行位置，必须完整展开；第 6 章同时收束贯穿全部分的 `-235 TOO_MANY_VERSION` 主线。

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-load-overview-and-txn.md) | 导入方式总览与事务模型 | 为何用批级事务 + Label 幂等 + 2PC 而非逐行幂等；五种导入方式的分类；`TransactionState` 状态机与双模式事务语义差异 |
| [第 2 章](02-stream-load-path.md) | Stream Load 全路径 | HTTP 重定向与认证透传、导入计划的形状、`OlapTableSink` 分桶分发、`max_filter_ratio` 容错——直到数据送到 `DeltaWriter` 门口 |
| [第 3 章](03-tablet-write-path.md) | Tablet 写入细节 | `DeltaWriter` 攒 memtable→flush→Segment 主干、主键表 Delete Bitmap 预算；导入"变慢"的三类根因（flush 跟不上、内存反压、写放大） |
| [第 4 章](04-commit-and-visibility.md) | 事务提交与可见性 | `COMMITTED ≠ VISIBLE` 的源码兑现；存算一体 `PublishVersionDaemon` 逐 BE publish vs 存算分离 MetaService 一次 FDB 事务；两种模式的延迟与失败模式 |
| [第 5 章](05-other-load-paths.md) | 其他导入方式 | Broker / Routine / Insert Into / Group Commit 与 Stream Load 共用同一写入内核，差异只在触发方与数据源接入层 |
| [第 6 章](06-compaction.md) | Compaction | 为何 LSM 类系统绕不开合并；base / cumulative / full 策略；双模式执行位置；`-235` 不是导入的病，而是后台合并的告警灯 |

读完这六章，你应当能画出一批数据从 `curl` 到可见、再到被后台合并的完整写入路径，判断任意一段写入代码在哪种模式下生效，并能从"导入变慢""事务卡住""-235 告警"这些症状反推到具体环节——这套导入路径与故障坐标系是第六部分导入类故障排查的直接基础。

**下一部分预告**：第四部分《元数据与 FE 内核》——把前两条主线里一带而过的 FE 侧机制挖深：Catalog 内存结构、EditLog / bdbje 持久化、选主与高可用，以及存算分离下 MetaService 的元数据服务内核。规划详见[系列总目录](../README.md)。
