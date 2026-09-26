# 第 2 章：主键模型的正确性边界 —— 两类"写路径与清理路径"陷阱

> 本章行号引用基于写作时核实所用的 HEAD（`9d7b437ec3`，源码树与系列基线一致）。文中所有当前态引用写作 `路径:行号`、历史态引用写作 `sha:路径`，二者不混用；每条案例的 commit sha 均以 `git show` 亲自核实，diff 走读取自真实 hunk（截断处标注省略）。跨部分回引均已 grep 目标文件确认内容存在。
>
> **本章沿用第 1 章的"案例五段式"**：每个案例按 `问题背景 → 根因分析 → 修复思路 → 源码对照 → 经验教训` 展开，对应"案例三问"——踩了什么坑、为什么会踩、怎么修的又为什么这么修。机制细节一律回链前六部分，本章的增量在于"用真实事故检验机制"。

[part5 第 4 章](../part5-storage-engine/04-mow-internals.md) 把 delete bitmap 的一生讲完了：写入面用主键索引定位旧行、两阶段计算把删除按 `(rowset, segment, version)` 记进 Roaring bitmap，读取面按读版本聚合做差集，compaction 面负责搬运、追赶与收缩，存算分离下这份 bitmap 的权威副本还搬到了 MetaService/FDB 上。那一章把机制讲对了；本章要问的是另一个问题——**当写路径叠加了一个"看似无关"的功能（建 rollup），或清理路径遇到了一个"看似不会发生"的异常（compaction 提交失败），这套机制会在哪里裂开？**

主键模型的正确性 bug 有个共性：它们几乎从不在"单功能、正常路径"上出现。partial update 自己是对的、rollup 自己是对的、compaction 提交成功时的清理也是对的——出事的永远是**两个各自正确的东西撞在一起**，或者是**一条平时跑不到的异常分支**。本章两个案例分别命中这两类模式：

- **案例一（C7，主案例，BE 写入面）**：命中"**写路径的功能组合矩阵**"。MoW 表建 rollup 后，对一个**不在 rollup 里的列**做 UPDATE（部分列更新），BE 侧 memtable 的列数 `_num_columns` 与实际输入列的偏移数组对不上，先是 core dump，止血后变成**恒失败**。partial update 测过、rollup 测过，但"partial update × rollup"这条组合对角线没测过。
- **案例二（C8，副案例，存算分离清理面）**：命中"**异常路径的资源清理**"。compaction 提交（`commit_tablet_job`）失败时，临时 rowset 被回收了，但挂在它上面的 delete bitmap KV 没有被连带删除，泄漏在 FDB 里，最终触发"找不到 delete bitmap 对应 rowset"的一致性告警。成功路径的清理是完整的，失败路径的清理漏了一块。

两个案例，一个在写入的组合边界、一个在清理的异常边界，但抽象内核相通：**正确性不是"每个功能单独正确"的简单叠加，而要求"功能组合"与"异常分支"也被显式设计过**。读完这一章你要建立的直觉是——拿到任何一段主键表的写入或清理代码，先追问两件事：它对"叠加了别的功能"这种输入表过态吗？它对"上游失败了"这条分支的资源清理，和成功分支是同一等级的完备吗？

---

## 案例一：建 rollup 后 partial update 更新非 rollup 列恒失败（C7）

- **Commit**：`6122098eb0`（`git show` 核实存在），message 标题 `[fix](partial update) fix partial update always failed after create rollup/MV (#58003)`，PR 号 #58003 出现在 message 中，故引用。
- **根因层**：BE 写入面，memtable 列偏移与列数的初始化（历史态 `6122098eb0:be/src/olap/memtable.cpp`，当前该文件已随目录重构迁移到 `be/src/load/memtable/memtable.cpp`）。

### 问题背景

现象很具体，且 commit message 直接给了复现步骤（非构造）。先在一张 MoW 表上建一个 rollup，这个 rollup **故意不包含 `city` 列**：

```sql
ALTER TABLE mow_table ADD ROLLUP rollup1(event_date, event_time, user_id, country, update_time)
```

然后对不在 rollup 里的 `city` 列做 UPDATE：

```sql
UPDATE mow_table SET city = "beijing" WHERE user_id = 2000
```

