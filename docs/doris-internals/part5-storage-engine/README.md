# 第五部分：存储引擎深潜

part2 查询、part3 导入走到 BE 存储侧时都在同一处一带而过：数据到底怎么摆在盘上、一次读取又怎么把它捞回内存。这一部分把这块黑盒彻底打开，主线是**从一个字节的物理布局到一次读取的完整旅程**。六章分三段：第 1、2、3 章是"数据本身与怎么读"的主干（文件格式 → 索引体系 → 读路径内核）；第 4、5 章讲不可变文件上的"可变"操作（MoW 的 delete bitmap、DELETE/Schema Change/分区生命周期）；第 6 章是**存算分离存储专章**，也是整部收官，章末用一张"数据文件的一生"对照表把写入落地、读取来源、合并产物、删除回收四条路径在两种模式下的走向缝成一图。前五章的存储格式与读逻辑两模式共享同一套实现，差异集中收束在第 6 章。

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-segment-format.md) | Rowset 与 Segment 文件格式 | Footer 为何在文件尾、"页"是贯穿数据/索引/字典/短key 的原子单位；编码与压缩分两层且先编码后压缩；字典按列共享一份字典页，单页字典满会**页级退化**成 PLAIN，但 `ColumnMetaPB` 的 `encoding` 仍记声明的 DICT——真实编码看数据页页首 4 字节 |
| [第 2 章](02-indexes.md) | 索引体系：四把裁剪的刀 | 前缀索引/ZoneMap/BloomFilter/倒排各答一问；前缀索引受"最左前缀"约束，排序键**列序**是实战关键；ZoneMap 裁剪对应的 profile 计数器反直觉地叫 `RowsStatsFiltered`（不是 RowsZoneMapFiltered）；BF 只服务等值、NGram BF 才服务 `LIKE` |
| [第 3 章](03-read-path.md) | 读路径内核：谓词下推、延迟物化与合并读 | 谓词列先读、非谓词列按行号回捞是段内默认形态，由"输出列数>谓词列数 且 有段内谓词"这个**纯结构条件**决定——**无 session 开关、无基于选择率的自适应**；高选择率宽表下延迟物化反而更慢，只能靠建模规避；上层 DUP 直通、AGG 读时聚合、UNIQUE-MoR 读时去重 |
| [第 4 章](04-mow-internals.md) | 主键模型内核：Delete Bitmap 全生命周期 | delete bitmap 按 `(RowsetId, SegmentId, Version)` **三元组键**组织、value 是 Roaring bitmap；两阶段计算（commit 摊大头、publish 补 delta）是延迟与正确性的折中；串行化两模式不同——一体靠 publish 天然串行 + tablet 级本地 mutex，分离靠 MetaService 的**表级**分布式锁（partition 维为 -1） |
| [第 5 章](05-data-management.md) | 数据管理：Delete、Schema Change 与分区生命周期 | 三件事共用**版本机制**绕开不可变；普通模型 DELETE 写 delete 谓词由 compaction 物化（`enable_delete_when_cumu_compaction` 默认 false，谓词一路等到 base compaction），MoW 走 bitmap；Schema Change 三档：light（秒级）、SchemaChangeJobV2 影子双写、元数据级轻量变更 |
| [第 6 章](06-cloud-storage.md) | 存算分离存储：File Cache 内核与对象存储交互 | **分离模式专章**：四条 LRU 队列容量可弹性借还；下载去重不是无锁 CAS，而是块 mutex 下的 downloader 认领；`S3FileWriter` 小文件走单次 PutObject、大对象才开 multipart；multipart 半途失败的**孤儿分片 BE 不主动 abort**，责任落在桶 lifecycle 规则而非 Recycler |

读完这六章，你应当能从磁盘上一个字节出发，画出它经过页寻址、索引裁剪、谓词下推、延迟物化、多版本合并直到进入内存 `Block` 的完整读取路径，判断任意一段存储引擎代码在哪种模式下生效，并能从"点查慢""DELETE 后越查越慢""改列类型 job 卡住""缓存命中率突降"这些症状反推到具体环节——这套坐标系是第六部分存储与缓存故障排查的直接基础。

**下一部分预告**：第六部分《集群运维与故障排查》——把前五部分建立的内核坐标系转成运维视角，从症状到根因给出系统化的故障定位方法论。该部分仍在规划中，章节规划详见[系列总目录](../README.md)。
