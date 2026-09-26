# 第七部分：经典 feature/bug 案例集

前六部分把 Doris 的机制讲对了，这一部分是那些机制的**"考试"**：从 git 历史精选 5 类方向、12 个真实事故，每个案例按"问题背景→根因分析→修复思路→源码对照→经验教训"五段展开，对应案例三问——踩了什么坑、为什么会踩、怎么修的又为什么这么修。每个案例的根因分析都**回链**它对应的机制章节，不重述机制，只用真实 commit 检验你是否读懂了它；每条 sha 均以 `git show` 亲自核实，diff 走读取自真实 hunk。

| 章节 | 方向 | 关键收获 |
|---|---|---|
| [第 1 章](01-optimizer-wrong-results.md) | 优化器错误结果 | 非幂等谓词被 `containsAll(emptySet)` 静默放行，一次 PR 给 8 个规则文件补守卫；示范 message 与 diff 冲突时以 diff 为准的走读纪律 |
| [第 2 章](02-mow-correctness.md) | 主键模型正确性 | partial update × rollup 的组合边界，与 compaction 失败路径的 bitmap 泄漏；message 列出的三个根因只有一个真正落进了 diff |
| [第 3 章](03-memory-regressions.md) | 内存与性能回退 | `allocated_bytes()` 含 padding 被误当可用空间、触发 4GB 分配；背压上限缺了字节量纲——容量语义与量纲各误判一次 |
| [第 4 章](04-cloud-consistency.md) | 存算分离一致性 | 同一 cloud schema change 机制四十天内被连环修三次，按真实提交时间序讲；主线是"BE 本地镜像必须逐版本对齐 MS 权威版本图" |
| [第 5 章](05-concurrency.md) | 并发竞争 | 两种 UAF（悬垂 `this` vs 悬垂 vtable，C18 的修复直接催生 C19）并列，加一种锁自嵌套自死锁，收尾给一份对象生命周期检查清单 |

至此，全系列七部分、四十三章走完：从一条 SQL 的请求路径，到元数据与存储引擎的内核，再到运维视角与真实事故的复盘，Doris 这只"黑盒"已被逐层打开。