MoW 表上的 `UPDATE` 会被规划成**部分列更新**（partial update，模式 `UPDATE_FIXED_COLUMNS`）——它只带主键列加上被 SET 的那几列，其余列靠 publish 时回读旧行补全（[part3 第 3 章](../part3-load-lifecycle/03-tablet-write-path.md) §3.3 讲过这步"回读旧行的其余列"是主键表写放大的来源之一）。问题就出在这里：加了 rollup 之后，这条 partial update 在 BE 侧直接 **core dump**。commit message 里点名了前一个止血 PR：`https://github.com/apache/doris/pull/57934 avoid core dump, but still always fail`——#57934 把崩溃堵住了，但更新**永远失败**，本 PR 才是治本。

这条链路里，rollup 的存在为什么会影响一次只碰 base 表列的 UPDATE？因为在 Doris 的逻辑层级里，rollup **不是**一张独立的表，而是 base 表所在 Partition 下的另一个 `MaterializedIndex`（[part1 第 3 章](../part1-architecture/03-data-model.md) §3.2 把这一层的存在理由讲透了：Partition 管"逻辑数据范围"、MaterializedIndex 管"这段范围的某一种物理排布"）。§3.2 有一句关键结论：**"一次导入必须同时写入 base 和所有 rollup 的对应 Tablet，否则 rollup 和 base 数据不一致"**。partial update 走的正是导入链路——所以建了 rollup 后，这次"看起来只更新 base 表一列"的操作，实际要面对"base + rollup 两套 schema"的写入上下文。base 表与 rollup 的列集合不同，一旦 memtable 在建立"输入列 → block 列"的映射时用错了列数口径，就会越界访问。

### 根因分析

根因是 memtable 里一个**派生量的口径不一致**：表示"这个 memtable 有多少列"的 `_num_columns`，和真正描述"输入的每一列落在 block 哪个下标"的偏移数组 `_column_offset`，是从两个不同来源算出来的，而在 partial update × rollup 这个组合下，两者对不上。

先看修复前的初始化顺序（历史态 `6122098eb0:be/src/olap/memtable.cpp`）。memtable 构造函数里，`_num_columns` 被**提前**赋值：默认取 `_tablet_schema->num_columns()`（tablet schema 的总列数）；若是 partial update 的 `UPDATE_FIXED_COLUMNS` 模式，则改成 `partial_update_info->partial_update_input_columns.size()`（部分列更新的输入列数），再按是否含 auto-increment 列 `+1`。赋完值之后，才调 `_init_columns_offset_by_slot_descs(slot_descs, tuple_desc)` 去遍历 `slot_descs`（本次导入真实的 slot 描述符），为每个 slot 在 tuple 里找到下标、push 进 `_column_offset`。

问题在于：**`_num_columns` 用的是"计划层声称的输入列数"，`_column_offset` 用的是"实际传下来的 slot 数"，这两个数在有 rollup 时不再相等**。#57934 的 message 把这层说破了：`Sometimes planner will plan error and cause slot_desc less than _num_columns and cause be core dump`——规划出来的 slot 描述符个数**少于** `_num_columns`。于是后续所有 `for (cid = num_key_columns; cid < _num_columns; ++cid)` 形态的循环（memtable 里访问 block 列、构造聚合函数的地方比比皆是）都会用一个偏大的 `_num_columns` 去索引只有 `_column_offset.size()` 项的 block，越界。groovy 复现脚本顶部的注释记下了崩溃现场：`CHECK failed: index < data.size() in block.h:182`——这正是 block 按列下标取列时的边界断言。

用 message 里那条 UPDATE 具象一下这个分歧。`UPDATE mow_table SET city = "beijing"` 的 partial update，输入列 = 主键三列（`user_id`、`event_date`、`event_time`）+ 被更新的 `city`，`partial_update_input_columns.size()` 据此算出一个列数 N；但由于表带了 rollup，规划这条导入时要同时准备 base 与 rollup 两套写入上下文，最终透传给这个 base 表 memtable 的 `slot_descs` 个数 M 与 N 对不上（M < N）。修复前 `_num_columns` 被定成 N、`_column_offset` 只有 M 项，后续循环 `cid` 从 `num_key_columns` 一路走到 N-1，越过了 `_column_offset` 的第 M 项就撞上 block 的边界断言。**同一次导入，"计划声称几列"与"实际给了几列"两个数字打了架**——这就是崩溃的直接算术来源。

