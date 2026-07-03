# 案例验证：口播流水线

> 用口播智能体流水线验证专家编排框架。这是框架的第一个落地案例——展示协议、编排流程、gsd 衔接点三套机制如何协同工作，把一条 6 节点流水线从"能跑 MVP"调优到"可用落地"。
>
> 依赖：[专家调度协议](./expert-protocol.md)、[编排流程](./orchestration-flow.md)、[gsd 衔接点](./gsd-integration.md)

---

## 一、口播流水线全景

一条完整的口播视频生产流水线，6 个节点：

| 节点 | 名称 | 做什么 | 类型 |
|------|------|--------|------|
| node-1 | 去水印抓文案 | 从原始口播视频提取干净文案 + 结构拆解 | **确定性工具** |
| node-2 | 仿写文案 | 按账号风格仿写，输出可配音的结构化文案 | **发散节点 → 专家** |
| node-3 | TTS 语音合成 | 文案转语音 | **确定性工具** |
| node-4 | 数字人生成 | 语音驱动数字人出镜 | **确定性工具** |
| node-5 | Remotion 合成 | 用 Remotion 把数字人 + 字幕 + 素材合成最终视频 | **发散节点 → 专家** |
| node-6 | 自动发布 | 发布到目标平台 | **确定性工具** |

### 1.1 为什么 node-2、node-5 是发散节点

用协议第一章的判断标准："换个大模型/换次对话，产出会天差地别吗？"

- **node-2 仿写**：会。同样一段原文案，不同次仿写风格可能从"沉稳客观"漂到"活泼煽情"；结构标记可能丢，导致下游 TTS 断句错乱。→ **需要专家收敛**
- **node-5 Remotion**：会。同样是合成视频，Remotion 代码可能从"组件化、可维护"到"一坨面条代码"，改字幕位置要翻 500 行。→ **需要专家收敛**

- **node-1/3/4/6**：不会。去水印就是调 API，TTS 就是调模型，发布就是调平台接口。换次执行结果一致。→ **普通工具调用，不需专家**

### 1.2 这就是框架的价值

没有专家编排层时：gsd 跑完 6 节点是 MVP（能跑通），但 node-2 仿写每次飘、node-5 代码乱，流水线不可重复、不可维护。

加了专家编排层：node-2、node-5 被收敛到可用区间，流水线可重复产出质量稳定的视频，代码可迭代。**这就是从 MVP 到可用应用的跨越。**

---

## 二、流水线 DAG 与节点配置

### 2.1 DAG 结构图

```
node-1 (tool)  去水印抓文案
   │
   ▼
node-2 (expert) 仿写文案  ←── 收敛：风格漂移、结构丢失
   │
   ├──▶ node-3 (tool) TTS
   │        │
   ├──▶ node-4 (tool) 数字人  ←── 消费 node-2 的 segments
   │        │
   └────────┤
            ▼
node-5 (expert) Remotion 合成  ←── 收敛：代码结构混乱（汇聚：需 2+3+4）
            │
            ▼
node-6 (tool) 自动发布
```

### 2.2 初始化 pipeline.json

进入编排模式后，`/gsd-bianxiezhuanjia init` 生成：

```json
{
  "$schema": "pipeline-context/v1",
  "pipeline-id": "voiceover-001",
  "pipeline-type": "voiceover",
  "created-at": "2026-07-02T15:00:00Z",
  "account": {
    "name": "某知识账号",
    "persona": "沉稳、客观、节奏平稳、不带情绪词",
    "style-guide": "短句为主，每段≤15字；结论先行；禁用'家人们''绝绝子'"
  },
  "source-video": "https://example.com/original.mp4",
  "target-platform": "douyin",
  "target-duration-sec": 60,
  "nodes": [
    { "id": "node-1", "type": "tool",   "name": "去水印抓文案" },
    { "id": "node-2", "type": "expert", "name": "仿写文案", "expert": "copywrite" },
    { "id": "node-3", "type": "tool",   "name": "TTS 语音合成" },
    { "id": "node-4", "type": "tool",   "name": "数字人生成" },
    { "id": "node-5", "type": "expert", "name": "Remotion 合成", "expert": "remotion" },
    { "id": "node-6", "type": "tool",   "name": "自动发布" }
  ]
}
```

---

## 三、逐节点走查

### node-1：去水印抓文案（tool）

**谁跑**：人直接调工具（去水印服务 + 文案提取 API）

**输入**：`context/pipeline.json` 的 `source-video`

**产出**：

```json
// state/node-1.extract.json
{
  "$schema": "expert-state/v1",
  "node-id": "node-1",
  "status": "completed",
  "artifacts": ["artifacts/extract/original-script.md"],
  "payload": {
    "original-script": "...",
    "segments": [
      { "id": 1, "text": "今天讲三个时间管理误区", "role": "hook" },
      { "id": 2, "text": "第一个误区是...", "role": "point" }
    ]
  },
  "handoff-ready": [{ "target": "node-2.copywrite", "via": "state/node-1.extract.json" }]
}
```

