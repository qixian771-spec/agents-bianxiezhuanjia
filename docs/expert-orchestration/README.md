# 开发专家编排框架（Expert Orchestration Framework）

> gsd 工作流之上的专家编排层。补齐"MVP → 可用应用"这一段真空——让专家对业务流水线里的"发散节点"做收敛，把能跑通的 MVP 调优成可重复、可维护、可部署的可用应用。
>
> 维护者：Senior Developer（高级开发工程师）｜ 版本：v1.0 ｜ 更新：2026-07-02

---

## 这套框架解决什么问题

gsd 的工作流（`/gsd-plan` → `/gsd-impl` → `/gsd-ship`）能跑出 MVP，[application-launch](../application-launch/README.md) 的 skill 包能补齐"合并 → 部署运维"。但中间有一段真空：

```
/gsd-ship → MVP → ？？？ → 可用应用 → /gsd-deploy
                  ↑
            这段原来没人管
```

**MVP"能跑通"不等于"能用"**。业务流水线里有一类"发散节点"——大模型处理时天然会跑偏：

- 文案仿写：每次仿出来风格飘、结构丢，下游 TTS 没法用
- Remotion 合成：代码堆成一坨面条，改字幕要翻 500 行

这类节点不约束就会乱来，MVP 阶段跑通一次，第二次就不可重复。**专家编排层就是给这些发散节点套上"专家"——收敛产出质量，让流水线从"能跑一次"变成"可重复产出可用结果"。**

---

## 核心概念：专家 vs 普通 skill

| | 普通 skill（gsd 已有） | 专家（本框架新增） |
|---|---|---|
| 解决什么 | 流程对不对（按规范走步骤） | 产出好不好（收敛发散性） |
| 节点性质 | 确定性（换次执行结果一致） | 发散性（换次执行天差地别） |
| 例子 | `/gsd-ship`、`gsd-deploy` | `/gsd-bianxiezhuanjia copywrite`、`/gsd-bianxiezhuanjia remotion` |
| 判断标准 | — | 换个大模型/换次对话，产出会天差地别吗？会→需要专家 |

> 一句话：普通 skill 管"流程对不对"，专家管"产出好不好、能不能往下传"。

---

## 文档结构

```
docs/expert-orchestration/
├── README.md                          # 本文件（框架入口）
├── expert-protocol.md                 # 专家调度协议：注册/触发/IO契约
├── orchestration-flow.md              # 编排流程：接力/状态传递/DAG
├── gsd-integration.md                 # gsd衔接点：挂在哪/怎么闭环
├── case-voiceover-pipeline.md         # 案例验证：口播流水线6节点走查
└── skills/
    └── gsd-bianxiezhuanjia/            # 统一入口 skill（编写专家框架）
        ├── SKILL.md                    # 主入口：子命令路由 + 编排器逻辑
        ├── verification-report.md      # 模拟验证报告
        └── references/
            ├── expert-copywrite.md     # 仿写专家详细流程
            └── expert-remotion.md      # Remotion 专家详细流程
```

| 文档 | 定位 | 先读这个如果... |
|------|------|----------------|
| **README.md**（本文档） | 框架总览，快速理解全貌 | 第一次接触本框架 |
| **expert-protocol.md** | 契约层：专家怎么注册、怎么触发、IO 长什么样 | 你要写一个新专家 |
| **orchestration-flow.md** | 机制层：专家怎么接力、状态怎么传 | 你要理解流水线怎么跑 |
| **gsd-integration.md** | 衔接层：挂在 gsd 哪、怎么闭环 | 你要把框架接进 gsd |
| **case-voiceover-pipeline.md** | 验证层：口播流水线完整走查 | 你要看框架跑起来什么样 |
| **skills/gsd-bianxiezhuanjia** | 统一入口 skill：`/gsd-bianxiezhuanjia` 一个命令管全部 | 你要运行/查看流水线进度 |

---

## 30 秒理解框架

```
gsd 工作流（开发落地）          专家编排层（本次补齐）              application-launch（应用落地）
/gsd-impl → /gsd-ship → MVP → /gsd-bianxiezhuanjia 专家接力调优 → 可用应用 → /gsd-deploy
                              ↑
                    人工接力 + 共享状态文件
                    .gsd-expert/state/*.json 流转
```

**三句话**：

