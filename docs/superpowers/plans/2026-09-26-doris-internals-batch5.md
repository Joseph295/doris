# 《Doris 内核透视》第五批交付物实施计划（第五部分：存储引擎深潜）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第五部分（存储引擎深潜）全部 6 章及部分目录页，并将系列 README 的第五部分状态翻转为"已完成"。

**Architecture:** 纯文档写作项目。每章按设计文档第 5 节五段式骨架写作；主线是"从一个字节的物理布局到一次读取的完整旅程"：文件格式（ch1）→索引体系（ch2）→读路径（ch3）→主键内核（ch4）→数据管理（ch5）→分离模式存储（ch6）。part2 第 7 章讲过 scan 的调度层、part3 第 3 章讲过写入落盘、part3 第 6 章讲过 compaction——本部分深入它们当时"留给 part5"的段内/页内细节，写作时先 grep 前部对应章节确认已讲内容，按差异深化不重复。每任务流程固定："核实素材 → 写作 → 引用校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid 图、对本仓库源码的 `文件:行号` 引用。

## Global Constraints

以下约束沿用前四批并补充第五部分锚点，对每个任务生效：

- 语言：中文；类名/函数名/日志/代码保留英文。
- **原理部分三连问**（严禁泛泛而谈）：遇到了什么问题？→ 有哪些候选方案、各有什么优劣？→ Doris 最终怎么考量和解决的？
- **源码走读分清主次**：显而易见的高度概括；不易理解的详细逐段解释；重点挖掘易错点/tricky 点并解释"为什么这么写、错写会怎样"。
- **动手实验双目的**：验证核心点 + 主动踩一遍易错点。环境说明引用 part1 第 5 章。
- 双模式并重：ch6 是分离模式专章；ch1-5 的存储格式与读逻辑两模式一致（数据文件相同），在各章双模式小节一句注明并指出差异集中于 ch6。
- 所有源码引用格式为 `路径:行号` 或 `路径`，必须在当前 master 真实存在；**先核实再落笔**。
- **裸类名陷阱**：正文类名/方法名逐一 `grep -rn "class Xxx"` 核实；**反引号内禁写"类名.方法名"**（门禁会误判为 .h 文件），用 `ClassName` 的 `method()` 分拆式（批 4 教训）。
- **配置可变性陷阱**：BE 配置核实 `DEFINE_mXxx`；FE 配置核实 `@ConfField(mutable)`。
- **跨部分引用惯例**：已成文部分首次提及用相对 md 链接+节号纯文本；未成文部分（part6/7）纯文本；**回引前必须 grep 目标文件确认内容真实存在**（批 3 教训：4 处虚假回引）。
- **前部已覆盖、本部须深化不重复的内容**（写作前先读对应章节）：
  - part2 第 7 章：scan 调度层（ScannerContext/scanner 池/背压）与 File Cache 使用面（命中/回填/淘汰队列）——ch3 从 segment 迭代器接力、ch6 深入 cache 内核实现
  - part3 第 3 章：DeltaWriter→MemTable→Flush 与 delete bitmap 写入面两阶段——ch4 深入 bitmap 全生命周期
  - part3 第 6 章：compaction 调度与版本替换——ch5/ch4 只讲与 schema change/bitmap 的交互差异
  - part1 第 3 章：Rowset/Segment 层级与版本区间——ch1 直接引用层级、深入文件内部
