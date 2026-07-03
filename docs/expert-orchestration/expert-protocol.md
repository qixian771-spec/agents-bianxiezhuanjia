# 专家调度协议（Expert Protocol）

> 定义"开发专家"如何注册、如何被触发、输入输出长什么样、如何接力。本协议是整个专家编排框架的契约层——所有专家 skill 必须遵守本协议，编排层才能识别和调度它们。
>
> 维护者：Senior Developer（高级开发工程师）｜ 版本：v1.0 ｜ 更新：2026-07-02

---

## 一、专家是什么：和普通 skill 的本质区别

gsd 已有一套工作流 skill（`/gsd-impl`、`/gsd-ship`、`gsd-deploy` 等），它们是**确定性的工作流步骤**——输入明确、流程固定、输出可预期。这类 skill 解决的是"按规范走流程"的问题。

但业务流水线里有一类节点，大模型处理时**天然会发散**：

| 节点 | 发散表现 | 不收敛的后果 |
|------|---------|------------|
| 文案仿写 | 风格漂移、结构丢失、语气不一致 | 仿出来不像，每次结果天差地别 |
| Remotion 合成 | 组件结构混乱、可维护性差、与文案不同步 | 代码能跑但没法迭代，改一处崩一片 |

这类节点的特点是：**大模型不约束就会乱来**。给它一个仿写任务，它可能写出十种风格；让它写 Remotion，它可能堆出一坨面条代码。

**专家 = 对"发散节点"施加约束的 skill。** 专家的核心价值不是"把流程走对"（那是普通 skill 的活），而是"把发散性收敛到可用区间"——保证产出物在质量、结构、一致性上达标，可被下游消费、可被后续维护。

> 一句话：普通 skill 管"流程对不对"，专家管"产出好不好、能不能往下传"。

**判断一个节点是否需要专家，问自己：这个节点换个大模型/换次对话，产出会天差地别吗？** 会→需要专家；不会（纯工具调用）→普通 skill 或直接调用即可。

---

## 二、专家注册：frontmatter 扩展字段

专家 skill 复用 gsd 的 `SKILL.md` 格式（保证 gsd 原生兼容），在 frontmatter 上**扩展一组协议字段**。编排层通过识别这些字段区分"专家"和"普通 skill"。

### 2.1 完整字段定义

```yaml
---
# ─── gsd 原生字段（与普通 skill 一致，保证兼容） ───
description: "对原文案做风格一致的仿写，输出结构化文案供下游 TTS/数字人消费"  # ≤100 字符
argument-hint: "[--source <原文案路径>] [--style <风格关键词>]"

# ─── 专家协议扩展字段（编排层识别用） ───
expert-id: copywrite                 # 专家唯一标识，小写连字符，与目录名一致
expert-version: "1.0"                # 专家版本，语义化
expert-stage: mvp-to-usable          # 所属阶段：mvp-to-usable（MVP→可用）/ hardening（加固）/ other

converges:                           # 本专家收敛的发散性（核心标识，回答"为什么需要专家"）
  - 文案仿写中的风格漂移
  - 仿写后结构标记丢失，下游 TTS 无法断句
  - 语气与原账号人设不一致

triggers:                            # 触发条件（满足任一即可能被激活）
  auto:                              # 自动激活：上下文/状态匹配时
    - context-keywords: ["仿写", "改写文案", "文案润色"]
    - state-present: "state/node-1.extract.json"   # 上游状态文件存在时
  manual: "/gsd-bianxiezhuanjia copywrite"    # 主动调用命令（统一入口+子命令）

input-contract:                      # 输入契约：本专家运行必须/可选的输入
  required:
    - source: state/node-1.extract.json
      desc: "节点1 抽取的原文案 + 拆解结构"
  optional:
    - source: context/pipeline.json
      field: styleGuide
      desc: "账号风格指南，无则按原文案推断"

output-contract:                     # 输出契约：本专家必须产出什么
  state: state/node-2.copywrite.json # 状态文件路径（编排层据此判断是否完成）
  artifacts:                         # 产物文件
    - "artifacts/copywrite/script.md"      # 仿写后文案（带结构标记）
    - "artifacts/copywrite/diff.md"        # 与原文的风格对比（可选）
  schema-version: "1.0"              # 状态文件 schema 版本

handoff:                             # 接力点：完成后可被哪些下游专家消费
  - target: node-5.remotion          # 下游专家 id（或节点 id）
    via: state/node-2.copywrite.json # 通过哪个状态文件接力
    consumes-field: script           # 下游消费状态文件的哪个字段

quality-gate: true                   # 是否有质量门禁（true 则必须过检查清单才算完成）
---
```

### 2.2 字段说明速查

