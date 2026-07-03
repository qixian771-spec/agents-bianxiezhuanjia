# gsd 衔接点（gsd Integration）

> 打通专家编排层与 gsd 工作流的衔接：编排层挂在 gsd 哪一步之后、怎么触发、如何闭环。让"开发专家框架"不是孤岛，而是 gsd 工作流的一个原生阶段。
>
> 依赖：[专家调度协议](./expert-protocol.md)、[编排流程](./orchestration-flow.md)

---

## 一、衔接点定位：MVP 之后、应用落地之前

gsd 的现有工作流覆盖"需求 → MVP 代码合并"，application-launch skill 包覆盖"合并 → 部署运维"。中间有一段真空：

```
gsd 工作流（开发落地）              ←真空→           application-launch（应用落地）
/gsd-plan → /gsd-impl → /gsd-ship  →  MVP  →  ???  →  可用应用  →  /gsd-deploy → ...
        gsd 已覆盖                              专家编排层补齐          gsd 已覆盖
```

**专家编排层就挂在这段真空里**——MVP 跑出来之后、达到"可部署的可用应用"之前。

### 1.1 为什么是这里

gsd 的 `/gsd-ship` 产出的是 MVP：功能跑通、PR 合并。但"能跑通"不等于"能用"：

| MVP 状态 | 可用应用状态 |
|---------|------------|
| 仿写文案能出，但风格漂移、结构不稳 | 仿写风格稳定、结构可被下游消费 |
| Remotion 能合成，但代码乱、改不动 | Remotion 代码结构清晰、可迭代 |
| 流水线能跑完一次 | 流水线可重复、产出质量稳定 |

这段"MVP → 可用"的提升，正是专家的用武之地——专家对发散节点做收敛，把"能跑"提升到"能用"。

### 1.2 不是哪里

专家编排层**不替代** gsd 的开发工作流（`/gsd-plan`、`/gsd-impl`、`/gsd-ship` 照常用），也**不替代** application-launch 的部署运维。它是两者之间的一个**可选增强阶段**：

- MVP 质量已经够用？跳过专家编排，直接 `/gsd-deploy`
- MVP 有发散节点需要收敛？进专家编排，调优完再 `/gsd-deploy`

---

## 二、闭环全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        gsd 工作流闭环                             │
│                                                                  │
│  /gsd-plan ──/gsd-impl──▶ /gsd-ship ──▶ MVP                     │
│   (需求/设计)  (编码)       (测试/PR)      │                      │
│                                            │  开发落地段完成        │
│                                            ▼                      │
│                              ┌──── 专家编排层（本次补齐）────┐    │
│                              │  /gsd-bianxiezhuanjia  进入编排 │    │
│                              │     │                          │    │
│                              │     ├─ init 初始化 pipeline    │    │
│                              │     ├─ 节点逐个跑（人工接力）    │    │
│                              │     │   ├─ tool 节点：直接调     │    │
│                              │     │   └─ expert：/gsd-bianxie..│
│                              │     │        copywrite/remotion │    │
│                              │     ├─ 状态在 .gsd-expert/ 流转  │    │
│                              │     └─ 全部 completed           │    │
│                              └──────────────┬───────────────┘    │
│                                            ▼                      │
│                                         可用应用                   │
│                                            │                      │
│                                            ▼                      │
│  /gsd-deploy ──▶ /gsd-healthcheck ──▶ /gsd-observability          │
│   (部署)         (验证)               (监控)     ← application-launch│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

三段衔接：

1. **入口**：`/gsd-ship` 完成（MVP 产出）→ 触发"是否进入专家编排"的提示
2. **中段**：专家编排层运行（人工接力，状态文件流转）
3. **出口**：全部节点 `completed` → 产出可用应用 → 触发 `/gsd-deploy`

---

## 三、触发与命令

### 3.1 进入专家编排模式

两种入口：

**入口 A：/gsd-ship 完成后自动建议**

`/gsd-ship` 合并 PR 后，gsd 检测到项目有 `.gsd-expert/` 或 `docs/expert-orchestration/` 配置，提示：

```
✓ PR 已合并，MVP 产出。
检测到本项目配置了专家编排层（流水线类型：voiceover）。
是否进入专家编排模式，把 MVP 调优为可用应用？
  → 输入 /gsd-bianxiezhuanjia 进入
  → 输入 /gsd-deploy 跳过，直接部署
```

**入口 B：主动调用**

任何时刻输入 `/gsd-bianxiezhuanjia` 进入编排模式。适用于：MVP 已经跑了一阵、现在要回头调优某个节点。

### 3.2 编排模式内的命令

统一入口 `/gsd-bianxiezhuanjia` + 子命令：