- 已核实的第五部分关键锚点（目录/文件级已核实，行号需现场核实）：
  - Segment 格式：`be/src/storage/segment/`（`segment_writer.cpp/h`、`column_writer.cpp/h`、`segment.h`、`segment_iterator.cpp/h`、`segment_loader.cpp`、各种 page：`binary_dict_page.cpp`、`bitshuffle_page.cpp`、`binary_prefix_page.cpp` 等 72 个文件）；`gensrc/proto/segment_v2.proto`（`PagePointerPB:24`、`ColumnMetaPB:189`、`SegmentFooterPB:265`，行号需复核）
  - 索引体系：`be/src/storage/index/`（`short_key_index.cpp/h`、`zone_map/zone_map_index.cpp/h`、`bloom_filter/`（`bloom_filter_index_reader.cpp`、`ngram_bloom_filter.cpp`、`block_split_bloom_filter.h`）、`inverted/`（analyzer/char_filter 子目录、`inverted_index_cache.cpp`））
  - 读路径：`be/src/storage/segment/segment_iterator.cpp`（谓词下推/延迟物化核心）、`column_reader.cpp`、`lazy_init_segment_iterator.cpp`、`be/src/storage/rowset/beta_rowset_reader.cpp`、`be/src/storage/delete/delete_handler.cpp`；聚合/去重读的上层在 `be/src/storage/` 下的 reader 类（现场找：`tablet_reader`/`block_reader` 等）
  - MoW 内核：`be/src/storage/tablet/base_tablet.h`（`commit_phase_update_delete_bitmap:212`、`update_delete_bitmap:260`、`update_delete_bitmap_without_lock:281`）、`be/src/storage/delete/delete_bitmap_calculator.cpp`、`be/src/storage/tablet_meta.h`（`DeleteBitmap` 类）；云侧 `fe/fe-core/src/main/java/org/apache/doris/cloud/transaction/DeleteBitmapUpdateLockContext.java`
  - 数据管理：`be/src/storage/schema_change/schema_change.cpp/h`、`be/src/storage/delete/delete_handler.cpp`、`fe/fe-core/src/main/java/org/apache/doris/clone/DynamicPartitionScheduler.java`、冷热分层 `be/src/storage/compaction/cold_data_compaction.cpp`
  - 分离模式存储：`be/src/io/cache/`（part2 ch7 已核实 `block_file_cache.h` 四队列/`file_block.h` 状态机——ch6 深入内核实现与对象存储交互）、`be/src/io/fs/`（`s3_file_reader.cpp`、`s3_file_writer.cpp` 现场核实）、`be/src/cloud/`（`cloud_tablet.cpp` 的 `sync_rowsets`）、回收链路引用 part4 第 4 章 Recycler 不重复
- 构建/测试命令与根 `AGENTS.md` 一致。章内基线注格式：「基于写作时核实所用的 HEAD（`XXXX`，源码树与系列基线 `7bc98f696f` 一致）」。
- 每完成一个文件即提交，前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行（见任务内命令）。
- 每个写作任务完成后执行统一**引用校验步骤**（含 `.g4` 后缀），预期输出为空：

```bash
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md|g4)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

---

### Task 1: 第 1 章《Rowset 与 Segment 文件格式》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/01-segment-format.md`

**Interfaces:**
- Consumes: part1 第 3 章层级与磁盘路径、part3 第 3 章 flush 产出 segment 的时机。
- Produces: 页/编码/压缩/Footer 的文件解剖，ch2 索引与 ch3 读路径的地基。

- [ ] **Step 1: 核实素材**

```bash
grep -n "message SegmentFooterPB\|message ColumnMetaPB\|message PagePointerPB\|message DataPageFooterPB" gensrc/proto/segment_v2.proto
grep -n "enum EncodingTypePB\|enum CompressionTypePB" gensrc/proto/segment_v2.proto
grep -n "class SegmentWriter\|append_block\|finalize" be/src/storage/segment/segment_writer.h | head -6
grep -n "class ColumnWriter" be/src/storage/segment/column_writer.h | head -2
ls be/src/storage/segment/ | grep -i page | head -20
grep -n "class BinaryDictPage\|DICT_ENCODING" be/src/storage/segment/binary_dict_page.h | head -3
grep -rn "class PageIO\|compress" be/src/storage/segment/page_io.h 2>/dev/null | head -3; find be/src/storage -name "page_io*" | head -2
grep -rn "DEFAULT_PAGE_SIZE\|page_size" be/src/common/config.cpp | grep -i page | head -3
grep -n "class Segment\b" be/src/storage/segment/segment.h | head -2
```

- [ ] **Step 2: 写作**

创建 `01-segment-format.md`（约 6000-8000 字），结构：

