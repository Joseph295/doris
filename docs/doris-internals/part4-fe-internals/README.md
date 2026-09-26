# 第四部分：元数据与 FE 内核

前两条主线（part2 查询、part3 导入）每往前一步，都要反复追问 FE 同一类问题：这个库存在吗？这张表的 schema 是什么？这个 tablet 的副本落在哪些 BE、可见版本是多少？这一部分转到 FE 内核，主线是**跟着一条元数据走完它的一生**——从诞生在 Master 的 JVM 堆里，到落盘与复制、到选主高可用、到（分离模式下）外移进 MetaService、再到被调度器消费。六章分三段：第 1、2、3 章走完存算一体下元数据的内存组织、持久化与高可用主干（Catalog 内存树 → EditLog/bdbje/Checkpoint → 选主与角色）；第 4 章是**存算分离元数据专章**（MetaService / FDB 布局 / Recycler）；第 5 章**双模式并重**讲调度，第 6 章收尾于外部数据源联邦查询。

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-catalog-and-memory.md) | Catalog 体系与元数据内存结构 | 元数据在 JVM 堆里的对象树（Env→Database→Table→Tablet）、为何同一个库既按 id 又按 name 建双索引、`TabletInvertedIndex` 倒排为何要单独建、三层读写锁如何隔离"高并发规划"与"低频 DDL 改结构" |
| [第 2 章](02-editlog-and-checkpoint.md) | 元数据持久化：EditLog、bdbje 与 Checkpoint | 一次 DDL 的日志之旅、bdbje 多数派复制、Checkpoint 用**同一 JVM 进程内的影子 Env** 成像——内存近似翻倍是真实且无法回避的，靠内存紧张时跳过本轮 checkpoint 兜底而非独立进程规避 |
| [第 3 章](03-fe-ha.md) | FE 高可用：选主、角色与故障切换 | bdbje 的第二重身份是选主基础；Master/Follower/Observer 三角色分工、切主后新主如何顶上、请求转发与读写路径，以及这套机制在运维里会咬人的场景 |
| [第 4 章](04-metaservice-fdb.md) | 存算分离元数据：MetaService、FDB 布局与 Recycler | **分离模式专章**：重且高频变的 tablet/版本/事务元数据外移到 MetaService（背后 FoundationDB），`cloud/src/meta-store/keys.h` 的 key 编码布局；FDB 单事务约 **5s / 10MB 硬限**（`transaction_too_old`/`transaction_too_large`）及分批+lazy 收尾的规避；Recycler 异步回收与计算组/节点管理 |
| [第 5 章](05-scheduling.md) | 调度体系：Tablet 均衡、副本修复与计算组管理 | **双模式并重**：存算一体 `TabletScheduler` 做副本修复 + 数据搬迁式均衡、三种均衡器分工；存算分离无本地副本，`CloudTabletRebalancer` 做 tablet→BE 映射再均衡 + cache 预热 |
| [第 6 章](06-external-catalog.md) | 外部数据源：Catalog 联邦查询架构概览 | 为何选"统一 Catalog 抽象 + 按源实现"而非搬数据/逐源专用入口；`CatalogFactory` 内建类型走 switch，而 **jdbc/es 走 SPI 连接器插件**路径；外部元数据多层缓存及各层失效由谁触发；"外表查不到新分区/schema 变更报错"对应哪一层缓存没刷 |

读完这六章，你应当能画出一条元数据从内存诞生、落盘复制、到被调度器消费的完整生命周期，判断任意一段 FE 代码在存算一体还是存算分离下生效，并能从"FE 起不来""选主异常""元数据损坏""外表查不到新数据"这些症状反推到具体环节——这套 FE 内核坐标系是第六部分 FE 故障排查的直接基础。

**下一部分预告**：第五部分《存储引擎深潜》——把 part2/part3 里一带而过的 BE 存储侧挖深：Rowset/Segment 文件格式、索引体系（前缀索引 / ZoneMap / BloomFilter / 倒排）、读路径内核与主键模型 Delete Bitmap 的全生命周期。该部分仍在规划中，章节规划详见[系列总目录](../README.md)。
