# 《Doris 内核透视》第七批交付物实施计划（第七部分：经典 feature/bug 案例集）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成第七部分（案例集）全部 5 章（10 个真实案例）及部分目录页，并将系列 README 的第七部分状态翻转为"已完成"——全系列收官。（实际交付 12 个：ch4/ch5 各扩为 3 案例）

**Architecture:** 纯文档写作项目。案例素材已由挖掘阶段产出并核实（`.superpowers/sdd/part7-case-candidates.md`，23 个候选，全部 sha 经 `git show --stat` 验证）。本部分每章 2 个真实案例，按设计文档第 6 节的案例结构写作：**问题背景→根因分析→修复思路→源码对照→经验教训**；每案例必须与前六部分的机制章节互链（"这个 bug 撞的正是 partX chY 讲过的那面墙"）。案例的源码对照用 `git show <sha>` 的关键 hunk 片段（≤40 行/段），修复前后对照讲解。每任务流程："读候选档案+git show 核实 → 写作 → 校验 → 提交"。

**Tech Stack:** Markdown（GFM）、mermaid、`git show` diff 片段、对本仓库源码与历史提交的引用。

## Global Constraints

沿用前六批全部规约（三连问在本部分变体为案例三问：**踩了什么坑？为什么会踩？怎么修的、为什么这么修？**），并补充案例专项：

- 语言：中文；类名/代码/日志保留英文。目录页 ≤700 字。
- **Commit 真实性**：每案例引用的 sha 必须再次 `git show --stat <sha>` 核实存在；PR 号仅当出现在 commit message 里才引用。禁止凭候选档案转述——写作时必须亲自 `git show` 读原始 diff 与 message。
- **Diff 走读纪律**：源码对照段用 `git show` 的真实 hunk（可截取、须标注省略）；修复前/后语义对照讲清；行号引用格式 `路径:行号`（当前 HEAD）或 `sha:路径`（历史态），二者不得混淆。
- **机制互链**：每案例"根因分析"段必须链接对应机制章节（相对 md 链接+节号），回引前 grep 目标确认内容存在；案例是前六部分的"考试"，不是机制的重述——机制复述超两句即算重复。
- **经验教训段**须提炼可迁移的模式（如"生命周期与回调的所有权约定"），不允许空泛（"要小心并发"之类）。
- 裸类名逐一 grep；**分拆式**（反引号内禁类名.方法名）；BE/FE 配置可变性核实。
- 案例素材源：`.superpowers/sdd/part7-case-candidates.md`（C1-C23 含 sha/根因文件/关联章节/diff 规模）。
- 章内基线注格式照旧；引用校验脚本（含 `.g4`）预期空输出；每章提交前缀 `[docs]`，落款含 Co-Authored-By 与 Claude-Session 行。

```bash
FILE=docs/doris-internals/xxx.md
grep -oE '`[A-Za-z0-9_./-]+\.(java|cpp|h|hpp|proto|sh|py|groovy|md|g4)' "$FILE" \
  | tr -d '`' | sort -u | while read -r p; do
    [ -e "$p" ] || echo "MISSING: $p"
  done
```

---

### Task 1: 第 1 章《优化器为何算错》（案例 C1 + C5）

**Files:**
- Create: `docs/doris-internals/part7-case-studies/01-optimizer-wrong-results.md`

**Interfaces:**
- Consumes: 候选档案 C1（非幂等函数下推）主 + C5（mark null-aware anti join NULL 语义）副；part2 ch3/ch4/ch8 机制章。
- Produces: 优化器正确性案例章；全书"谓词下推是历史 bug 高发区"（part2 ch3 3.3 预言）的实证收口。

- [ ] **Step 1: 核实素材**：读 `.superpowers/sdd/part7-case-candidates.md` C1/C5 条目；`git show <sha>` 亲读两案例完整 diff 与 message；grep part2 ch3 3.3（outer join 下推错误结果预言）、ch8（join 语义）确认互链点。
- [ ] **Step 2: 写作**（约 6000-8000 字）：章首导语（本章两案例分别命中"等价变换的隐含前提"与"NULL 三值逻辑"两类优化器正确性陷阱）；每案例按 问题背景（可复现 SQL/现象）→根因分析（机制章互链）→修复思路（为什么这么修而非别的）→源码对照（git show hunk 走读）→经验教训（可迁移模式）；章末合并 排查启示（遇到疑似优化器错误结果，用 part2 ch3 3.6 的规则 bisect 法+本章两个根因模式先查）。
- [ ] **Step 3: 校验**：引用校验脚本；sha 复核；裸类名/分拆式/回引 grep。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: ch1 optimizer wrong-result cases`（落款照旧）。

---

### Task 2: 第 2 章《主键模型的正确性边界》（案例 C7 + C8）

**Files:**
- Create: `docs/doris-internals/part7-case-studies/02-mow-correctness.md`

**Interfaces:**
- Consumes: C7（partial update × rollup）主 + C8（compaction 失败 delete bitmap KV 泄漏）副；part5 ch4、part3 ch3/ch6。
- Produces: MoW 正确性案例章；part5 ch4"bitmap 全生命周期"的事故实证。

- [ ] **Step 1: 核实素材**：C7/C8 亲读 diff；grep part5 ch4（bitmap 生命周期/compaction 迁移）、part3 ch6（tablet job）确认互链。
- [ ] **Step 2: 写作**（约 6000-8000 字）：结构同 Task 1（两案例分别命中"写路径的功能组合矩阵"与"异常路径的资源清理"两类模式）。
- [ ] **Step 3: 校验**：同上。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: ch2 mow correctness cases`。