```markdown
# 第 1 章：Rowset 与 Segment 文件格式 —— 一个字节的物理归宿

## 1.1 问题：列存文件怎么摆才能又小又快
（三连问：行存直写（分析扫描读放大）vs 纯列拼接（点查/局部读要全列扫）vs
 分页列存+页级索引（Parquet/ORC 一脉）；页大小的权衡（太小=元数据爆炸、
 太大=读放大）；编码与压缩分两层做的理由（编码利用类型局部性、
 压缩兜底通用冗余））

## 1.2 源码走读：Segment 文件解剖
（SegmentFooterPB 为纲：Footer→ColumnMetaPB→ordinal index→data pages 的
 自包含结构，为什么 Footer 在尾部（一次写完不回填）；
 mermaid 文件布局图；PagePointerPB 的 offset+size 寻址；
 tricky 点：Footer 损坏=整文件不可读——为什么可接受（副本/对象存储兜底）；
 易错点：segment 文件与 rowset 元数据的一致性——`.dat` 文件存在但
 rowset meta 没有它时是谁的孤儿（呼应 part4 第 4 章 Recycler 正逆校验））

## 1.3 源码走读：页、编码与压缩
（SegmentWriter→ColumnWriter→page 构建链；编码选择：字典页
 （BinaryDictPage，字典满退化 plain 的机制——核实真实退化逻辑）、
 bitshuffle（数值列）、prefix（前缀去重）；EncodingTypePB 与列类型的
 默认映射（现场核实映射表所在）；压缩层（CompressionTypePB，
 建表属性 compression）；
 tricky 点：字典页跨 page 共享字典还是 page 内字典（核实！写错会
 误判内存与压缩率）；易错点：低基数字符串列忘开字典/高基数列
 误用字典的性能后果）

## 1.4 双模式对比
（文件格式两模式完全一致（同一个 segment 文件），一句注明；
 差异只在文件放哪（本地盘 vs 对象存储+cache），见 ch6）

## 1.5 动手实验
（核心点：用 meta tool 或 debug 工具解剖一个真实 segment（现场核实
 be 侧是否有 segment dump 工具/`meta_tool`），对照 1.2 布局图逐段看；
 无工具则退化为：小表 flush 后用 `xxd`/proto 解码脚本读 Footer magic；
 易错点：同一列分别用 plain 与 dict 编码建两张表灌同样数据，
 对比 segment 文件大小与查询耗时——把编码选择的后果量化）

## 1.6 排查清单
（症状→路径：segment 损坏报错（checksum/footer）怎么定位副本修复 /
 文件大小异常膨胀（编码退化）/ 版本升级后旧格式兼容问题去哪查）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/01-segment-format.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/01-segment-format.md
git commit -m "[docs] doris-internals part5: ch1 rowset segment file format

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 2: 第 2 章《索引体系：前缀、ZoneMap、BloomFilter 与倒排》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/02-indexes.md`