1. **专家**是给"发散节点"套的约束 skill，复用 gsd SKILL.md 格式 + 协议扩展字段（`converges` / `input-contract` / `output-contract` / `handoff`）
2. **编排**走人工接力：每个专家跑完写状态文件，人启动下一个专家读状态接着干，零基础设施、可审计
3. **衔接**挂在 `/gsd-ship`（MVP）之后、`/gsd-deploy`（应用落地）之前，不改动 gsd 任何现有命令

---

## 快速上手

### 1. 安装专家框架 skill

```bash
cp -r docs/expert-orchestration/skills/gsd-bianxiezhuanjia ~/.<runtime>/skills/
# 重启运行时，/gsd-bianxiezhuanjia 即可被发现（一个命令管全部子操作）
```

### 2. 进入编排模式

```
/gsd-bianxiezhuanjia init     # 初始化 .gsd-expert/context/pipeline.json（声明流水线节点）
/gsd-bianxiezhuanjia          # 查看流水线状态（=status），看哪些节点可激活
```

### 3. 逐节点跑（人工接力）

```
跑 node-1（工具）              → 产出 state/node-1.extract.json
/gsd-bianxiezhuanjia copywrite → 产出 state/node-2.copywrite.json（过质量门禁）
跑 node-3、node-4（工具）
/gsd-bianxiezhuanjia remotion  → 产出 state/node-5.remotion.json（过质量门禁）
跑 node-6（工具）              → 全部 completed
```

### 4. 部署

```
/gsd-bianxiezhuanjia status   # 确认全部 completed
/gsd-deploy           # 部署可用应用
```

---

## 与现有两套文档的关系

本框架是项目工程化体系的**第三块**，与已有两块互补不重叠：

```
┌─────────────────────────────────────────────────────┐
│  docs/engineering-standards/   通用方法论地基         │
│  定义"应该怎样"——编码/协作/架构/质量/传承规范          │
├─────────────────────────────────────────────────────┤
│  docs/application-launch/     应用落地 skill 包       │
│  定义"gsd-ship 之后怎么部署运维"                       │
├─────────────────────────────────────────────────────┤
│  docs/expert-orchestration/   开发专家编排框架（本次）  │
│  定义"MVP → 可用怎么调优"——专家收敛发散节点            │
└─────────────────────────────────────────────────────┘
```

| 文档集 | 解决 | 阶段 |
|--------|------|------|
| engineering-standards | 团队工程化规范不统一 | 全程地基 |
| application-launch | gsd-ship 之后的部署运维真空 | 应用落地 |
| **expert-orchestration**（本框架） | **MVP → 可用的调优真空** | **专家编排** |

三者的衔接：工程化标准是地基 → gsd 按标准开发出 MVP → 专家编排把 MVP 调优成可用 → application-launch 把可用应用部署上线。

---

## 设计原则

1. **gsd 原生兼容**：专家 skill 就是标准 SKILL.md，丢进 skills 目录即被发现，不改动 gsd 任何现有命令
2. **契约刚性**：输入输出契约是硬约束，不是建议——发散节点最容易"这次结构和上次不一样"，契约锁死结构
3. **人工接力优先**：发散节点产出需人把关，编排层只"建议下一步调谁"，不自动执行
4. **状态全文件化**：所有跨专家信息在 `.gsd-expert/` 文件里，可审计、可重跑、可替换
5. **专家可替换**：专家实现（用哪个模型/prompt）与契约解耦，换实现不改上下游

---

## 适用与不适用

### 适用

- 业务流水线含"发散节点"（大模型产出质量不稳的环节）
- MVP 已跑通，但产出质量不可重复、不可维护
- 需要多个专家协作、有明确上下游依赖

### 不适用

- 纯确定性流水线（全工具调用，无大模型发散环节）——直接 gsd 工作流即可
- 单个专家能搞定、无上下游依赖——直接写一个 skill，不需要编排层
- 还在需求探索期、流水线结构未定——先跑通 MVP 再说

---

## 后续可扩展

本框架 v1.1 已包含编排入口 skill，未来可按需扩展：

- **新流水线**：换一条业务流水线（图文、直播切片），只需重写 `pipeline.json` + 新专家 skill，编排机制复用
- **新专家**：发现新的发散节点，按协议写 SKILL.md 即可接入
- **流水线模板**：为 `gsd-expert init` 补充更多预设模板（图文/直播切片），当前仅口播模板
- **自动编排器**：v2.0 可引入编排器自动串联（当前人工接力），届时 `triggers.auto` 升级为可自动执行

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
