# gsd 应用落地 Skill 包

> 补齐 gsd-core 从"代码合并"到"生产稳定运行"之间的能力真空。gsd 的 `/gsd-ship` 止步于 PR，这套 skill 接管部署、发布、健康检查、迁移、故障应急、可观测性——让 gsd 的工作流真正延伸到**应用落地**。

---

## 为什么需要这套 skill

gsd + ECC 的组合覆盖了"需求 → 代码 → PR 合并"（开发落地），但合并之后是真空：

```
开发落地段（gsd + ECC 已覆盖）          应用落地段（本 skill 包补齐）
需求 → 设计 → 编码 → 测试 → CR → 合并 → 部署 → 发布 → 健康检查 → 运行 → 监控 → 故障应急
                                                  ↑
                                          这一段原来没人管
```

这套 6 个 skill 按 gsd 的 SKILL.md 格式编写，丢进 `~/.<runtime>/skills/` 即可与 gsd 命令无缝协作。

---

## 包含的 skill

| Skill | 命令对应能力 | 解决什么 |
|-------|-------------|---------|
| **gsd-deploy** | 部署编排 | 环境晋升、灰度/蓝绿/滚动、回滚预案、部署前后验证 |
| **gsd-release** | 版本发布 | 语义化版本、tag、changelog、release notes、发布前检查 |
| **gsd-healthcheck** | 健康检查 | 存活/就绪/启动探针、健康端点设计、依赖级联、雪崩防御 |
| **gsd-migration** | 数据库迁移 | 前向/回滚脚本、大表分步变更、零停机迁移、先扩后缩 |
| **gsd-incident** | 故障应急 | 定级、四阶段响应、SOP、复盘模板、告警收敛 |
| **gsd-observability** | 可观测性 | 结构化日志、RED/USE 指标、链路追踪、仪表盘、告警 |

每个 skill 内部都标注了与 gsd 工作流的衔接点，形成闭环：

```
/gsd-ship (合并 PR)
    → gsd-deploy (部署)
        → gsd-healthcheck (验证)
        → gsd-observability (持续监控)
    → gsd-release (打 tag 发版)
    → 出问题 → gsd-incident (应急 + 复盘)
    → 涉及 DB → gsd-migration (安全变更)
```

---

## 安装

### 方式一：复制到运行时 skill 目录

```bash
# 以 Claude Code 为例
cp -r docs/application-launch/skills/gsd-* ~/.claude/skills/

# Cursor
cp -r docs/application-launch/skills/gsd-* ~/.cursor/skills/

# CodeBuddy
cp -r docs/application-launch/skills/gsd-* ~/.codebuddy/skills/
```

### 方式二：项目级安装（仅当前项目生效）

```bash
cp -r docs/application-launch/skills/gsd-* .claude/skills/
```

安装后重启运行时，skill 即可被发现。

---

## 使用方式

这些 skill 会在相关场景被自动激活。你也可以主动触发：

- **部署时**：让 AI "按 gsd-deploy 流程部署到生产"——它会跑部署前检查清单、选策略、验证
- **发版时**：让 AI "按 gsd-release 流程发布 v1.2.0"——它生成 changelog、打 tag、写 release notes
- **出故障时**：让 AI "按 gsd-incident 流程响应"——它定级、引导响应、最后产复盘文档
- **改 DB 时**：让 AI "按 gsd-migration 设计迁移方案"——它判断风险、设计分步变更
- **补监控时**：让 AI "按 gsd-observability 检查可观测性"——它对照清单找缺口

---

## 与工程化标准的关系

这套 skill 是 [工程化标准体系](../engineering-standards/README.md) 在"应用落地"环节的执行器：

| 工程化标准文档 | 对应的应用落地 skill |
|--------------|-------------------|
| 代码规范通用指南（日志/安全章节） | gsd-observability, gsd-deploy |
| 架构设计原则（稳定性章节） | gsd-healthcheck, gsd-incident |
| 测试工程化指南 | gsd-deploy（部署后验证） |
| Git 工作流（版本管理章节） | gsd-release |

工程化标准定义"应该怎样"，这套 skill 定义"在 gsd 工作流里具体怎么做"。

---

## 设计原则

1. **gsd 原生格式**：每个 skill 都是标准 SKILL.md，frontmatter 的 description ≤ 100 字符
2. **环境无关**：skill 提供检查清单和决策点，不绑定具体云厂商或工具，按实际环境填
3. **检查清单驱动**：每个 skill 都有 BLOCK 级检查项，AI 照着跑就不会漏
4. **闭环衔接**：skill 之间互相引用，形成从部署到运维的完整链路
5. **可裁剪**：不需要的 skill 可以不装，不影响其他 skill 工作

---

## 文件结构

```
docs/application-launch/
├── README.md                          # 本文件
└── skills/
    ├── gsd-deploy/SKILL.md            # 部署编排
    ├── gsd-release/SKILL.md           # 版本发布
    ├── gsd-healthcheck/SKILL.md       # 健康检查
    ├── gsd-migration/SKILL.md         # 数据库迁移
    ├── gsd-incident/SKILL.md          # 故障应急
    └── gsd-observability/SKILL.md     # 可观测性
```

---

## 后续可扩展

这套 6 个是"应用落地"的最核心能力。未来可按需补充：

- `gsd-cost` — 云成本监控与优化
- `gsd-security-scan` — 安全扫描与漏洞响应（比 ECC 的 AgentShield 更偏运维侧）
- `gsd-backup` — 备份策略与恢复演练
- `gsd-capacity` — 容量规划与压测

需要哪个再补哪个，保持轻量。