把两个来源摆在一起看，bug 的本质是：**`_num_columns` 是一个派生量，它本应等于 `_column_offset` 的长度（因为二者描述的是同一件事——这个 memtable 到底要处理几列），却被从另一个独立来源（`partial_update_input_columns.size()`）算出来，两个来源在 partial update × rollup 的组合下产生了分歧。** partial update 单独测没问题（无 rollup 时 slot 数与输入列数一致）、rollup 单独测没问题（全列导入时两个口径也一致）——只有"部分列更新 + rollup 且更新非 rollup 列"这条组合路径，才让 planner 传下来的 slot 数与 `partial_update_input_columns` 的口径分了家。这就是典型的"功能组合矩阵的对角线漏测"。

### 修复思路：为什么这么修

止血与治本是两步，值得对照着看，因为它们回答的是不同的问题。

**#57934（止血）** 的做法是：在 `_init_agg_functions` 开头加一道防御——`if (_num_columns > _column_offset.size()) throw`，把"越界访问导致 BE 崩溃"换成"抛异常、这次导入失败"。它回答的是"别让一个坏输入拖垮整个 BE 进程"，是正确的防御性编程，但**没有修正 `_num_columns` 本身的错误**，所以更新恒失败——message 里 "still always fail" 说的就是它。

**#58003（治本，本案例）** 的核心决策是：**让 `_num_columns` 不再有独立来源，而是直接从 `_column_offset` 派生**。既然 `_column_offset` 是遍历真实 slot 建出来的、代表这个 memtable 实际要处理的每一列，那"列数"就应当**等于**这个数组的长度，而不是另算一遍。修复把构造函数里所有对 `_num_columns` 的提前赋值全部删掉，改到 `_init_columns_offset_by_slot_descs` 的**末尾**、在 `_column_offset` 建完之后写一句 `_num_columns = _column_offset.size();`。

为什么这么修最贴合本质？因为 bug 的根不是"某个分支算错了列数"，而是"列数这个量有两个真相源、且会分歧"。逐个去修每个来源的算法（比如"partial update 分支再减去不在 rollup 的列"）是治标——只要还有两个来源，就还会有下一个组合让它们对不上。**把派生量收敛到单一真相源**（single source of truth：`_num_columns` 永远等于 `_column_offset.size()`），才能让"再叠加任何功能"都不破坏这个不变量。#57934 的防御检查也因此从"临时护栏"升级成了"不变量的运行时断言"——在正确实现下它永远不该触发，留着它是为了兜住未来任何重新引入分歧的回归。

### 源码对照

`git show 6122098eb0` 的核心 hunk（历史态 `6122098eb0:be/src/olap/memtable.cpp`，构造函数部分）：

```cpp
     _vec_row_comparator = std::make_shared<RowInBlockComparator>(_tablet_schema);
-    _num_columns = _tablet_schema->num_columns();
     if (partial_update_info != nullptr) {
         _partial_update_mode = partial_update_info->update_mode();
         if (_partial_update_mode == UniqueKeyUpdateModePB::UPDATE_FIXED_COLUMNS) {
-            _num_columns = partial_update_info->partial_update_input_columns.size();
             if (partial_update_info->is_schema_contains_auto_inc_column &&
                 !partial_update_info->is_input_columns_contains_auto_inc_column) {
                 _is_partial_update_and_auto_inc = true;
-                _num_columns += 1;
             }
         }
     }
     _init_columns_offset_by_slot_descs(slot_descs, tuple_desc);
```

同一个 commit 在 `_init_columns_offset_by_slot_descs` 末尾补上派生赋值（历史态 `6122098eb0:be/src/olap/memtable.cpp`）：

```cpp
     if (_is_partial_update_and_auto_inc) {
         _column_offset.emplace_back(_column_offset.size());
     }
+    _num_columns = _column_offset.size();
 }
```

**修复前语义**：`_num_columns` 在构造函数里按"计划声称的输入列数"提前定死，`_column_offset` 随后按"真实 slot 数"建立，二者在 partial update × rollup 下分歧 → 越界 core（#57934 后变为抛异常恒失败）。**修复后语义**：`_num_columns` 取消所有提前赋值，改为在偏移数组建完后取其长度——列数与偏移数组永远一致，越界从根上消失。注意到 auto-increment 列的处理也一并统一了：修复前它在两处各 `+1`（构造函数给 `_num_columns` 加、offset 里给数组 push），修复后只保留 offset 里那次 `emplace_back`，`_num_columns` 自动跟着 `+1`——又一次印证"单一真相源"消除了重复维护。