| 字段 | 必填 | 作用 | 编排层用途 |
|------|------|------|-----------|
| `expert-id` | ✅ | 专家唯一标识 | 调度寻址、状态文件命名 |
| `expert-version` | ✅ | 语义化版本 | 兼容性判断、升级提示 |
| `expert-stage` | ✅ | 所属阶段 | 按阶段分组编排 |
| `converges` | ✅ | 收敛什么发散性 | **专家身份的核心标识**，区分专家与普通 skill |
| `triggers.auto` | ⬜ | 自动激活条件 | 上下文/状态匹配时主动建议 |
| `triggers.manual` | ✅ | 主动调用命令 | 用户显式触发 |
| `input-contract` | ✅ | 输入依赖 | 编排层据此判断前置条件是否满足 |
| `output-contract` | ✅ | 输出承诺 | 编排层据此判断专家是否完成 |
| `handoff` | ⬜ | 接力下游 | 构建 DAG、提示可激活的下游专家 |
| `quality-gate` | ⬜ | 是否有质量门禁 | 控制是否强制过检查清单 |

### 2.3 命名规范

本框架采用**统一入口 + 子命令**模式，所有专家挂在同一个 skill 包下：

- **skill 包名**：`gsd-bianxiezhuanjia`（编写专家框架，统一入口）
- **专家文件**：`references/expert-<id>.md`。例：`references/expert-copywrite.md`、`references/expert-remotion.md`
- **触发命令**：`/gsd-bianxiezhuanjia <子命令>`。例：`/gsd-bianxiezhuanjia copywrite`、`/gsd-bianxiezhuanjia remotion`
- **状态文件**：`state/<node-id>.<expert-id>.json`。例：`state/node-2.copywrite.json`
- **产物目录**：`artifacts/<expert-id>/`。例：`artifacts/copywrite/`

> 新增专家时：在 `references/` 下加 `expert-<id>.md`，在主 SKILL.md 的子命令路由表加一行，不改 skill 包名。

---

## 三、触发机制

专家有两种触发方式，可并存：

### 3.1 主动调用（manual）

用户显式输入命令触发，最可控：

```
/gsd-bianxiezhuanjia copywrite --source artifacts/extract/script.md
```

适用于：用户明确知道当前要调哪个专家、要传什么参数。**推荐作为主路径**——人工接力模式下，每一步都由人显式启动，可审计。

### 3.2 自动激活（auto）

当上下文或状态满足 `triggers.auto` 条件时，编排层**建议**激活专家（不强制执行）：

```
检测到 state/node-1.extract.json 已生成（节点1 完成）
→ 编排层提示：「节点1 已完成，建议激活 /gsd-bianxiezhuanjia copywrite 进行仿写」
→ 用户确认后执行
```

适用于：长流水线中，减少用户记忆负担——编排层主动告诉你"下一步该调谁"。

> **设计原则**：自动激活只做"建议"，不做"自动执行"。原因：专家处理的是发散节点，产出质量需要人把关；自动串联容易把上一步的错误传递到下游。人工接力 = 人决策、专家干活。

---

## 四、输入输出契约

专家的输入输出全部通过**文件**传递（人工接力模式下无内存共享），约定如下：

### 4.1 输入：状态文件 + 上下文文件

| 输入来源 | 路径 | 说明 |
|---------|------|------|
| 上游状态 | `state/<upstream>.json` | 上游专家的 `output-contract.state` 产出 |
| 流水线上下文 | `context/pipeline.json` | 全局参数（账号信息、风格指南、目标平台等） |
| 原始产物 | `artifacts/<upstream>/*` | 上游专家的产物文件（文案、音频、视频素材等） |

专家启动时，编排层（或人）按 `input-contract.required` 检查前置条件是否满足。缺项则拒绝执行，提示哪个上游没完成。

### 4.2 输出：状态文件 + 产物

**状态文件**（`state/<node-id>.<expert-id>.json`）是编排层的"信号灯"——它的存在表示专家已完成。所有状态文件遵循统一 schema：

```json
{
  "$schema": "expert-state/v1",
  "expert-id": "copywrite",
  "expert-version": "1.0",
  "node-id": "node-2",
  "status": "completed",            // completed | failed | partial
  "started-at": "2026-07-02T15:23:00Z",
  "completed-at": "2026-07-02T15:31:00Z",
  "quality-gate": {
    "passed": true,
    "checks": [
      { "name": "风格一致性", "passed": true, "note": "语气与原账号匹配" },
      { "name": "结构标记完整", "passed": true }
    ]
  },
  "artifacts": [
    "artifacts/copywrite/script.md",
    "artifacts/copywrite/diff.md"
  ],
  "payload": {                      // 下游消费的业务数据
    "script": "...",                // 仿写后文案正文
    "segments": [                   // 结构化分段，供 TTS/数字人用
      { "id": 1, "text": "...", "emotion": "calm", "duration-est": 3.2 }
    ]
  },
  "handoff-ready": [                // 告诉编排层：可激活哪些下游
    { "target": "node-5.remotion", "via": "state/node-2.copywrite.json" }
  ]
}
```

**产物文件**（`artifacts/<expert-id>/*`）是专家产出的实际内容（文案、代码、媒体等），给人看、给下游消费。状态文件的 `payload` 里放精简版/索引，完整内容放产物文件。

