# 编排流程（Orchestration Flow）

> 定义专家之间怎么接力、谁先谁后、中间状态怎么传。基于"人工接力 + 共享状态文件"模型——零基础设施、可审计、贴合 gsd 单命令习惯。
>
> 依赖：[专家调度协议](./expert-protocol.md)（本文档建立在协议之上）

---

## 一、编排模型：人工接力 + 共享状态文件

### 1.1 为什么是这个模型

| 模型 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| **人工接力 + 状态文件（本方案）** | 零基础设施、可审计、可重跑、可替换 | 需人显式启动每步 | 发散节点流水线、质量需把关 |
| 编排器自动串联 | 全自动、省心 | 要维护编排器、失败传递难定位 | 确定性流水线、高吞吐 |
| gsd 命令链触发 | 改动最小 | 灵活度受限、难处理汇聚节点 | 简单线性流程 |

专家处理的是**发散节点**——产出质量必须人把关，上一步错了不能自动传到下一步。所以选人工接力：**人决策"下一步调谁"，专家负责"把活干好"，状态文件负责"把信息传准"**。

### 1.2 一句话描述

> 每个专家跑完，把状态和产物写到约定路径的文件里；人看一眼产出满意，再显式启动下一个专家——下一个专家读状态文件接着干。编排层（gsd）只负责"提示你下一步可以调谁"，不自动执行。

---

## 二、工作目录结构

所有编排状态集中在项目根目录的 `.gsd-expert/` 下，与代码仓库分离（建议加入 `.gitignore`，或单独提交到 `.planning/` 分支）：

```
项目根/
├── .gsd-expert/                       # 编排工作区（运行时生成，非手写）
│   ├── context/
│   │   └── pipeline.json              # 流水线上下文（全局参数，人初始化）
│   ├── state/                         # 各专家输出状态（信号灯）
│   │   ├── node-1.extract.json
│   │   ├── node-2.copywrite.json
│   │   ├── node-3.tts.json
│   │   ├── node-4.digital-human.json
│   │   ├── node-5.remotion.json
│   │   └── node-6.publish.json
│   ├── artifacts/                     # 各专家产物（实际内容）
│   │   ├── extract/                   # 节点1：原文案 + 拆解
│   │   ├── copywrite/                 # 节点2：仿写文案
│   │   │   ├── script.md
│   │   │   └── diff.md
│   │   ├── tts/                       # 节点3：语音
│   │   │   └── audio.mp3
│   │   ├── digital-human/             # 节点4：数字人
│   │   │   └── avatar.mp4
│   │   ├── remotion/                  # 节点5：合成代码
│   │   │   └── src/
│   │   └── publish/                   # 节点6：发布产物
│   └── log/                           # 专家运行日志（可审计）
│       └── 2026-07-02-copywrite.log
│
├── docs/expert-orchestration/         # 框架文档（本目录）
└── ...（项目其他内容）
```

### 2.1 各目录职责

| 路径 | 谁写 | 谁读 | 作用 |
|------|------|------|------|
| `context/pipeline.json` | 人（初始化） | 所有专家 | 流水线全局参数：账号、风格、目标平台 |
| `state/*.json` | 专家（产出） | 编排层 + 下游专家 | 信号灯 + 业务数据索引 |
| `artifacts/<expert>/*` | 专家（产出） | 人 + 下游专家 | 实际产物（文案/音频/代码） |
| `log/*.log` | 专家（运行时） | 人（审计） | 运行过程留痕 |

### 2.2 初始化

进入专家编排模式时，人或编排层先初始化 `context/pipeline.json`：

```json
{
  "$schema": "pipeline-context/v1",
  "pipeline-id": "voiceover-001",
  "pipeline-type": "voiceover",
  "created-at": "2026-07-02T15:00:00Z",
  "account": {
    "name": "某知识账号",
    "persona": "沉稳、客观、节奏平稳",
    "style-guide": "..."
  },
  "source-video": "https://...",        // 原始口播视频（节点1 输入）
  "target-platform": "douyin",
  "target-duration-sec": 60,
  "nodes": [                            // 流水线节点声明
    { "id": "node-1", "type": "tool",    "name": "去水印抓文案" },
    { "id": "node-2", "type": "expert",  "name": "仿写文案", "expert": "copywrite" },
    { "id": "node-3", "type": "tool",    "name": "TTS 语音合成" },
    { "id": "node-4", "type": "tool",    "name": "数字人生成" },
    { "id": "node-5", "type": "expert",  "name": "Remotion 合成", "expert": "remotion" },
    { "id": "node-6", "type": "tool",    "name": "自动发布" }
  ]
}
```