---

### Task 3: 第 3 章《内存与性能回退》（案例 C10 + C11）

**Files:**
- Create: `docs/doris-internals/part7-case-studies/03-memory-regressions.md`

**Interfaces:**
- Consumes: C10（PaddedPODArray 容量误判）主 + C11（Debezium 队列仅行数上限）副；part6 ch6、part3 ch6/ch5。
- Produces: 内存回退案例章；part6 ch6 内存模型的事故实证。

- [ ] **Step 1: 核实素材**：C10/C11 亲读 diff；grep part6 ch6（tracker/水位）、part3 ch5（Routine Load）互链点；C11 若与教程机制关联弱（候选档案已注明），在章内诚实定位为"容量上限的单位错配"通用模式并以 Doris 侧视角讲。
- [ ] **Step 2: 写作**（约 5000-7000 字）：结构同 Task 1（模式："容量语义误判"与"背压上限的量纲缺失"）。
- [ ] **Step 3: 校验**：同上。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: ch3 memory regression cases`。

---

### Task 4: 第 4 章《存算分离的一致性暗礁》（案例 C13 + C15 连环）

**Files:**
- Create: `docs/doris-internals/part7-case-studies/04-cloud-consistency.md`

**Interfaces:**
- Consumes: C13（SC 未删本地 rowset 即 add_rowsets）+ C15（SC 后版本图留洞）——同一 schema change 机制的连环 bug，按时间线写成一条教学线；part5 ch6、part5 ch4、part4 ch4、part5 ch5（SC）。
- Produces: 分离一致性案例章；"BE 本地状态是 MS 权威的镜像"这一全书反复出现主题的最强实证。

- [ ] **Step 1: 核实素材**：C13/C15（及档案中的 C14 作背景）亲读 diff 与时间顺序；grep part5 ch5（SC 双写）、part5 ch6（sync_rowsets）、part4 ch4（MS 权威）互链。
- [ ] **Step 2: 写作**（约 6000-8000 字）：本章特殊结构——先给"同一机制连环修"的时间线图（mermaid），再逐案例五段；经验教训聚焦"镜像状态一致性"模式与"修一个 bug 暴露下一个"的工程现实。
- [ ] **Step 3: 校验**：同上。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: ch4 cloud consistency case chain`。

---

### Task 5: 第 5 章《并发竞争的经典形态》（案例 C18 + C19 对照，C22 简评）

**Files:**
- Create: `docs/doris-internals/part7-case-studies/05-concurrency.md`

**Interfaces:**
- Consumes: C18（取消后线程池任务 UAF）与 C19（析构期 vtable 切换）两种 UAF 形态对照为主线；C22（自死锁）作 500 字内简评收尾；part2 ch6（生命周期/Dependency）、part5 ch4、part3 ch3。
- Produces: 并发案例章；全系列最后一个内容章。

- [ ] **Step 1: 核实素材**：C18/C19/C22 亲读 diff（C18/C19 candidates 档案称带崩溃栈——引用真实栈片段）；grep part2 ch6（task 生命周期）、part5 ch4（bitmap 计算线程）互链。
- [ ] **Step 2: 写作**（约 6000-8000 字）：对照结构——两种 UAF 的触发时序图（mermaid sequence）并列，修法对照（所有权收紧 vs 生命周期栅栏），C22 简评展示第三形态（锁自嵌套）；经验教训提炼"异步任务与宿主生命周期"检查清单。
- [ ] **Step 3: 校验**：同上。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: ch5 concurrency cases`。

---

### Task 6: 部分目录页 + 系列 README 收官翻转

**Files:**
- Create: `docs/doris-internals/part7-case-studies/README.md`
- Modify: `docs/doris-internals/README.md`（第七部分翻转"（规划中）"→"（已完成）"并把案例方向列表改为 5 章链接；移除"以下部分的章节规划摘自设计文档"免责声明（已无未成文部分）；若 README 有全系列状态语句一并收官更新）
- Modify: `docs/doris-internals/part6-operations/README.md`（预告改直链 ../part7-case-studies/README.md）
- Modify: `docs/doris-internals/part6-operations/06-memory.md`（其部级收束段中"第七部分（规划中）"的表述与链接更新为直链已完成的 part7 README——先 grep 确认现文）

**Interfaces:**
- Consumes: Task 1-5 产出。
- Produces: 全系列 7/7 完成的收官导航。

- [ ] **Step 1: 核实现状**：`grep -n "第七部分" docs/doris-internals/README.md docs/doris-internals/part6-operations/*.md`。
- [ ] **Step 2: 写作与修改**：part7 README（≤700 字：导语——案例集是前六部分的"考试"、5 章 10 案例按方向分组、每案例都互链机制章；5 行表格；无预告改为全系列结语一句）；系列 README 收官翻转；part6 两处预告/收束更新。
- [ ] **Step 3: 全量校验**：四文件引用校验 + 全树死链检查（预期无输出）。
- [ ] **Step 4: 提交**：`[docs] doris-internals part7: part index, series complete`。

---

## 收尾

全部任务完成后：最终整分支审查（fable，含：案例 sha 全量复核、互链真实性专项、案例结构五段完整性、机制复述重复度、全系列收官状态一致性（README 无残留"规划中"）、台账 triage），修复确认后 push。**然后向作者做全系列收官简报**（7 部分 43 章+7 索引页的完整交付统计、历批审阅质量数据、遗留项清单），请作者审阅。