### 4.3 契约的刚性

`input-contract` 和 `output-contract` 是**契约，不是建议**：

- 专家**必须**产出 `output-contract.state` 声明的状态文件，否则编排层视为未完成
- 状态文件的 `payload` schema **必须**与 `output-contract` 声明一致，下游才能稳定消费
- schema 变更（`schema-version` 升级）需在专家版本号上体现，编排层据此做兼容性提示

> 这条刚性是整个框架可用的前提。发散节点最容易出的问题就是"这次产出的结构和上次不一样"——契约就是为了锁死结构，让下游不用每次猜上游给了什么。

---

## 五、接力点（handoff）

接力点声明"我完成后，谁能接着干、通过什么接"。这是编排层构建 DAG 依赖的依据。

### 5.1 单向接力（线性流水线）

```
node-1.extract → node-2.copywrite → node-5.remotion → node-6.publish
```

每个专家在 `handoff` 里声明下游，编排层据此画出流水线，并在上游完成后提示可激活的下游。

### 5.2 多输入接力（汇聚节点）

某些专家需要多个上游都完成才能启动。在 `input-contract.required` 里声明多个依赖：

```yaml
input-contract:
  required:
    - source: state/node-2.copywrite.json   # 文案
      desc: "仿写后文案 + 分段"
    - source: state/node-3.tts.json         # TTS 音频
      desc: "语音合成产物"
    - source: state/node-4.digital-human.json  # 数字人
      desc: "数字人视频素材"
```

编排层检查：三个状态文件都存在且 `status: completed`，才允许激活本专家。否则提示"还缺 node-3、node-4"。

### 5.3 接力的边界

接力**只传状态和产物路径，不传执行上下文**。即：下游专家不会继承上游专家的对话历史、变量绑定等运行时状态——每个专家是独立的运行单元，只通过文件通信。

这样设计的原因：
1. **可审计**：所有跨专家的信息都在文件里，可检查、可回放
2. **可重跑**：某个专家产出不满意，改完重跑不影响其他专家
3. **可替换**：某个专家换成另一个实现，只要契约一致，上下游无感

---

## 六、专家生命周期

```
注册（写 SKILL.md + 协议字段）
  → 发现（编排层扫描 skills/ 目录，解析 frontmatter）
    → 触发（主动调用 / 自动激活建议）
      → 前置检查（input-contract.required 是否满足）
        → 执行（专家主体逻辑）
          → 质量门禁（quality-gate 为 true 时必须过检查清单）
            → 产出（写状态文件 + 产物）
              → 接力就绪（handoff-ready 通知编排层可激活下游）
```

任一环节失败：
- **前置检查失败** → 提示缺哪个上游，不执行
- **执行失败** → 写 `status: failed` 的状态文件，提示重试或人工介入
- **质量门禁未过** → 写 `status: partial`，列出未通过的检查项，由人决定是否放行

---

## 七、最小可用的专家示例

本框架采用统一入口模式，专家不再独立成 skill，而是作为 `references/expert-<id>.md` 被主入口按子命令加载。一个专家文件最少需要这些（以仿写为例）：

```yaml
# references/expert-copywrite.md 头部（协议字段，供编排器识别）
expert-id: copywrite                                      # 协议：身份
expert-version: "1.0"                                     # 协议：版本
expert-stage: mvp-to-usable                               # 协议：阶段
converges: ["文案仿写中的风格漂移"]                        # 协议：核心标识
triggers:
  manual: "/gsd-bianxiezhuanjia copywrite"                # 协议：触发（统一入口+子命令）
input-contract:
  required:
    - source: state/node-1.extract.json                   # 协议：输入契约
output-contract:
  state: state/node-2.copywrite.json                      # 协议：输出契约
  artifacts: ["artifacts/copywrite/script.md"]
handoff:
  - target: node-5.remotion
    via: state/node-2.copywrite.json
quality-gate: true
```

```markdown
# 仿写文案专家（copywrite）

## 何时使用
...（专家主体，见各 skill 文档）

## 质量门禁检查清单
- [ ] 风格与原账号人设一致
- [ ] 结构标记完整（每段有 emotion / duration-est）
- [ ] 无事实性错误
...
```

完整示例见 `skills/gsd-bianxiezhuanjia/references/expert-copywrite.md` 和 `expert-remotion.md`。

---

## 八、协议演进

本协议是 v1.0。未来扩展方向（不破坏兼容性）：

- **v1.1**：增加 `parallel-with` 字段，支持多个专家并行（当前仅串行 + 汇聚）
- **v1.2**：增加 `retry-policy`，声明失败后的自动重试策略（当前需人工重跑）
- **v2.0**：引入编排器自动串联（当前为人工接力），届时 `triggers.auto` 升级为可自动执行

任何协议变更需同步更新本文档版本号，并在受影响的专家 skill 上 bump `expert-version`。

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