**Interfaces:**
- Consumes: ch1 的页与 Footer 结构。
- Produces: 四类索引的原理与适用边界，ch3 读路径在裁剪时逐一消费。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class ShortKeyIndexBuilder\|class ShortKeyIndexDecoder" be/src/storage/index/short_key_index.h | head -3
sed -n '1,60p' be/src/storage/index/short_key_index.h   # 头部注释块（若有设计说明）
grep -n "class ZoneMapIndexWriter\|class ZoneMapIndexReader" be/src/storage/index/zone_map/zone_map_index.h | head -3
grep -rn "short_key_max_num\|NUM_SHORT_KEY" be/src/common/config.cpp fe/fe-common/src/main/java/org/apache/doris/common/Config.java 2>/dev/null | head -3
ls be/src/storage/index/bloom_filter/
grep -n "class NGramBloomFilter" be/src/storage/index/bloom_filter/ngram_bloom_filter.h | head -2
ls be/src/storage/index/inverted/ | head -15
grep -rn "class InvertedIndexReader\|class InvertedIndexWriter" be/src/storage/index/inverted/*.h | head -4
grep -rn "bloom_filter_fpp\|bloom_filter_columns" fe/fe-core/src/main/java/org/apache/doris/common/util/PropertyAnalyzer.java | head -3
grep -rn "zone map" be/src/storage/segment/segment_iterator.cpp | head -3
```

- [ ] **Step 2: 写作**

创建 `02-indexes.md`（约 6000-8000 字），结构：

```markdown
# 第 2 章：索引体系 —— 四把裁剪的刀

## 2.1 问题：不读数据就排除数据
（三连问：全扫+快速过滤（向量化再快也要 IO）vs 全局二级索引
 （写放大+分布式一致性）vs 轻量局部索引组合拳（排序键内建+统计摘要+
 概率过滤+可选倒排）；四类索引各自回答什么问题：
 前缀索引=「排序键定位到行」、ZoneMap=「这页/段有没有可能有」、
 BloomFilter=「这个值几乎肯定没有」、倒排=「哪些行含这个词/值」）

## 2.2 源码走读：前缀索引与 ZoneMap
（short key 的构成规则（36 字节/前 3 列类限制——现场核实真实规则与
 常量）、稀疏索引每 1024 行一项（核实粒度常量）；
 ZoneMap 的 segment 级+页级两层、类型支持边界；
 tricky 点：前缀索引只对排序键前缀有效——`WHERE k2=?`（k1 未定）
 为什么用不上，建表列序的实战意义；
 易错点：ZoneMap 对已删除行的语义（delete 谓词与 zonemap 的交互，
 核实 delete_handler 与 zonemap 的关系）、字符串截断语义）

## 2.3 源码走读：BloomFilter 与 NGram
（块级 bloom filter 的 fpp/空间权衡（bloom_filter_fpp 属性）；
 NGram BF 对 LIKE 的加速原理；
 易错点：BF 只支持等值/IN——对范围查询零收益还占空间；
 高基数才值得建 BF 的判断标准）

## 2.4 源码走读：倒排索引（概览+边界）
（倒排目录结构（与 segment 并存的 idx 文件——核实文件组织）、
 analyzer/char_filter 的分词管线、inverted_index_cache；
 定位：本章讲结构与适用性，全文检索语法细节不展开；
 tricky 点：倒排与 BF/ZoneMap 的选择边界——什么查询形态该建哪个；
 易错点：倒排索引的写放大与 compaction 联动成本）

## 2.5 双模式对比
（索引均内嵌于 segment/伴生文件，两模式一致，一句注明；
 倒排 idx 文件在分离模式同样经对象存储+cache，见 ch6）

## 2.6 动手实验
（核心点：同一张表构造四类查询（前缀点查/范围/等值 IN/LIKE），
 用 profile 的裁剪计数器（RowsKeyRangeFiltered/RowsZoneMapFiltered/
 RowsBloomFilterFiltered——与 part2 第 9 章指纹表对齐，核实计数器名）
 观察各索引生效与否；
 易错点：把等值列建成 NGram BF/把范围查询寄望于 BF，
 观察计数器为 0 的"无效索引"现象；调整列序让前缀索引失效再恢复）

## 2.7 排查清单
（症状→路径：索引没生效（列序/类型/谓词形态三查）/ 建倒排后写入
 变慢 / BF 占空间但命中率低的评估方法）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/02-indexes.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/02-indexes.md
git commit -m "[docs] doris-internals part5: ch2 index families

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 3: 第 3 章《读路径内核：谓词下推、延迟物化与合并读》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/03-read-path.md`

**Interfaces:**
- Consumes: ch1 页结构、ch2 索引；part2 第 7 章的 scanner 调度层（衔接点）。
- Produces: segment 内读取全流程，ch4 的 bitmap 读侧消费、part6 慢查询定位引用。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class SegmentIterator" be/src/storage/segment/segment_iterator.h | head -2
grep -n "_next_batch\|next_batch" be/src/storage/segment/segment_iterator.cpp | head -5
grep -rn "late materializ\|lazy materializ\|_lazy_materialization" be/src/storage/segment/segment_iterator.cpp | head -5
grep -n "vec_condition\|short_circuit\|pre_eval" be/src/storage/segment/segment_iterator.h | head -5
find be/src/storage -name "tablet_reader*" -o -name "block_reader*" -o -name "vertical_block_reader*" | grep -v test | head -5
grep -n "class TabletReader\|class BlockReader" be/src/storage/tablet_reader.h be/src/storage/block_reader.h 2>/dev/null | head -4
grep -n "class DeleteHandler" be/src/storage/delete/delete_handler.h | head -2
grep -rn "AGG_KEYS\|UNIQUE_KEYS" be/src/storage/block_reader.cpp 2>/dev/null | head -3
grep -n "class LazyInitSegmentIterator" be/src/storage/segment/lazy_init_segment_iterator.h | head -2
```

- [ ] **Step 2: 写作**

创建 `03-read-path.md`（约 6000-8000 字），结构：

```markdown
# 第 3 章：读路径内核 —— 谓词下推、延迟物化与合并读

## 3.1 问题：读 100 列中的 3 列、命中 1% 的行
（三连问：全列读回内存再过滤（IO 与内存全浪费）vs 谓词列先读+
 行号回捞其余列（延迟物化——多一次随机读换大幅 IO 节省）vs
 索引先裁剪再物化（组合）；合并读的第二个问题：多版本 rowset
 的聚合/去重语义在读时怎么补（AGG 聚合、MoR 去重、DUP 直通））

## 3.2 源码走读：SegmentIterator 的一次 next_batch
（从 rowid 范围（ch2 索引裁剪的产物）到列数据：谓词列先读→
 向量化/短路求值（vec/short_circuit 谓词分类——核实真实分类）→
 生成选中行号→延迟物化回捞非谓词列；
 mermaid 流程图；
 tricky 点：延迟物化的收益边界——选择率高时反而多付随机读，
 有没有自适应开关（现场核实）；
 易错点：`_lazy_materialization` 相关的列拆分逻辑读错会怎样）

## 3.3 源码走读：多 rowset 合并读
（TabletReader/BlockReader（核实真实类族）按 keys type 分流：
 DUP 直通、AGG 读时聚合、UNIQUE-MoR 读时去重（与 ch4 MoW 读侧
 对照留钩子）；delete 谓词（DeleteHandler）在合并时的过滤位置；
 tricky 点：读时聚合的内存与性能代价——为什么 part1 第 3 章说
 count(*) 会随 compaction 变化，这里给出机制级答案（回引核实）；
 易错点：漏算 stale rowsets/版本链选择（引用 part1 3.3 版本路径））

## 3.4 双模式对比
（segment 内读取逻辑两模式一致；分离模式的差异在"页从哪来"
 （cache/对象存储），一句指向 ch6；引用 part2 第 7 章 scanner 层衔接）

## 3.5 动手实验
（核心点：构造宽表低选择率查询，profile 对比延迟物化相关计数器
 （核实真实计数器名）与关闭延迟物化（session 变量，核实存在性与
 名称）的差异；
 易错点：AGG 表上高频导入后立刻 count(*)，观察读时聚合的耗时，
 手动触发 compaction 后再测——把"读时补账"的代价亲手量出来）

## 3.6 排查清单
（症状→路径：同 SQL 忽快忽慢（版本数/合并读代价）/ 宽表点查慢
 （物化策略）/ AGG 表查询结果"变化"的解释与验证）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/03-read-path.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/03-read-path.md
git commit -m "[docs] doris-internals part5: ch3 read path internals

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 4: 第 4 章《主键模型内核：Delete Bitmap 的全生命周期》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/04-mow-internals.md`

**Interfaces:**
- Consumes: part3 第 3 章 3.3 的写入面两阶段（回引核实后深化）、ch3 的读路径。
- Produces: MoW 全生命周期，part6 主键表故障与 part7 正确性案例的基础。

- [ ] **Step 1: 核实素材**

```bash
sed -n '200,290p' be/src/storage/tablet/base_tablet.h   # bitmap 相关方法族
grep -n "class DeleteBitmap\b" be/src/storage/tablet_meta.h | head -2
grep -n "agg\|get_agg" be/src/storage/tablet_meta.h | grep -i bitmap | head -5
grep -n "class DeleteBitmapCalculator" be/src/storage/delete/delete_bitmap_calculator.h | head -2
grep -rn "lookup_row_key" be/src/storage/tablet/base_tablet.cpp | head -3
grep -rn "primary key index\|pk_index\|primary_key_index" be/src/storage/segment/segment_writer.h be/src/storage/segment/segment.h 2>/dev/null | head -4
find be/src/storage -name "*primary_key*" | grep -v test | head -4
grep -rn "delete_bitmap" be/src/storage/compaction/compaction.cpp | head -4
grep -rn "sentinel" be/src/storage/compaction/compaction.cpp | head -2
grep -n "getDeleteBitmapUpdateLock" fe/fe-core/src/main/java/org/apache/doris/cloud/transaction/CloudGlobalTransactionMgr.java | head -2
grep -rn "delete_bitmap" cloud/src/meta-service/meta_service_txn.cpp | head -4
```

- [ ] **Step 2: 写作**

创建 `04-mow-internals.md`（约 6000-8000 字），结构：

```markdown
# 第 4 章：主键模型内核 —— Delete Bitmap 的全生命周期

## 4.1 问题：列存上做主键更新的三条路
（三连问：copy-on-write（重写文件，写放大不可接受）vs
 merge-on-read（读时去重，读放大+谓词下推失效）vs
 delete-and-insert（MoW：新写标记旧行删除，读近似 DUP）；
 MoR/MoW 的本质权衡回顾（part1 第 3 章一句回引）与 MoW 的
 新代价：写入时要"找到旧行"——主键索引与 bitmap 计算；
 为什么 bitmap 按 (rowset, segment, version) 组织而不是全局一张）

## 4.2 源码走读：写入面——从 lookup 到两阶段 bitmap
（主键索引（segment 内 pk index，现场核实结构）→ lookup_row_key
 定位旧行 → bitmap 标记；part3 3.3 讲过两阶段（commit 预算/publish
 权威）——本章深化：为什么必须两阶段（可见版本集合在 publish 才
 确定）、DeleteBitmapCalculator 的多路归并细节；
 tricky 点：bitmap 的 agg/合并语义（DeleteBitmap 的聚合方法，
 版本前缀查询）；
 易错点：并发导入相同 key 的 bitmap 竞争——为什么要锁（一体：本地锁；
 分离：MetaService 的 delete_bitmap_update_lock，回引 part3/part4））

## 4.3 源码走读：读取面与 compaction 面
（读时：ch3 的迭代器如何带着 bitmap 过滤（bitmap 下推到 segment
 iterator 的位置，核实）；
 compaction 时：输入 rowset 的 bitmap 怎么迁移到输出（sentinel 标记、
 calc_compaction_output...，回引 part3 第 6 章的伏笔并深化）；
 tricky 点：compaction 期间新导入的 bitmap 落在旧 rowset 上的
 追赶处理（核实真实机制）；
 易错点：bitmap 本身的存储增长——高频 upsert 下 bitmap 膨胀与
 其自身的"compaction"（核实是否有 bitmap 合并/GC））

## 4.4 双模式对比
（bitmap 计算逻辑一致；存放与锁不同：一体=TabletMeta 本地+
 rocksdb（核实）、分离=MetaService/FDB + 分布式锁；
 回引 part3 第 4 章 commit 七步中的 bitmap 步骤）

## 4.5 动手实验
（核心点：MoW 表高频 upsert 同一批 key，用 profile/元数据接口
 观察 bitmap 数量与大小增长（核实观察入口：SHOW TABLETS?
 meta 接口?）；
 易错点：sync_mode 并发 upsert 相同 key，观察锁竞争的表现
 （日志/耗时），对比不同 key 的并发——把"热点主键"的代价量出来）

## 4.6 排查清单
（症状→路径：MoW 表导入越来越慢（bitmap/锁/compaction 三查）/
 主键表查出重复行（bitmap 缺失的严重故障定位）/ 分离模式
 DELETE_BITMAP_LOCK_ERROR 重试风暴（回引 part3 第 2 章））
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/04-mow-internals.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/04-mow-internals.md
git commit -m "[docs] doris-internals part5: ch4 merge-on-write internals

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 5: 第 5 章《数据管理：Delete、Schema Change 与分区生命周期》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/05-data-management.md`

**Interfaces:**
- Consumes: ch1-4 的存储结构认知。
- Produces: 三类数据管理操作的机制，part6 运维篇引用。

- [ ] **Step 1: 核实素材**

```bash
grep -n "class DeleteHandler\|generate_delete_predicate" be/src/storage/delete/delete_handler.h | head -3
grep -rn "DELETE FROM" fe/fe-core/src/main/java/org/apache/doris/load/DeleteJob.java | head -2; grep -n "class DeleteJob" fe/fe-core/src/main/java/org/apache/doris/load/DeleteJob.java
ls be/src/storage/schema_change/
grep -n "class SchemaChangeJob\|LINKED_SCHEMA_CHANGE\|DIRECT_SCHEMA_CHANGE\|SORTED" be/src/storage/schema_change/schema_change.h | head -5
grep -rln "class SchemaChangeJobV2" fe/fe-core/src/main/java/org/apache/doris/alter/ | head -1
grep -rn "light schema change\|lightSchemaChange" fe/fe-core/src/main/java/org/apache/doris/alter/SchemaChangeHandler.java | head -3
grep -n "class DynamicPartitionScheduler" fe/fe-core/src/main/java/org/apache/doris/clone/DynamicPartitionScheduler.java
grep -rn "cooldown" be/src/storage/tablet/tablet.cpp | head -3
grep -rn "class CooldownDelayer\|cold_data" be/src/storage/compaction/cold_data_compaction.h | head -2
```

- [ ] **Step 2: 写作**

创建 `05-data-management.md`（约 6000-8000 字），结构：

```markdown
# 第 5 章：数据管理 —— Delete、Schema Change 与分区生命周期

## 5.1 问题：不可变文件上做可变操作
（三连问贯穿三件事：a. 删除——就地删（违背不可变）vs 标记删
 （delete 谓词 vs delete bitmap 两种标记的分工）；
 b. 改表结构——停写重建 vs 影子表双写转换 vs 元数据级轻量变更
 （light schema change）；c. 分区生命周期——手动管 vs 动态分区
 自动滚动+冷热分层；三者共同的底色：版本机制（part1 3.3））

## 5.2 源码走读：两种删除
（DELETE FROM 的谓词删除：FE DeleteJob→delete 谓词入 rowset meta→
 读时过滤（DeleteHandler，与 ch3 合并读回扣）；为什么大范围
 delete 谓词会拖慢读（每次读都要评估）；
 主键表的 DELETE=写 delete 行（ch4 bitmap 路径回引）；
 tricky 点：delete 谓词与 compaction 的互动（谓词何时被"物化掉"，
 回引 part3 第 6 章 delete 版本边界）；
 易错点：把 DELETE 当高频操作用（谓词堆积）——正确姿势与替代方案）

## 5.3 源码走读：Schema Change 三档
（FE SchemaChangeJobV2 编排：影子索引/双写/数据转换；
 BE schema_change.cpp 的 LINKED（硬链接零拷贝）/DIRECT（逐行转换）/
 SORTED（重排序）三档判定（核实真实判定逻辑）；
 light schema change：纯元数据加减列为何能跳过 BE（列 unique id
 机制——回引 part3 第 2 章 useSchemaLightChange 门槛并核实）；
 tricky 点：SC 期间导入的双写与版本对齐（核实机制）；
 易错点：以为所有 SC 都轻量——哪些 DDL 触发全量重写、代价预估）

## 5.4 源码走读：分区生命周期与冷热分层
（DynamicPartitionScheduler 的滚动创建/回收；
 冷热分层（cooldown）：本地 SSD→对象存储的迁移（cold_data_compaction、
 remote storage policy——核实类与配置）；与存算分离的关系
 （part1 第 4 章 4.2 讲过两者取舍——回引核实）；
 易错点：动态分区参数（start/end/buckets）配错导致的分区风暴/
 数据误删——真实参数语义核实）

## 5.5 双模式对比
（delete/SC 逻辑一致；冷热分层是一体模式专属（分离模式天然分层），
 一句注明并回引 part1 4.2；分离模式 SC 的差异若有（tablet 元数据在
 MS）简要核实交代）

## 5.6 动手实验
（核心点：对同一张表做加列（观察 light SC 秒级完成）与改列类型
 （观察全量转换 job 进度，SHOW ALTER TABLE），对比两者的
 job 形态与耗时；
 易错点：连续发几十条 DELETE FROM 谓词删除后查询变慢，
 手动触发 compaction 观察恢复——把"谓词堆积"踩一遍）

## 5.7 排查清单
（症状→路径：SC 卡住（双写版本对齐/资源）/ DELETE 后查询变慢 /
 动态分区没按预期创建/误删的定位与恢复窗口）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/05-data-management.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/05-data-management.md
git commit -m "[docs] doris-internals part5: ch5 delete schema-change partition lifecycle

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 6: 第 6 章《存算分离存储：File Cache 内核与对象存储交互》

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/06-cloud-storage.md`

**Interfaces:**
- Consumes: part2 第 7 章的 cache 使用面（回引核实后深化内核）、part4 第 4 章 Recycler。
- Produces: 分离模式存储闭环，part6 缓存故障篇基础。

- [ ] **Step 1: 核实素材**

```bash
ls be/src/io/cache/
grep -n "class BlockFileCache\b" be/src/io/cache/block_file_cache.h | head -2
grep -n "_index_queue\|_normal_queue\|_ttl_queue\|_disposable_queue" be/src/io/cache/block_file_cache.h | head -5
grep -rn "class FileBlock\b\|enum class State" be/src/io/cache/file_block.h | head -4
grep -rn "get_or_set" be/src/io/cache/block_file_cache.cpp | head -3
grep -rn "evict" be/src/io/cache/block_file_cache.cpp | head -5
ls be/src/io/fs/ | grep -iE "s3|obj" | head -8
grep -n "class S3FileWriter\|multipart" be/src/io/fs/s3_file_writer.h | head -4
grep -rn "sync_rowsets" be/src/cloud/cloud_tablet.cpp | head -3
grep -rn "warm_up\|warmup" be/src/cloud/ -r --include=*.h -l | head -5
grep -rn "file_cache_path\|enable_file_cache" be/src/common/config.cpp | head -4
```

- [ ] **Step 2: 写作**

创建 `06-cloud-storage.md`（约 6000-8000 字），结构：

```markdown
# 第 6 章：存算分离存储 —— File Cache 内核与对象存储交互

## 6.1 问题：对象存储的物理现实
（三连问：延迟高两个数量级+按请求计费+最终一致的对象接口，
 怎么撑起交互式分析——直读（延迟不可接受）vs 全量本地副本
 （回到存算一体）vs 块级缓存+异步预热+请求合并；
 cache 的三个子问题：粒度（文件/块）、分类（数据/索引/临时）、
 淘汰（谁先走）——part2 第 7 章给过使用面答案，本章从内核实现
 回答"为什么这么设计"）

## 6.2 源码走读：BlockFileCache 内核
（part2 7.3 的四队列（disposable/index/normal/ttl）从实现层深化：
 队列间容量挪用（借还机制，核实）、块状态机（EMPTY/DOWNLOADING/
 DOWNLOADED/SKIP_CACHE，回引核实）与并发下载去重
 （get_or_set_downloader 的 CAS，回引 part2 并深化实现）；
 淘汰的两级水位（回引 part2 7.3 的 90/88 与 88/85 并深化代码）；
 tricky 点：TTL 队列的语义与误用（谁该进 TTL）；
 易错点：cache 目录放错盘（与数据盘/日志盘争 IO）的隐性代价）

## 6.3 源码走读：对象存储交互
（S3FileWriter 的 multipart 上传（分片阈值/并发/失败清理，核实）；
 读侧 S3FileReader 的重试与限流（回引 part2 第 7 章 s3 计数器）；
 请求合并与预取（现场核实是否有 read-ahead/merge 逻辑）；
 tricky 点：multipart 半途失败的孤儿分片——谁清理（对象存储
 lifecycle vs Recycler，核实后落笔，回引 part4 第 4 章）；
 易错点：对象存储限流（429/503）下的雪崩形态与退避参数）

## 6.4 源码走读：版本同步与预热
（CloudTablet 的 sync_rowsets 拉版本（回引 part3 第 4 章）；
 预热：计算组切换/新节点的 warmup 机制（回引 part4 第 5 章
 CloudWarmUpJob 并从 BE 侧深化：warm_up 的执行链路，核实）；
 tricky 点：预热与按需加载的取舍——全量预热的带宽冲击）

## 6.5 双模式对比
（本章即分离模式专章；给一张"数据文件的一生"对照表收束全部分：
 写入落地/读取来源/合并产物/删除回收 在两模式的完整路径对照——
 综合 part3/part4 与本部分前五章，形成 part5 收官）

## 6.6 动手实验
（核心点：有分离环境则清 cache 冷读同一查询三次，观察
 cache 指标爬升（回引 part2 第 7 章计数器）与队列分布
 （cache http 接口，核实路径）；无环境则纸上推演一个 1GB segment
 的冷读请求数与费用估算（按 block size 算）；
 易错点：把 file_cache_path 容量配大于物理盘，或 TTL 设置不当，
 观察/推演淘汰抖动——回引 part2 7.3 的周期性冷查询现象并给出
 内核级解释）

## 6.7 排查清单
（症状→路径：命中率突降（容量/TTL/淘汰风暴三查）/ 对象存储
 费用异常（请求数分解：读/写/合并/回收）/ 429 限流雪崩的
 止血顺序）
```

- [ ] **Step 3: 引用校验**

统一脚本（FILE=docs/doris-internals/part5-storage-engine/06-cloud-storage.md）。预期无 MISSING；人工核对裸类名与回引真实性。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/06-cloud-storage.md
git commit -m "[docs] doris-internals part5: ch6 cloud storage file cache internals

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

### Task 7: 部分目录页 + 系列 README 翻转

**Files:**
- Create: `docs/doris-internals/part5-storage-engine/README.md`
- Modify: `docs/doris-internals/README.md`（第五部分状态翻转+章节改链接，免责声明下移到 part5 之后）
- Modify: `docs/doris-internals/part4-fe-internals/README.md`（下一部分预告改为直链 ../part5-storage-engine/README.md）

**Interfaces:**
- Consumes: Task 1-6 产出的 6 个章节文件。
- Produces: 完整可导航的第五部分。

- [ ] **Step 1: 核实现状**

```bash
ls docs/doris-internals/part5-storage-engine/
grep -n "第五部分" docs/doris-internals/README.md
grep -n "第五部分\|part5" docs/doris-internals/part4-fe-internals/README.md
```

- [ ] **Step 2: 写作与修改**

part5 README（约 500-700 字，仿前部目录页：导语（主线"从一个字节到一次读取"；ch6 分离专章收官）+ 6 行表格（关键收获反映各章真实结论，不得与章内修正矛盾）+ part6 预告）；系列 README 最小化修改；part4 README 预告改直链。

- [ ] **Step 3: 全量校验**

三个文件引用校验 + 全树死链检查（脚本同前，预期无输出）。

- [ ] **Step 4: 提交**

```bash
git add docs/doris-internals/part5-storage-engine/README.md \
        docs/doris-internals/README.md \
        docs/doris-internals/part4-fe-internals/README.md
git commit -m "[docs] doris-internals part5: part index, series index flip to complete

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01RcEr9tj9GmzUJjd6mRd3hj"
```

---

## 收尾

全部任务完成后：最终整分支审查（fable 模型，含台账 triage、回引真实性专项 grep、主线交接检查（前四批均在结尾交接处抓到过缺陷，重点查）、fix-later 清单处置——本批顺带裁决是否执行 part3 linkify 系列维护提交），修复确认后 push，向作者简报第五部分完成情况，继续第六部分计划制定。