| 命令 | 作用 |
|------|------|
| `/gsd-bianxiezhuanjia` | 看流水线进度（默认 = status） |
| `/gsd-bianxiezhuanjia init` | 初始化 `context/pipeline.json`（首次进入或重置） |
| `/gsd-bianxiezhuanjia status` | 查看流水线进度（读 `state/` 重建 DAG 状态） |
| `/gsd-bianxiezhuanjia copywrite` | 激活仿写专家 |
| `/gsd-bianxiezhuanjia remotion` | 激活 Remotion 专家 |
| `/gsd-bianxiezhuanjia exit` | 退出编排模式（不清理状态，可随时再进入） |

### 3.3 与现有 gsd 命令的关系

| gsd 现有命令 | 关系 |
|-------------|------|
| `/gsd-plan` `/gsd-impl` `/gsd-ship` | **上游**，产 MVP。专家编排不干预 |
| `gsd-deploy` 等 application-launch 6 个 | **下游**，专家编排完成后触发。两者无重叠 |
| ECC / AgentShield | **正交**，安全防护贯穿全程，与专家编排并行 |

专家编排层是 gsd 命令族的一个**新分支**（`/gsd-bianxiezhuanjia` 统一入口 + 子命令），不改动任何自带命令。

---

## 四、衔接的物理实现

专家编排层复用 gsd 的 skill 发现机制——一个统一的 `SKILL.md` 放在 skills 目录即可被 gsd 发现，专家详细逻辑放在同目录的 `references/` 下按子命令加载：

```
~/.<runtime>/skills/
├── gsd-deploy/SKILL.md              # application-launch（已有）
├── gsd-release/SKILL.md
├── ...
└── gsd-bianxiezhuanjia/             # 专家编排框架（本次新增，统一入口）
    ├── SKILL.md                     # 主入口：子命令路由 + 编排器逻辑
    └── references/
        ├── expert-copywrite.md      # 仿写专家详细流程
        └── expert-remotion.md       # Remotion 专家详细流程
```

### 4.1 设计：统一入口 + 子命令路由

`gsd-bianxiezhuanjia` 是编排框架的唯一入口 skill，职责：

1. 进入编排模式，`init` 子命令初始化/读取 `.gsd-expert/context/pipeline.json`
2. `status` 扫描 `state/`，重建当前 DAG 状态，显示进度
3. 根据 `handoff-ready` 和 `input-contract`，提示可激活的专家
4. 激活专家（`copywrite`/`remotion` 子命令）前做前置条件检查（门禁）
5. 专家详细流程按子命令加载 `references/expert-*.md`

编排器只做"提示"不做"执行"——每一步仍由人显式启动，保留人工接力的可审计性。

### 4.2 安装

```bash
# 安装专家编排框架（统一入口，一个 skill 包管全部）
cp -r docs/expert-orchestration/skills/gsd-bianxiezhuanjia ~/.<runtime>/skills/
```

安装后，`/gsd-bianxiezhuanjia` 即可被发现和调用，按 Tab 看子命令。

---

## 五、闭环的关键：状态如何回流 gsd

专家编排层的产出（`.gsd-expert/artifacts/`）需要被 gsd 后续阶段（`/gsd-deploy`）消费，形成闭环：

### 5.1 产物回流

专家编排全部完成后，`artifacts/` 下是可用的应用产物：

```
.gsd-expert/artifacts/
├── copywrite/script.md          # 最终文案
├── tts/audio.mp3                # 最终语音
├── digital-human/avatar.mp4     # 最终数字人
├── remotion/src/                # 最终合成代码（可构建的视频项目）
└── publish/                     # 最终发布产物
```

这些产物就是"可用应用"的载体。`/gsd-deploy` 据此部署：

- Remotion 项目 → 构建为视频服务 / CLI
- 发布产物 → 部署到目标平台

### 5.2 状态回流

`state/` 下最后一个节点的 `status: completed` 是"可用应用就绪"的信号，gsd 据此触发 `/gsd-deploy`：

```
/gsd-bianxiezhuanjia status 显示全部 completed
→ gsd 提示：「专家编排完成，可用应用就绪。是否 /gsd-deploy 部署？」
```

### 5.3 回退闭环

如果部署后发现某个专家产出有问题，可回退到专家编排层：

```
/gsd-deploy 后发现问题
→ /gsd-bianxiezhuanjia（回到编排模式）
→ 重跑问题节点（覆盖 state + artifacts）
→ /gsd-deploy 重新部署
```

因为状态全在文件里，回退和重跑都不丢上下文。

---

## 六、一句话总结衔接点

> 专家编排层挂在 `/gsd-ship`（MVP）之后、`/gsd-deploy`（应用落地）之前，通过 `/gsd-bianxiezhuanjia` 统一入口 + 子命令触发，状态在 `.gsd-expert/` 流转，产物回流给 `/gsd-deploy`。它不改动 gsd 任何自带命令，是工作流闭环里补齐"MVP → 可用"的那一段。

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