这段修复后的代码**在当前 HEAD 仍然存在，且文件已迁移**：`be/src/load/memtable/memtable.cpp:100` 正是 `_num_columns = _column_offset.size();`（在 `_init_columns_offset_by_slot_descs` 末尾，函数定义于 `:86`），构造函数里对 `_num_columns` 的提前赋值确已不见（`:71`~`:78` 只剩对 partial update 模式与 auto-inc 标志的判定）。#57934 那道防御检查也演进后保留在 `be/src/load/memtable/memtable.cpp:104`~`:108`（`_init_agg_functions` 开头，`_num_columns > _column_offset.size()` 时抛 `doris::Exception`，措辞由原来的 `std::runtime_error` 改为统一异常类型，语义未变）——它现在扮演的正是"不变量运行时断言"的角色。

本案例带一份**真实回归测试**（非构造），随 commit 一起加入，当前位于 `regression-test/suites/query_p0/update/update_after_create_rollup.groovy`（commit 加入时在 `nereids_p0` 目录下，后经 #61842 `Merge nereids_p0 test cases into query_p0` 并入 `query_p0`）。它精确复刻了故障三步——建含 34 列的 MoW 表、灌 3 行、`ADD ROLLUP rollup1(...)`（故意漏掉 `city`）、`explain` 确认查询命中 `rollup1`，然后跑三组 UPDATE 断言：更新非 rollup 列（`city`）、更新 rollup 内列（`country`）、以及两者混更。基线数据在 `regression-test/data/query_p0/update/update_after_create_rollup.out`。这个用例的价值不在"覆盖了一个 bug"，而在**它把一条此前没人测过的功能组合对角线钉进了回归基线**——脚本顶部的注释 `Root cause: _num_columns set from partial_update_input_columns.size() but actual input has fewer columns` 直接把根因写在了测试里。

### 经验教训

可迁移的模式有三条：

1. **功能矩阵的对角线测试——两个各自正确的功能，组合起来可能踩出未定义行为**。partial update 正确、rollup 正确，但"partial update 更新非 rollup 列"这条组合路径无人走过。任何"特性 A × 特性 B"的笛卡尔积里，对角线（两个特性同时开启且相互作用）都是测试盲区的高发地带。写测试时不能只覆盖单特性主干，要专门枚举"本特性与哪些其他特性共享同一段代码路径"，逐个交叉。
2. **派生量必须收敛到单一真相源，禁止多来源各算一遍**。`_num_columns` 语义上就是 `_column_offset.size()`，却被从 `partial_update_input_columns.size()` 独立算出——两个来源一旦在某个组合下分歧就是 bug。凡是"这个量本可以从另一个量推出来"的场景，就应当直接推、而不是并行维护。并行维护的两份数据，迟早会在某条你没预料的路径上对不上。
3. **止血与治本要分清，防御检查修好后应升级为不变量断言**。#57934 用 `throw` 把 core 换成失败是正确的止血（保护进程），但它不解决错误本身；#58003 修正了派生逻辑才是治本。治本之后，那道防御检查不该删——它从"临时护栏"转正为"`_num_columns == _column_offset.size()` 这一不变量的运行时哨兵"，专门兜住未来任何重新引入分歧的回归。

---

## 案例二：compaction 提交失败导致 delete bitmap KV 泄漏（C8）

- **Commit**：`bf943cc8f3`（`git show` 核实存在），message 标题 `[fix](mow) delete bitmap is not deleted if commit compaction job failed (#56758)`，PR 号 #56758 出现在 message 中。
- **根因层**：存算分离清理面，Recycler 回收临时 rowset 时对 delete bitmap KV 的清理（`cloud/src/recycler/recycler.cpp`），配套一个 BE 侧的失败注入点（`be/src/cloud/cloud_meta_mgr.cpp`）。

### 问题背景

现象是一条一致性检查告警（commit message 原文附带）：

```
[delete bitmap check fails] can't find corresponding rowset for delete bitmap
instance_id=113100978, tablet_id=1759979382398,
rowset_id=0200000000000027bc46eb5855e1afdc75b79046710c329f, version=13, segment_id=0
```

意思是：FDB 里有一条 delete bitmap KV，但它指向的那个 rowset 已经不存在了——delete bitmap 成了没有主人的孤儿。这是主键表元数据不一致的典型信号。message 把根因列成三条：

1. compaction 更新 delete bitmap 时，不写 pending key；
2. compaction job 提交失败时，delete bitmap KV 没被删；
3. 删临时 rowset KV 时，没有连带删 delete bitmap KV。