> tool 节点也写状态文件——格式与专家一致，只是没有 `converges`、`quality-gate`。这样编排层统一处理，不用区分。

---

### node-2：仿写文案（expert）

**谁跑**：`/gsd-bianxiezhuanjia copywrite`

**收敛什么**（专家的核心）：
1. 风格漂移 → 锁定到账号人设（沉稳客观）
2. 结构标记丢失 → 保证每段有 emotion / duration-est，供 TTS/数字人消费
3. 语气不一致 → 对照 style-guide 逐项检查

**输入契约**：

```yaml
input-contract:
  required:
    - source: state/node-1.extract.json
      desc: "原文案 + 结构拆解"
  optional:
    - source: context/pipeline.json
      field: account.style-guide
      desc: "账号风格指南"
```

**产出**：

```json
// state/node-2.copywrite.json
{
  "$schema": "expert-state/v1",
  "expert-id": "copywrite",
  "node-id": "node-2",
  "status": "completed",
  "quality-gate": {
    "passed": true,
    "checks": [
      { "name": "风格一致性", "passed": true, "note": "语气沉稳，无情绪词" },
      { "name": "结构标记完整", "passed": true, "note": "8 段均有 emotion+duration" },
      { "name": "禁用词检查", "passed": true, "note": "未出现'家人们'等" },
      { "name": "时长匹配", "passed": true, "note": "总时长 58s ≤ 60s" }
    ]
  },
  "artifacts": ["artifacts/copywrite/script.md"],
  "payload": {
    "script": "...",
    "segments": [
      { "id": 1, "text": "时间管理的三个误区", "emotion": "calm", "duration-est": 3.1 },
      { "id": 2, "text": "误区一：把忙当高效", "emotion": "calm", "duration-est": 4.5 }
    ]
  },
  "handoff-ready": [
    { "target": "node-3.tts", "via": "state/node-2.copywrite.json" },
    { "target": "node-4.digital-human", "via": "state/node-2.copywrite.json" },
    { "target": "node-5.remotion", "via": "state/node-2.copywrite.json" }
  ]
}
```

> 专家详情见 `skills/gsd-bianxiezhuanjia/references/expert-copywrite.md`

---

### node-3：TTS 语音合成（tool）

**谁跑**：人调 TTS 服务，输入 node-2 的 segments

**产出**：`state/node-3.tts.json` + `artifacts/tts/audio.mp3`

`payload` 含 `audio-path`、`duration-actual`（实际时长，回填给 node-5 校准）。

---

### node-4：数字人生成（tool）

**谁跑**：人调数字人服务，输入 node-3 音频 + node-2 segments（口型对齐用）

**产出**：`state/node-4.digital-human.json` + `artifacts/digital-human/avatar.mp4`

---

### node-5：Remotion 合成（expert，汇聚节点）

**谁跑**：`/gsd-bianxiezhuanjia remotion`

**收敛什么**：
1. 代码结构混乱 → 组件化分层（Scene / Caption / Avatar / BGM 各自独立组件）
2. 可维护性差 → props 驱动、配置外置、字幕用数据驱动非硬编码
3. 与文案不同步 → segments → props 自动映射，改文案不用改代码

**输入契约（汇聚，三个依赖）**：

```yaml
input-contract:
  required:
    - source: state/node-2.copywrite.json
      desc: "仿写文案 + segments（字幕源）"
    - source: state/node-3.tts.json
      desc: "TTS 音频 + 实际时长"
    - source: state/node-4.digital-human.json
      desc: "数字人视频"
```

**前置检查**：编排层激活前确认三者齐备且 `completed`。

**产出**：

```json
// state/node-5.remotion.json
{
  "$schema": "expert-state/v1",
  "expert-id": "remotion",
  "node-id": "node-5",
  "status": "completed",
  "quality-gate": {
    "passed": true,
    "checks": [
      { "name": "组件化分层", "passed": true, "note": "Scene/Caption/Avatar/BGM 独立组件" },
      { "name": "props 驱动", "passed": true, "note": "字幕/时长/素材全走 props" },
      { "name": "可构建", "passed": true, "note": "npm run build 通过" },
      { "name": "时长校准", "passed": true, "note": "视频时长 58s = TTS 时长" }
    ]
  },
  "artifacts": ["artifacts/remotion/src/"],
  "payload": {
    "component-path": "artifacts/remotion/src/Root.tsx",
    "props": { "segments": [...], "avatar-src": "...", "audio-src": "..." },
    "duration-sec": 58,
    "build-command": "npm run build"
  },
  "handoff-ready": [{ "target": "node-6.publish", "via": "state/node-5.remotion.json" }]
}
```

> 专家详情见 `skills/gsd-bianxiezhuanjia/references/expert-remotion.md`

---

### node-6：自动发布（tool）

**谁跑**：人调发布服务，输入 node-5 构建产物

**产出**：`state/node-6.publish.json` + `artifacts/publish/`（发布记录、视频 URL）

`status: completed` → 编排层提示「流水线完成，可用应用就绪，可 /gsd-deploy」。

