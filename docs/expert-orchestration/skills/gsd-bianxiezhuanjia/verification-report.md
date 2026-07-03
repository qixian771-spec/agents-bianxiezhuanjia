# /gsd-bianxiezhuanjia 编排入口 skill 模拟验证报告

> 用 Python 模拟器按 `skills/gsd-bianxiezhuanjia/SKILL.md` 定义的逻辑跑 9 个阶段，验证设计自洽性。
>
> 时间：2026-07-03 ｜ 验证者：Senior Developer

---

## 验证方式

模拟器（`/tmp/gsd-expert-sim/simulator.py`）按 SKILL.md 的 init/status 逻辑实现，模拟 9 个阶段的状态演进，每个阶段跑一次 status 检查输出是否符合设计。

---

## 验证结果总览

| # | 阶段 | 验证点 | 结果 |
|---|------|--------|------|
| 0 | init | 生成目录 + pipeline.json | ✓ 通过 |
| 1 | 空流水线 status | 只 node-1 可激活，其余阻塞 | ✓ 通过 |
| 2 | node-1 完成 | node-2 可激活 | ✓ 通过 |
| 3 | node-2 专家完成 | 质量门禁 2/2 显示，node-3/4 可激活 | ✓ 通过 |
| 4 | node-3 完成、node-4 未跑 | node-5 汇聚门禁阻塞（缺 node-4） | ✓ 通过 |
| 5 | node-4 完成 | node-5 汇聚门禁通过，可激活 | ✓ 通过 |
| 6 | node-5 完成 | node-6 可激活 | ✓ 通过 |
| 7 | 全部完成 | 提示 /gsd-deploy | ✓ 通过 |
| 8 | 重跑 node-2 | 下游标记"可能过期" | ✓ 通过（修复后） |
| 9 | node-2 变 partial | partial 显示 + 下游"可能过期" | ✓ 通过 |

---

## 发现的两个 bug（已修复）

### Bug 1：进度表重复显示

**现象**：未开始节点在进度表显示一行"○ 未开始"，又在下面阻塞列表再列一行"← 阻塞：..."，同一节点出现两次。

**根因**：输出逻辑里"进度表"和"阻塞列表"分开打印，未开始节点两边都进。

**修复**：阻塞原因合并进进度表每行，不再单独列阻塞段。例：

```
node-2 仿写文案(专家)  ○ 未开始  ← 阻塞：缺 node-1.extract
```

**对 SKILL.md 的启示**：SKILL.md 的 status 输出示例没说清楚"阻塞信息怎么呈现"。需补充：阻塞原因直接附在进度表对应行，不单独成段。

### Bug 2：stale 在"全部完成"时不显示

**现象**：阶段8重跑 node-2（completed-at 变晚），所有节点都 completed，"可能过期"提示却没出现。

**根因**：stale 检测代码放在 `if all_done: ... else: ...` 的 else 分支里，全部完成时被跳过。

**修复**：stale 检测独立于 all_done，无论是否全部完成都显示。

**对 SKILL.md 的启示**：SKILL.md 的 status 流程没提"全部完成时也可能有 stale"这个边界。需补充：stale 提示独立于完成状态，全部完成也可能有下游过期。

---

## 验证通过的关键机制

### 1. 线性接力（阶段1-3）

```
node-1 完成 → node-2 可激活 → node-2 完成 → node-3/4 可激活
```

状态文件存在 + status=completed → 下游 input-contract 满足 → 可激活。✓

### 2. 汇聚节点门禁（阶段4-5）

node-5 需 2+3+4 三个上游。阶段4 只有 2+3，node-5 阻塞"缺 node-4"；阶段5 补齐 node-4，node-5 可激活。✓

### 3. 质量门禁显示（阶段3、6）

专家节点（node-2、node-5）completed 时显示"质量门禁 N/M"，tool 节点不显示。✓

### 4. 重跑传播（阶段8）

node-2 重跑后 completed-at=03:00，晚于下游 node-3(01:40)/node-4(01:52)/node-5(02:10)。status 扫描时对比时间戳，标记三个下游"可能过期"。✓

**这是人工接力模式下感知"重跑传播"的关键机制**——不靠这个，重跑上游后下游不知道该重跑，状态文件会对不上。

### 5. partial 状态（阶段9）

node-2 重跑后质量门禁未过→status=partial。status 显示"⚠ partial (门禁未过)"。下游已完成节点标记"可能过期"（基于可能有问题上游的结果）。✓

---

## 需要回补 SKILL.md 的边界情况

模拟暴露了 SKILL.md 没说清楚的两个边界：

1. **阻塞信息呈现**：阻塞原因应合并进进度表行，不单独成段（避免重复）
2. **全部完成 + stale**：全部完成时也可能有下游过期，stale 提示应独立显示

→ 已据此更新 `skills/gsd-bianxiezhuanjia/SKILL.md` 的 status 章节。

---

## 结论

gsd-expert 编排入口 skill 的核心机制（init / 状态扫描 / 可激活判断 / 汇聚门禁 / 重跑传播 / partial 处理）**设计自洽，可落地**。两个输出 bug 已修复并反向修正了 SKILL.md 的边界说明。

框架从"纸面协议"到"可运行编排"验证通过。
