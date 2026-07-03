# agents-bianxiezhuanjia

> 把 AI 编码工作流（gsd）从"能跑出 MVP"推进到"可用应用稳定上线"的一套工程化方法论 + 可落地 skill 包。三块产出层层递进：**通用工程化地基 → 应用落地 skill → 开发专家编排框架**。

---

## 这是什么

用 gsd（[opengsd](https://opengsd.net)）这类 AI 编码工作流开发应用时，普遍存在三段真空：

```
/gsd-impl → /gsd-ship → MVP → ？？？ → 可用应用 → ？？？ → 稳定运行
                  ↑              ↑                    ↑
              gsd 已覆盖     调优真空（发散节点没人管）  部署运维真空
```

本仓库补齐右侧两段真空，让 gsd 工作流真正闭环。

---

## 三块产出

| 产出 | 位置 | 解决什么 |
|------|------|---------|
| ① **工程化标准体系** | `docs/engineering-standards/` | 团队规范不统一、架构弱、测试缺失、传承断层 |
| ② **应用落地 Skill 包** | `docs/application-launch/` | gsd-ship 合并 PR 后到稳定运行之间的部署运维真空 |
| ③ **开发专家编排框架** | `docs/expert-orchestration/` | MVP→可用之间的调优真空（发散节点不约束就乱来） |

### 三者关系

```
工程化标准（地基：定义"应该怎样"）
    ↓
gsd 按标准开发出 MVP（/gsd-plan → /gsd-impl → /gsd-ship）
    ↓
专家编排框架（MVP→可用的调优：/gsd-bianxiezhuanjia）
    ↓
应用落地 skill（可用→稳定运行：/gsd-deploy 等）
```

---

## ① 工程化标准体系

`docs/engineering-standards/` — 8 篇文档构成团队"宪法"：

- 五层结构：编码规范 → 协作规范 → 架构设计 → 质量保障 → 技术传承
- 通用原则 + 技术栈专规双层结构，适配多技术栈混合团队
- 工具优先（lint/format 卡住的不靠人肉），每篇配自检清单
- 4 周分阶段落地路线图

详见 [`docs/engineering-standards/README.md`](docs/engineering-standards/README.md)。

---

## ② 应用落地 Skill 包

`docs/application-launch/` — 6 个 gsd 格式 skill，补齐"合并 PR → 稳定运行"：

| Skill | 命令 | 解决 |
|-------|------|------|
| gsd-deploy | `/gsd-deploy` | 部署编排（灰度/蓝绿/回滚） |
| gsd-release | `/gsd-release` | 版本发布（semver/tag/changelog） |
| gsd-healthcheck | `/gsd-healthcheck` | 健康检查（存活/就绪/启动探针） |
| gsd-migration | `/gsd-migration` | 数据库迁移（前向/回滚/零停机） |
| gsd-incident | `/gsd-incident` | 故障应急（定级/响应/复盘） |
| gsd-observability | `/gsd-observability` | 可观测性（日志/指标/链路） |

环境无关、检查清单驱动，丢进 `~/.<runtime>/skills/` 即可用。详见 [`docs/application-launch/README.md`](docs/application-launch/README.md)。

---

## ③ 开发专家编排框架（核心）

`docs/expert-orchestration/` — 补齐 "MVP → 可用应用" 之间的调优真空。

### 核心问题

业务流水线里有"发散节点"——大模型处理时天然会跑偏：
- 文案仿写：每次仿出来风格飘、结构丢
- Remotion 合成：代码堆成一坨面条，改不动

这类节点不约束就会乱来，MVP 跑通一次，第二次就不可重复。

### 核心概念：专家 vs 普通 skill

| | 普通 skill（gsd 已有） | 专家（本框架新增） |
|---|---|---|
| 解决什么 | 流程对不对 | 产出好不好（收敛发散性） |
| 判断标准 | — | 换个大模型/换次对话，产出会天差地别吗？会→需要专家 |

### 命令（统一入口 + 子命令）

```
/gsd-bianxiezhuanjia              # 看流水线进度（默认）
/gsd-bianxiezhuanjia init         # 初始化编排工作区
/gsd-bianxiezhuanjia copywrite    # 跑仿写专家
/gsd-bianxiezhuanjia remotion     # 跑 Remotion 专家
/gsd-bianxiezhuanjia exit         # 退出编排模式
```

gsd 自带的 `/gsd-plan` `/gsd-impl` `/gsd-ship` 完全不动。

### 关键设计

- **协议层**：专家文件头部带协议字段（`converges` / `input-contract` / `output-contract` / `handoff` / `quality-gate`），编排器据此识别和调度
- **编排层**：人工接力 + 共享状态文件（`.gsd-expert/state/*.json`），零基础设施、可审计、可重跑
- **衔接层**：挂在 `/gsd-ship` 之后、`/gsd-deploy` 之前，不改动 gsd 任何自带命令
- **编排器机制**：status 扫描时对比 `completed-at` 时间戳，自动检测"上游已更新→建议重跑下游"

详见 [`docs/expert-orchestration/README.md`](docs/expert-orchestration/README.md)。

### 案例验证：口播流水线

6 节点流水线（去水印抓文案 → 仿写 → TTS → 数字人 → Remotion 合成 → 自动发布），其中 node-2 仿写、node-5 Remotion 是发散节点走专家，其余 4 个是 tool 节点直接调用。9 阶段模拟验证全部通过。详见 [`docs/expert-orchestration/case-voiceover-pipeline.md`](docs/expert-orchestration/case-voiceover-pipeline.md)。

---

## 目录结构

```
.
├── README.md                              # 本文件
├── overview.md                            # 工程化标准工作总结
└── docs/
    ├── engineering-standards/             # ① 工程化标准（8 篇文档）
    ├── application-launch/                # ② 应用落地 skill 包（6 个 skill）
    │   ├── README.md
    │   └── skills/
    │       ├── gsd-deploy/SKILL.md
    │       ├── gsd-release/SKILL.md
    │       ├── gsd-healthcheck/SKILL.md
    │       ├── gsd-migration/SKILL.md
    │       ├── gsd-incident/SKILL.md
    │       └── gsd-observability/SKILL.md
    └── expert-orchestration/              # ③ 开发专家编排框架
        ├── README.md                      # 框架总览
        ├── expert-protocol.md             # 专家调度协议
        ├── orchestration-flow.md          # 编排流程
        ├── gsd-integration.md             # gsd 衔接点
        ├── case-voiceover-pipeline.md     # 口播流水线案例验证
        └── skills/
            └── gsd-bianxiezhuanjia/       # 统一入口 skill 包
                ├── SKILL.md
                ├── verification-report.md
                └── references/
                    ├── expert-copywrite.md
                    └── expert-remotion.md
```

---

## 快速上手

### 装 skill 到 Claude CLI（或其他支持 gsd 的运行时）

```bash
# 1. 应用落地 skill（6 个）
cp -r docs/application-launch/skills/gsd-* ~/.claude/skills/

# 2. 专家编排框架（1 个统一入口）
cp -r docs/expert-orchestration/skills/gsd-bianxiezhuanjia ~/.claude/skills/
```

重启 Claude CLI，输入 `/gsd` 按 Tab 看可用命令。

### 典型使用流程

```
/gsd-plan                            # 1. 规划（gsd 自带）
/gsd-impl                            # 2. 编码到 MVP（gsd 自带）
/gsd-ship                            # 3. 测试+合并（gsd 自带）

/gsd-bianxiezhuanjia init            # 4. 进入专家编排（MVP→可用调优）
/gsd-bianxiezhuanjia                 #    看进度
/gsd-bianxiezhuanjia copywrite       #    跑仿写专家
/gsd-bianxiezhuanjia remotion        #    跑 Remotion 专家

/gsd-deploy                          # 5. 部署上线（应用落地 skill）
/gsd-healthcheck                     # 6. 健康检查
/gsd-observability                   # 7. 补监控
```

---

## 设计原则

1. **gsd 原生兼容**：所有 skill 都是标准 SKILL.md，丢进 skills 目录即被发现，不改动 gsd 任何自带命令
2. **契约刚性**：专家输入输出契约是硬约束——发散节点最容易"这次结构和上次不一样"，契约锁死结构
3. **人工接力优先**：发散节点产出需人把关，编排器只"建议下一步调谁"，不自动执行
4. **状态全文件化**：所有跨专家信息在 `.gsd-expert/` 文件里，可审计、可重跑、可替换
5. **专家可替换**：专家实现（用哪个模型/prompt）与契约解耦，换实现不改上下游

---

## 适用场景

- 用 gsd/opengsd 开发应用，想把 MVP 推进到可用、稳定上线
- 业务流水线含"发散节点"（大模型产出质量不稳的环节）
- 多技术栈混合团队需要统一的工程化标准

---

## 维护者

Senior Developer（高级开发工程师）

## 版本

v1.2 ｜ 2026-07-03

## License

MIT
