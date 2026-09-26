# 第五部分：存储引擎深潜

part2 查询、part3 导入走到 BE 存储侧都在同一处一带而过：数据怎么摆在盘上、一次读取又怎么把它捞回内存。这一部分把黑盒打开，主线是**从一个字节的物理布局到一次读取的完整旅程**。六章分三段：第 1、2、3 章是"数据本身与怎么读"的主干（文件格式 → 索引体系 → 读路径内核）；第 4、5 章讲不可变文件上的"可变"操作；第 6 章是**存算分离存储专章**兼整部收官，章末用"数据文件的一生"对照表把两模式差异缝成一图。前五章两模式共享同一套实现，差异收束在第 6 章。

| 章节 | 主题 | 关键收获 |
|---|---|---|
| [第 1 章](01-segment-format.md) | Rowset 与 Segment 文件格式 | 字典按列共享一份字典页，单页字典满会**页级退化**成 PLAIN，但 `ColumnMetaPB` 的 `encoding` 仍记声明的 DICT——真实编码看数据页页首 4 字节 |
| [第 2 章](02-indexes.md) | 索引体系：四把裁剪的刀 | 前缀索引受"最左前缀"约束，排序键**列序**是实战关键；ZoneMap 裁剪对应的 profile 计数器叫 `RowsStatsFiltered` 而非 RowsZoneMapFiltered |
| [第 3 章](03-read-path.md) | 读路径内核：谓词下推、延迟物化与合并读 | 谓词列先读、非谓词列按行号回捞由"输出列数>谓词列数 且 有段内谓词"这个**纯结构条件**决定——**无 session 开关、无基于选择率的自适应**，高选择率宽表下反而更慢 |
| [第 4 章](04-mow-internals.md) | 主键模型内核：Delete Bitmap 全生命周期 | delete bitmap 按 `(RowsetId, SegmentId, Version)` **三元组键**组织；串行化两模式不同——分离靠 MetaService 的**表级**分布式锁（partition 维为 -1），一体靠 publish 天然串行 |
| [第 5 章](05-data-management.md) | 数据管理：Delete、Schema Change 与分区生命周期 | 三件事共用**版本机制**绕开不可变；普通模型 DELETE 写 delete 谓词由 compaction 物化（`enable_delete_when_cumu_compaction` 默认 false，谓词一路等到 base compaction），MoW 走 bitmap |
| [第 6 章](06-cloud-storage.md) | 存算分离存储：File Cache 内核与对象存储交互 | 下载去重不是无锁 CAS，而是块 mutex 下的 downloader 认领；multipart 半途失败的**孤儿分片 BE 不主动 abort**，责任落在桶 lifecycle 而非 Recycler |

读完这六章，你应当能从磁盘上一个字节出发，画出它经页寻址、索引裁剪、谓词下推、延迟物化、多版本合并直到进入内存 `Block` 的完整读取路径，判断任意一段存储引擎代码在哪种模式下生效，并能从"点查慢""缓存命中率突降"等症状反推到具体环节——这是第六部分存储与缓存故障排查的直接基础。

**下一部分预告**：[第六部分《集群运维与故障排查》](../part6-operations/README.md)——把前五部分的内核坐标系转成运维视角，给出从症状到根因的故障定位方法论。