**这里必须做一次 diff-vs-message 的诚实核对**（本系列纪律：结论取自 diff，不取自 message 叙述）。`git show bf943cc8f3` 的实际改动只有三个文件、41 行，落地的**代码修复只对应第 3 条**：在 Recycler 回收临时 rowset 时补上 delete bitmap KV 的清理。此外 BE 侧加了一个**失败注入点** `DBUG_EXECUTE_IF("CloudMetaMgr::commit_tablet_job.fail", ...)`，那是为了在单测里模拟"第 2 条"——commit 失败——从而验证清理路径，它本身不是修复。message 的第 1 条（不写 pending key）在本次 diff 里**看不到任何对应改动**。也就是说，message 描述的是作者对整个问题域的三点认知，而本 PR 真正动手补的是"临时 rowset 回收漏删 delete bitmap"这一处——读者若据此判断"pending key 已经补写了"，会被 message 误导。这正是"读 commit 要读 diff、message 只是作者当时的心智模型"该自我检验的地方。

要理解这条链路，得先接上机制。存算分离下，compaction 不是 BE 本地自治的：BE 抢到 MetaService 上的 tablet job 分布式锁后合并、把输出写到对象存储，最后走 `commit_tablet_job` 发 `finish_tablet_job` RPC，让 MetaService 在**一个 FDB 事务里**把临时输出 rowset 转正、把被替换的 input rowset 投进回收队列（[part3 第 6 章](../part3-load-lifecycle/06-compaction.md) §6.4 把这套 start/finish + 租约机制讲全了）。而 MoW 的 delete bitmap，其权威副本也在 MetaService/FDB 上——[part5 第 4 章](../part5-storage-engine/04-mow-internals.md) §4.4 指出，分离模式的 bitmap "commit 时随事务一起写入，pending 态用 `meta_pending_delete_bitmap_key` 暂存"。compaction 产出的临时 rowset，会先把它的 delete bitmap 也写进 FDB。**如果 `finish_tablet_job` 这一步失败了**，临时 rowset 转正没成功，它就会作为"导入失败/中止留下的临时 rowset"，日后由 Recycler 的 `recycle_tmp_rowsets` 清理（[part4 第 4 章](../part4-fe-internals/04-metaservice-fdb.md) §4.3 把回收分类清单列全了，其中 `recycle_tmp_rowsets` 专管 `meta "rowset_tmp"` 家族）。而 bug 就在这里：**Recycler 删临时 rowset 时，没把挂在它上面的 delete bitmap KV 一起删**，于是 rowset 走了、bitmap 留下，成了上面那条告警里的孤儿。

### 根因分析

根因是**异常路径上的清理不完整**，且被 delete bitmap 的"两种 key 编码"放大了。

先说"两种编码"。FDB 里的 delete bitmap key 有两套并存的编码：一套是非版本化的 `meta_delete_bitmap_key`（`cloud/src/meta-store/keys.cpp:330`，key 结构含 `{instance, tablet, rowset, version, segment}`），另一套是版本化的 `versioned::meta_delete_bitmap_key`（`cloud/src/meta-store/keys.cpp:808`，在 `namespace versioned` 内）。修复前，`recycle_tmp_rowsets` 在删临时 rowset 时**只删了版本化那套**（已有的 `delete_versioned_delete_bitmap_kvs` 逻辑），漏了非版本化那套——于是即便走到了清理，也只清掉了一半，另一半 KV 照样泄漏。

再说"异常路径"。这是问题的骨架：compaction 的**成功**路径里，`finish_tablet_job` 在同一个 FDB 事务里完成转正与旧 rowset 入回收队列，一切都是原子的、配对的。但**失败**路径——`commit_tablet_job` 返回错误——把系统留在了一个中间态：临时 rowset 及其 delete bitmap 已经写进了 FDB，转正却没发生。这个中间态的收尾，交给了后台的 `recycle_tmp_rowsets`。可这条回收路径当初设计时，心里装的是"回收一个导入失败留下的普通临时 rowset"——普通导入的临时 rowset 未必带 delete bitmap，所以清理逻辑没把"连带删 delete bitmap"作为标配。当 compaction 失败也复用这条回收路径时，它带来的临时 rowset **是带 delete bitmap 的**，而回收逻辑对此没有对称处理，bitmap 就漏了。

