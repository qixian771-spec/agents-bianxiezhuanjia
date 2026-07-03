# 仿写文案专家（copywrite）

> 本文档是 `/gsd-bianxiezhuanjia copywrite` 子命令的详细执行流程。由主 SKILL.md 在前置门禁通过后加载。
>
> 协议字段（供编排器识别）：
> - `expert-id`: copywrite
> - `converges`: 文案仿写风格漂移 / 结构标记丢失 / 语气与人设不一致 / 时长偏离
> - `input-contract.required`: state/node-1.extract.json
> - `output-contract.state`: state/node-2.copywrite.json
> - `handoff`: node-3.tts / node-4.digital-human / node-5.remotion（via state/node-2.copywrite.json）
> - `quality-gate`: true

---

## 收敛什么（为什么需要专家而不是直接让大模型仿写）

直接让大模型仿写口播文案，会发散出四类问题——本专家逐项收敛：

| 发散问题 | 不收敛的后果 | 本专家怎么收敛 |
|---------|------------|--------------|
| 风格漂移 | 每次仿出来语气不一样，账号人设崩 | 对照 style-guide 逐项检查，不过门禁重写 |
| 结构标记丢失 | 下游 TTS 不知道在哪断句、数字人不知道口型节奏 | 强制每段输出 emotion + duration-est |
| 语气不一致 | 一篇文案前半沉稳后半煽情 | 全篇语气统一性检查 |
| 时长偏离 | 文案太长视频超时，太短留白尴尬 | 按目标时长校准，段数和字数有约束 |

---

## 仿写流程（5 步）

### 第 1 步：读取输入

```
读 state/node-1.extract.json → 取 payload.original-script + payload.segments
读 context/pipeline.json → 取 account.style-guide + target-duration-sec
```

前置检查：
- `state/node-1.extract.json` 存在且 `status: completed`？否则拒绝
- `original-script` 非空？否则报 `upstream-quality` 错误

### 第 2 步：解构原文案风格

从原文案提取"可仿写的风格指纹"（如果 style-guide 已给，对照验证）：

- **句式特征**：短句为主？长复句？排比？
- **语气词**：有无"啊""呢""家人们"？
- **结构**：hook→points→CTA？结论先行？
- **节奏**：快节奏信息密度高？还是慢节奏留白多？

把风格指纹写入 `artifacts/copywrite/diff.md`，作为仿写依据和后续审计材料。

### 第 3 步：仿写（带约束）

仿写时强制以下约束（不是建议，是硬约束）：

1. **结构对齐**：原文有几个 segment，仿写就产出几个，role 一一对应（hook→hook，point→point）
2. **每段必带标记**：
   ```json
   { "id": 1, "text": "...", "emotion": "calm", "duration-est": 3.2 }
   ```
   - `emotion`：calm / serious / warm / energetic（与账号人设一致）
   - `duration-est`：按 4 字/秒估算（中文口播均值），用于时长校准
3. **禁用词清单**：style-guide 里的禁用词一个不能出现
4. **时长约束**：所有段 `duration-est` 之和应在 `target-duration-sec ± 10%` 内

### 第 4 步：质量门禁（BLOCK 级，全部通过才允许产出 completed）

跑以下检查清单，任一未过则 `status: partial`，列出未过项：

- [ ] **风格一致性**：全篇语气与 style-guide / 原账号人设一致，无突变
- [ ] **结构标记完整**：每段都有 emotion + duration-est，无遗漏
- [ ] **禁用词检查**：style-guide 禁用词零出现
- [ ] **时长匹配**：总 duration-est 在 target ± 10% 内
- [ ] **段数对齐**：仿写段数 = 原文段数（结构对应）
- [ ] **无事实性错误**：仿写未引入原文没有的事实/数据

### 第 5 步：产出

写状态文件 `state/node-2.copywrite.json`（schema 见协议第四章）：

```json
{
  "$schema": "expert-state/v1",
  "expert-id": "copywrite",
  "expert-version": "1.0",
  "node-id": "node-2",
  "status": "completed",
  "quality-gate": { "passed": true, "checks": [...] },
  "artifacts": ["artifacts/copywrite/script.md", "artifacts/copywrite/diff.md"],
  "payload": {
    "script": "（仿写后全文）",
    "segments": [
      { "id": 1, "text": "...", "emotion": "calm", "duration-est": 3.2 },
      { "id": 2, "text": "...", "emotion": "calm", "duration-est": 4.5 }
    ]
  },
  "handoff-ready": [
    { "target": "node-3.tts", "via": "state/node-2.copywrite.json" },
    { "target": "node-4.digital-human", "via": "state/node-2.copywrite.json" },
    { "target": "node-5.remotion", "via": "state/node-2.copywrite.json" }
  ]
}
```

产物文件：
- `artifacts/copywrite/script.md`：仿写后全文（人可读，带分段标记）
- `artifacts/copywrite/diff.md`：风格指纹 + 与原文的对比（审计用）

---

## 质量门禁未过时怎么办

| 未过项 | 处理建议 |
|--------|---------|
| 风格不一致 | 加 `--style "更沉稳/更活泼"` 参数重跑，或在 style-guide 补充约束 |
| 时长偏离 | 加 `--target-sec <N>` 参数，重新校准段数/字数 |
| 段数不对 | 检查 node-1 的 segments 是否准确，必要时重跑 node-1 |

人也可直接编辑 `artifacts/copywrite/script.md`，手动改完后把 `state/node-2.copywrite.json` 的 status 改 `completed`（人工放行）。

---

## 与其他节点的衔接

| 方向 | 衔接 |
|------|------|
| 上游 | node-1.extract（提供原文案 + 结构） |
| 下游 | node-3.tts（消费 segments 的 text + emotion） |
| 下游 | node-4.digital-human（消费 segments 做口型对齐） |
| 下游 | node-5.remotion（消费 segments 做字幕源） |