---

## 四、状态流转总览

完整跑一遍后，`.gsd-expert/` 长这样：

```
.gsd-expert/
├── context/pipeline.json
├── state/
│   ├── node-1.extract.json        ✓ completed
│   ├── node-2.copywrite.json      ✓ completed (quality-gate passed)
│   ├── node-3.tts.json            ✓ completed
│   ├── node-4.digital-human.json  ✓ completed
│   ├── node-5.remotion.json       ✓ completed (quality-gate passed)
│   └── node-6.publish.json        ✓ completed
├── artifacts/
│   ├── extract/original-script.md
│   ├── copywrite/script.md
│   ├── tts/audio.mp3
│   ├── digital-human/avatar.mp4
│   ├── remotion/src/              ← 可构建的 Remotion 项目
│   └── publish/                   ← 发布产物
└── log/
```

`/gsd-bianxiezhuanjia status` 输出：

```
口播流水线 voiceover-001  ✓ 全部完成
  node-1 去水印抓文案      ✓ 15:05
  node-2 仿写文案(专家)    ✓ 15:31  质量门禁 4/4 通过
  node-3 TTS              ✓ 15:40
  node-4 数字人            ✓ 15:52
  node-5 Remotion(专家)    ✓ 16:10  质量门禁 4/4 通过
  node-6 自动发布          ✓ 16:15

可用应用就绪 → 可执行 /gsd-deploy 部署
```

---

## 五、关键场景验证

### 场景 1：node-2 仿写质量不达标，重跑

```
/gsd-bianxiezhuanjia copywrite 跑完，quality-gate 未过：
  ✗ 风格一致性：语气偏活泼，原账号偏沉稳
  → status: partial

人决策：重跑
  → /gsd-bianxiezhuanjia copywrite --style "更沉稳"
  → 覆盖 state/node-2.copywrite.json
  → quality-gate 通过，status: completed
  → node-3/4/5 检测到 node-2 completed-at 更新，编排层提示重跑下游
```

**验证点**：状态文件可覆盖、重跑不影响其他节点、下游能感知上游更新。

### 场景 2：node-5 汇聚，缺 node-4

```
/gsd-bianxiezhuanjia remotion 激活，前置检查：
  state/node-2.copywrite.json  ✓
  state/node-3.tts.json        ✓
  state/node-4.digital-human.json  ✗ 缺失
→ 拒绝激活，提示「node-5 需 node-2/3/4 均完成，当前缺 node-4」
```

**验证点**：汇聚节点的多依赖门禁生效。

### 场景 3：换仿写专家实现

node-2 当前用 `expert-copywrite.md`（基于模型A）。换一个基于模型B 的实现 `expert-copywrite-v2.md`：

- 只要 v2 产出的 `state/node-2.copywrite.json` 符合契约（schema 一致）
- node-3/4/5 **无需任何改动**

**验证点**：契约解耦，专家实现可替换。

### 场景 4：从 MVP 进入，调优完部署

```
/gsd-ship  → MVP 产出（6 节点跑通，但 node-2/5 质量不稳）
/gsd-bianxiezhuanjia → 进入编排模式
  逐节点走查，node-2/5 专家收敛质量
  全部 completed
/gsd-deploy → 部署可用应用
```

**验证点**：gsd 衔接闭环——开发落地 → 专家编排 → 应用落地。

---

## 六、案例验证结论

| 验证项 | 结果 |
|--------|------|
| 协议：专家与 tool 节点统一状态格式 | ✓ tool 节点也写 state/*.json，编排层统一处理 |
| 协议：converges 区分专家与工具 | ✓ node-2/5 有 converges，node-1/3/4/6 无 |
| 协议：输入输出契约刚性 | ✓ node-5 多依赖门禁、payload schema 稳定 |
| 流程：人工接力状态传递 | ✓ 全程文件通信，零基础设施 |
| 流程：汇聚节点 | ✓ node-5 需 2+3+4 齐备 |
| 流程：重跑与回退 | ✓ 覆盖状态文件，下游感知 |
| 衔接：MVP→可用→部署闭环 | ✓ /gsd-ship → /gsd-bianxiezhuanjia → /gsd-deploy |
| 衔接：不改动 gsd 自带命令 | ✓ 新增 /gsd-bianxiezhuanjia 统一入口 |

框架在口播流水线上跑通了。节点2、5 的具体专家实现见 `skills/gsd-bianxiezhuanjia/references/` 下两个专家文档。

---

## 七、可复用到其他流水线

本案例的 6 节点结构是口播场景特化的，但框架机制是通用的。换一条流水线（如：图文流水线、直播切片流水线），只需：

1. 重新定义 `pipeline.json` 的 `nodes`（哪些是 tool、哪些是 expert）
2. 为新的发散节点写专家 skill（遵守协议 frontmatter）
3. 声明 DAG 依赖

编排层、状态流转、gsd 衔接机制**全部复用**。这就是框架的价值——机制做一次，流水线按需配置。

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