一致性 checker 随后把这个泄漏抓了出来。[part4 第 4 章](../part4-fe-internals/04-metaservice-fdb.md) §4.3 讲过，cloud 侧有个独立 Checker 做双向校验，其中 `do_inverted_check` 反向扫对象存储/元数据、查"有数据/标记但没人引用"的泄漏——上面那条 `can't find corresponding rowset for delete bitmap` 正是这类校验的产物：delete bitmap 还在，它 key 里编的那个 rowset 却查无此物。这也印证了 §4.3 的判断：标记—延迟清理这套异步机制，天然有"该删的没删（漏 → 泄漏）"的风险，而 Checker 就是兜这类风险的最后一道网。

一句话收束根因：**成功路径的资源清理是配对、原子、完整的；失败路径复用了一条"为更简单场景设计"的回收逻辑，没有同等完整地清理 compaction 特有的 delete bitmap，且连仅有的清理都漏了非版本化那套编码。**

下面用一张图把两条路径的清理完整度对照出来——这正是"异常路径与成功路径不同级"的形状：

```mermaid
flowchart TB
    A["compaction 写出临时 rowset<br/>+ delete bitmap KV（两套编码）到 FDB"] --> B{commit_tablet_job<br/>(finish_tablet_job)}
    B -->|成功| C["一个 FDB 事务里：<br/>临时 rowset 转正 + 旧 rowset 入回收队列<br/>（配对、原子、完整）"]
    B -->|失败| D["中间态：临时 rowset 及其 bitmap 已写入 FDB，未转正"]
    D --> E["后台 recycle_tmp_rowsets 兜底清理"]
    E --> F1["删 tmp rowset KV ✓"]
    E --> F2["删 versioned delete bitmap KV ✓"]
    E --> F3["修复前：非版本化 delete bitmap KV 未删 ✗<br/>→ 孤儿泄漏 → Checker 告警"]
    E --> F4["修复后：新增 delete_delete_bitmap_kvs<br/>把非版本化 KV 也删净 ✓"]
```

### 修复思路：为什么这么修

修复目标是让**失败路径的清理与成功路径同级完备**。具体动作有两处：

其一，在 `recycle_tmp_rowsets` 里**补一个清理非版本化 delete bitmap KV 的动作**，与已有的"清理版本化 KV"配对——删临时 rowset 时，两套编码的 delete bitmap KV 都要删干净。为什么用范围删除（`txn_remove(start, end)` 覆盖某 rowset 下所有 version/segment）？因为一个 rowset 的 delete bitmap 可能有多个 version、多个 segment 的条目（§4.2 讲过 bitmap 按 `(rowset, segment, version)` 逐条组织），必须按 `{tablet, rowset}` 前缀把整段扫掉，而不是删单条。

其二，加一个失败注入点 `CloudMetaMgr::commit_tablet_job.fail`，让单测能**主动构造"commit 失败"这条平时跑不到的异常分支**，再断言 delete bitmap KV 确实被回收干净。这一步不是修复本身，但它是"异常路径必须可测"这条工程纪律的落地——异常路径之所以长期漏清理，恰恰因为它平时跑不到、没有测试逼它暴露；补上注入点，才让这条路径进入回归网。

为什么不去修 message 里的第 1 条（compaction 更新 bitmap 时写 pending key）？因为那属于"从源头让 bitmap 有 pending 标记、便于回收识别"的更大改造，不在本 PR 范围内——本 PR 选择的是**在回收侧兜底**：无论 bitmap 当初有没有 pending 标记，回收临时 rowset 时都按 `{tablet, rowset}` 把它的 bitmap KV 扫删干净。这是一个更防御、也更局部的修法：不改写入侧的既有约定，只保证清理侧"删 rowset 必删其 bitmap"这条不变量成立。回到 §4.4——pending key（`meta_pending_delete_bitmap_key`）的本意，正是给"已写入但尚未随事务定稿"的 bitmap 打一个可被识别、可被回收的暂存标记；message 的第 1 条想补的就是让 compaction 的 bitmap 也纳入这套 pending 语义。本 PR 没走那条更彻底的路，而是在回收侧把网收严，属于同一目标下的两种打法，本 PR 选了代价更小的一种。

### 源码对照

先看 BE 侧的失败注入点（`git show bf943cc8f3`，`be/src/cloud/cloud_meta_mgr.cpp`）：