`type: tool` 的节点是确定性工具调用（不需要专家），`type: expert` 的节点走专家调度协议。

---

## 三、状态文件统一 schema

所有 `state/*.json` 遵循同一结构（协议第四章已定义，这里重申关键字段）：

```json
{
  "$schema": "expert-state/v1",
  "expert-id": "copywrite",
  "expert-version": "1.0",
  "node-id": "node-2",
  "status": "completed",
  "started-at": "2026-07-02T15:23:00Z",
  "completed-at": "2026-07-02T15:31:00Z",
  "quality-gate": { "passed": true, "checks": [...] },
  "artifacts": ["artifacts/copywrite/script.md"],
  "payload": { ... },                   // 业务数据（下游消费）
  "handoff-ready": [                    // 可激活的下游
    { "target": "node-5.remotion", "via": "state/node-2.copywrite.json" }
  ]
}
```

### 3.1 status 取值

| status | 含义 | 编排层行为 |
|--------|------|-----------|
| `completed` | 正常完成 + 质量门禁通过 | 允许激活下游 |
| `partial` | 完成但质量门禁未全过 | 提示人决定是否放行 |
| `failed` | 执行失败 | 不激活下游，提示重试或介入 |

### 3.2 payload 的契约性

`payload` 是下游真正消费的业务数据。它的 schema 由各专家在 `output-contract` 里声明，**必须稳定**。例：

- 节点2 仿写的 payload：`{ script, segments[] }` —— segments 给 TTS/数字人用
- 节点5 Remotion 的 payload：`{ component-path, props, duration-sec }` —— 给发布节点用

payload 变结构 = 下游崩。所以 payload schema 版本化（`schema-version`），变更必须 bump。

---

## 四、DAG 依赖与接力机制

### 4.1 流水线即 DAG

每条流水线是一个有向无环图（DAG）。节点间的依赖关系来自两处：

1. `context/pipeline.json` 的 `nodes` 顺序（隐式线性依赖）
2. 各专家 `input-contract.required` 声明的 `source`（显式依赖）

显式优先。汇聚节点（多个上游）必须显式声明所有依赖。

### 4.2 口播流水线的 DAG

```
node-1 (tool)    去水印抓文案
   │
   ▼
node-2 (expert)  仿写文案 ← 需 node-1
   │
   ├──────────────┐
   ▼              ▼
node-3 (tool)   (文案同时给 4 和 5)
   │              │
   ▼              │
node-4 (tool)  数字人 ← 需 node-2 的 segments
   │              │
   └──────┬───────┘
          ▼
node-5 (expert)  Remotion 合成 ← 需 node-2 + node-3 + node-4（汇聚）
          │
          ▼
node-6 (tool)    自动发布 ← 需 node-5
```

### 4.3 接力流程（以 node-2 → node-5 为例）

```
1. node-1 完成 → 产出 state/node-1.extract.json (status: completed)
       │
2. 编排层扫描 state/，发现 node-1 完成
       │  → 读取 node-1 的 handoff-ready，提示：
       │    「node-1 完成，可激活 node-2 (/gsd-bianxiezhuanjia copywrite)」
       │
3. 人确认 → 执行 /gsd-bianxiezhuanjia copywrite
       │  → 专家读取 state/node-1.extract.json + context/pipeline.json
       │  → 执行仿写逻辑
       │  → 跑质量门禁检查清单
       │  → 产出 state/node-2.copywrite.json + artifacts/copywrite/*
       │  → handoff-ready 标记 [node-3, node-4, node-5] 可激活
       │
4. node-3、node-4 是 tool 节点，人直接跑工具，产出各自 state
       │
5. 到 node-5（汇聚）→ 编排层检查 input-contract.required：
       │  state/node-2.copywrite.json ✓
       │  state/node-3.tts.json ✓
       │  state/node-4.digital-human.json ✓
       │  三者齐备且 completed → 允许激活
       │
6. 人执行 /gsd-bianxiezhuanjia remotion → ... → 产出 state/node-5.remotion.json
       │
7. node-6 发布
```