```cpp
 Status CloudMetaMgr::commit_tablet_job(const TabletJobInfoPB& job, FinishTabletJobResponse* res) {
     VLOG_DEBUG << "commit_tablet_job: " << job.ShortDebugString();
     TEST_SYNC_POINT_RETURN_WITH_VALUE("CloudMetaMgr::commit_tablet_job", Status::OK(), job, res);
+    DBUG_EXECUTE_IF("CloudMetaMgr::commit_tablet_job.fail", {
+        return Status::InternalError<false>("inject CloudMetaMgr::commit_tablet_job.fail");
+    });
```

再看核心修复——Recycler 侧新增的清理 lambda 与调用点（`git show bf943cc8f3`，`cloud/src/recycler/recycler.cpp`，省略前后上下文）：

```cpp
+    auto delete_delete_bitmap_kvs = [&](int64_t tablet_id, const std::string& rowset_id) {
+        auto delete_bitmap_start =
+                meta_delete_bitmap_key({instance_id_, tablet_id, rowset_id, 0, 0});
+        auto delete_bitmap_end =
+                meta_delete_bitmap_key({instance_id_, tablet_id, rowset_id, INT64_MAX, INT64_MAX});
+        auto ret = txn_remove(txn_kv_.get(), delete_bitmap_start, delete_bitmap_end);
+        if (ret != 0) {
+            LOG(WARNING) << "failed to delete delete bitmap kv, instance_id=" << instance_id_
+                         << ", tablet_id=" << tablet_id << ", rowset_id=" << rowset_id;
+        }
+        return ret;
+    };
```

```cpp
                 if (delete_versioned_delete_bitmap_kvs(rs.tablet_id(), rs.rowset_id_v2()) != 0) {
                     ...
                 }
+                if (delete_delete_bitmap_kvs(rs.tablet_id(), rs.rowset_id_v2()) != 0) {
+                    LOG(WARNING) << "failed to delete delete bitmap kv, rs="
+                                 << rs.ShortDebugString();
+                    return;
+                }
```

**修复前语义**（历史态 `bf943cc8f3` 之前）：回收临时 rowset 时只调 `delete_versioned_delete_bitmap_kvs`，非版本化那套 `meta_delete_bitmap_key` 编码的 KV 无人清理，随 compaction 提交失败而泄漏。**修复后语义**：新增 `delete_delete_bitmap_kvs`，与版本化清理配对，删临时 rowset 时两套 KV 都按 `{tablet, rowset}` 前缀范围删净；任一删除失败就 `return`、不继续删 tmp rowset key，保证"rowset 与其 bitmap 要么一起走、要么都留下重试"，避免删了一半再泄漏。

这段修复后的代码**在当前 HEAD 仍然存在**：`delete_delete_bitmap_kvs` lambda 位于 `cloud/src/recycler/recycler.cpp:6050`（与之配对的 `delete_versioned_delete_bitmap_kvs` 在 `:6036`），调用点在 `recycle_tmp_rowsets`（`:5906`）内部的 `:6100`（版本化）与 `:6105`（非版本化）——两处紧邻、先后调用，正是"两套编码都清"的落地形态。BE 侧的注入点也仍在 `be/src/cloud/cloud_meta_mgr.cpp:1833`（`commit_tablet_job` 定义于 `:1830`）。

配套单测在 `cloud/test/recycler_test.cpp` 的 `recycle_tmp_rowsets` 用例里：它给每个临时 rowset 用 `create_delete_bitmaps` 造好 delete bitmap KV，回收前用 `check_delete_bitmap_keys_size` 断言 KV 存在（v1/v2 两套编码各查一遍，这也是为什么该辅助函数新增了 `version` 参数区分两套 key 编码），回收后再断言两套 KV 的数量都归零。测试对"两种编码都要清"这个修复要点做了精确覆盖。

### 经验教训

可迁移的模式有三条：