### 4.4 编排层（gsd）的职责边界

编排层在人工接力模型下**只做三件事**：

1. **扫描**：读 `state/`，判断哪些节点完成、哪些可激活
2. **提示**：根据 `handoff-ready` 和 `input-contract`，告诉人"下一步可调谁"
3. **门禁**：激活专家前检查 `input-contract.required` 是否满足，缺项拒绝

**不做**：自动执行下游、自动重试、自动回滚。这些都由人决策。

---

## 五、中间状态怎么传

### 5.1 传什么

| 传递内容 | 载体 | 示例 |
|---------|------|------|
| 完成信号 | 状态文件存在 + `status` 字段 | `node-2` 完成了吗？看 `state/node-2.copywrite.json` |
| 业务数据 | 状态文件 `payload` | 仿写文案正文、分段结构 |
| 大体积产物 | `artifacts/` 文件 + 状态文件引用 | 音频在 `artifacts/tts/audio.mp3`，状态文件引用路径 |
| 全局参数 | `context/pipeline.json` | 账号风格、目标平台（所有专家共享只读） |

### 5.2 不传什么

- **对话历史 / 运行时变量**：每个专家独立运行，不继承上游上下文
- **执行栈**：失败重试不传递"上次跑到哪"

这样保证：换一个专家实现、重跑某一步，上下游无感。

### 5.3 传递的确定性

下游专家**只依赖状态文件 schema，不依赖上游实现**。即：node-2 换个仿写专家（不同模型/不同 prompt），只要产出的 `state/node-2.copywrite.json` 符合契约，node-5 无需改动。

这是整个框架"可替换"的根基——发散节点最怕锁定某个实现，契约层把实现和接口解耦。

---

## 六、异常处理

### 6.1 前置条件不满足

专家激活时，编排层检查 `input-contract.required`：

```
缺 state/node-3.tts.json
→ 拒绝激活 node-5
→ 提示：「node-5 需要 node-2、node-3、node-4 均完成，当前缺 node-3」
```

### 6.2 专家执行失败

专家写 `status: failed` 的状态文件：

```json
{
  "status": "failed",
  "error": {
    "type": "upstream-quality",
    "message": "node-1 抽取的文案为空",
    "retryable": true
  }
}
```

编排层提示：可重试（修复上游后重跑），或人工介入。

### 6.3 质量门禁未过

`status: partial`，`quality-gate.checks` 列出未通过项：

```json
{
  "status": "partial",
  "quality-gate": {
    "passed": false,
    "checks": [
      { "name": "风格一致性", "passed": false, "note": "语气偏活泼，原账号偏沉稳" }
    ]
  }
}
```

由人决策：
- **放行**：认可当前产出，手动改 `status: completed`，继续下游
- **重跑**：调整参数重新执行专家
- **介入**：人直接改产物文件，再手动写 completed 状态

### 6.4 回退与重跑

任意一步可重跑，不影响其他已完成的节点：

- 重跑 node-2：删 `state/node-2.copywrite.json`（或直接覆盖），重跑后下游 node-5 检测到上游 `completed-at` 更新，提示"上游已更新，建议重跑 node-5"
- 重跑不影响 node-1、node-3、node-4 的状态

> 关键原则：状态文件是唯一的真相源。重跑 = 覆盖状态文件，下游通过对比 `completed-at` 时间戳感知变化。

---

## 七、可观测性

### 7.1 查看流水线状态

人或编排层读 `state/` 即可重建当前进度：

```
node-1.extract     ✓ completed  (15:05)
node-2.copywrite   ✓ completed  (15:31)
node-3.tts         ✓ completed  (15:40)
node-4.digital-human ✗ 缺失
node-5.remotion    (阻塞：缺 node-4)
node-6.publish     (阻塞：缺 node-5)
```

### 7.2 审计

`log/` 下保留每次专家运行的日志（输入快照、检查清单结果、产出摘要）。出了问题可回溯"当时仿写专家拿到了什么、产出了什么、哪一步没过门禁"。

---

## 八、从模型到实践

本文档定义的是**机制**。具体到口播流水线怎么跑，见 [案例验证：口播流水线](./case-voiceover-pipeline.md)；具体到节点2/5 专家怎么写，见 `skills/` 下的两个示例。

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