1. **错误路径的资源清理必须与成功路径同级设计**。成功路径的清理往往被反复走、自然趋于完整；失败/中止路径平时跑不到，清理逻辑最容易残缺。凡是"上游操作可能失败"的场景，都要显式追问：失败时，我在成功路径上申请/写入的每一份资源（这里是临时 rowset 的 delete bitmap KV），是否都有对称的清理？"成功时配对写入、失败时配对清理"应当是同一份设计，而不是事后补丁。
2. **一份资源有多种表示/编码时，清理点必须逐一覆盖，漏一种就等于没清**。delete bitmap 在 FDB 里有版本化与非版本化两套 key 编码，只清一套等于泄漏另一套。任何"同一逻辑对象有多种物理落地"的系统（多套索引、主副本、新旧格式并存），清理与失效逻辑都要枚举全部表示，用统一入口或成对调用保证不漏。
3. **异常分支必须可注入、可测**。这条路径长期漏清理，根因之一是"commit 失败"平时几乎不发生、没有测试逼它暴露。修复特意加了 `DBUG_EXECUTE_IF` 失败注入点，把"平时跑不到的分支"变成"单测能主动触发的分支"。给关键的失败路径预置注入点，是让异常处理进入回归网、而非只活在代码审查想象里的必要手段。

---

## 章末：主键表元数据不一致的排查启示

本章两个案例，一个让写入**恒失败**（有明确报错，好定位），一个让元数据**悄悄泄漏**（无前台症状，难发现）——恰好覆盖主键表故障"响不响"的两端。把它们和 [part5 第 4 章](../part5-storage-engine/04-mow-internals.md) §4.6 的 MoW 排查清单合起来用：

1. **partial update / UPDATE 报错或恒失败时，先问"这张表有没有 rollup / 同步物化视图"**。案例一的触发条件极窄：MoW 表 + 建了 rollup + partial update 更新非 rollup 列。若最小复现能对上这三点，且 BE 日志出现 `num_columns ... is greater than block columns` 这类列数越界的异常（或旧版本直接 core 在 `be/src/core/block/block.h` 的 `index < data.size()` 断言上），基本可锁定"memtable 列口径不一致"这一族。判断"是否已修"要以 `git show` 的 diff 为准：修复把 `_num_columns` 收敛成 `_column_offset.size()`（当前 `be/src/load/memtable/memtable.cpp:100`），若你的版本里构造函数还在从 `partial_update_input_columns.size()` 提前赋值，就是未收口的老代码。

2. **"主键表查出重复行"或"delete bitmap 找不到对应 rowset"时，分清是"缺失"还是"泄漏"两个方向**。§4.6 排查清单里"主键表查出重复行（严重故障）"那一行讲的是 delete bitmap **缺失**——该标删的旧行没被标删，读时未减掉，先看 BE 日志有无 `"check delete bitmap correctness failed!"`（§4.3 的 sentinel 自检）。而案例二是相反方向的 delete bitmap **泄漏**——bitmap 还在、它指向的 rowset 没了，症状是 cloud 侧 Checker 的 `can't find corresponding rowset for delete bitmap` 告警（[part4 第 4 章](../part4-fe-internals/04-metaservice-fdb.md) §4.3 的 `do_inverted_check` 逆向校验）。两者都属"delete bitmap 与 rowset 对不上"，但一个是少了标记、一个是多了孤儿，处置完全不同：缺失要查两阶段计算/compaction 追赶是否漏算，泄漏要查 Recycler 的清理路径是否在某条失败分支上漏删。

> **一处必须诚实的定位口径修正**：本章的排查启示挂靠 §4.6（part5 ch4 的 MoW 排查清单，含"主键表查出重复行"专项），而**非** [part6 第 4 章](../part6-operations/04-replica-and-cache-issues.md) §4.2——后者 §4.2 是"一体模式副本故障手册"（tablet 健康状态、修复六因），与主键表 delete bitmap 的正确性无关。遇到主键表查出重复行/元数据不一致，应查 part5 §4.6 与 part4 §4.3，不要按章节号误入 part6 的副本手册。

3. **两个案例共享的方法论：主键表 bug 往往藏在"组合"与"失败分支"里，而非主干**。案例一要 partial update 与 rollup 同时在场才触发，案例二要 compaction 提交恰好失败才暴露——用"单功能、正常路径"的样本永远测不出来。这解释了为什么这类 bug 常在生产的真实拓扑（建了 rollup、偶发提交失败）上才现形，也提示写主键表回归用例时，必须刻意构造"特性叠加"（partial update × rollup × auto-inc × schema change 的交叉）与"注入失败"（用 `DBUG_EXECUTE_IF` 让 commit/publish 失败）这两类边界场景。

两个案例、两类陷阱，收束成一句可迁移的判据：**主键模型的正确性不由"每个功能单独正确"保证，而由"功能组合"与"异常清理"是否被显式设计过决定**——排查疑似主键表故障时，先问"是不是叠加了别的特性"，再问"是不是某条失败分支没把该清的清干净"。
