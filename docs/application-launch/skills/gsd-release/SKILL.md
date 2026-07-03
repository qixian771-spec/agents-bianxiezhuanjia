---
description: "Release management: semver tagging, changelog generation, release notes, pre-release checks"
argument-hint: "[--bump <major|minor|patch>] [--prerelease <alpha|beta|rc>]"
---

# gsd-release — 版本发布

> 部署到生产不等于发布。本 skill 管版本号、tag、changelog、release notes，让每一次发布都有据可查、可回溯。与 `gsd-deploy` 配合：deploy 把代码推上去，release 把版本标出来。

## 何时使用

- 准备发布一个新版本（对外或对内的正式版本）
- 需要生成 changelog
- 需要打 tag 并写 release notes
- 发布前需要跑一遍检查清单

## 版本号规则（语义化版本 SemVer）

格式：`MAJOR.MINOR.PATCH`

| 位 | 何时递增 | 示例 |
|----|---------|------|
| MAJOR | 不兼容的 API 变更 | 1.x.x → 2.0.0 |
| MINOR | 向后兼容的新功能 | 1.2.x → 1.3.0 |
| PATCH | 向后兼容的 Bug 修复 | 1.2.3 → 1.2.4 |

### 预发布版本

| 标识 | 用途 | 示例 |
|------|------|------|
| `alpha` | 内部测试，功能可能不完整 | 2.0.0-alpha.1 |
| `beta` | 外部测试，功能完整但可能有 bug | 2.0.0-beta.1 |
| `rc` | 发布候选，如无重大问题即正式发布 | 2.0.0-rc.1 |

### 判断本次该 bump 哪一位

问自己：
1. 这次有没有破坏旧 API（调用方不改代码会挂）？→ MAJOR
2. 这次有没有加新功能（调用方不改代码也能用，但有了新能力）？→ MINOR
3. 这次只是修 bug？→ PATCH

**拿不准时选保守的那一档**——PATCH 误判为 MINOR 最多让用户多升级，MINOR 误判为 MAJOR 会让用户该升级的不升级。

## 发布前检查清单（BLOCK 级）

### 代码与质量
- [ ] 本次发布包含的所有 PR 已合并到主干
- [ ] 主干 CI 全绿
- [ ] 测试覆盖率达到目标（见工程化标准 testing-standards.md）
- [ ] 已知 P0/P1 bug 全部修复（或确认可带病发布并记录）

### 文档
- [ ] CHANGELOG 已更新，覆盖本次所有变更
- [ ] API 文档（如有）已同步更新
- [ ] 升级指南（如有破坏性变更）已编写
- [ ] README 的版本号 / 安装说明已更新

### 依赖与安全
- [ ] 依赖漏洞扫描通过（npm audit / composer audit / pip-audit）
- [ ] 无高危依赖漏洞
- [ ] 许可证合规检查通过

### 部署就绪
- [ ] 迁移脚本已在 staging 验证（见 gsd-migration）
- [ ] 回滚方案已准备（见 gsd-deploy）
- [ ] 监控告警已配置（见 gsd-observability）

### 沟通
- [ ] 发布时间和影响范围已通知相关方
- [ ] 发布窗口已确认（避开业务高峰）
- [ ] 值班人员已就位

## Tag 规范

```bash
# 创建带注释的 tag
git tag -a v1.2.0 -m "Release v1.2.0: 新增退款功能"

# 推送 tag
git push origin v1.2.0
```

格式：`v<MAJOR>.<MINOR>.<PATCH>`，如 `v1.2.0`。预发布加后缀：`v2.0.0-rc.1`。

## CHANGELOG 格式

维护 `CHANGELOG.md`，按版本倒序：

```markdown
# Changelog

## [1.2.0] - 2026-07-02

### Added
- 支持部分金额退款（PROJ-789）
- 新增微信扫码登录

### Changed
- 折扣计算改用策略模式（内部重构，API 不变）

### Fixed
- 修复并发支付重复扣款问题
- 修复 token 过期后未清除本地缓存

### Deprecated
- `GET /api/old-orders` 标记废弃，将于 2.0.0 移除

### Security
- 升级 lodash 修复原型污染漏洞
```

### 分类约定

| 分类 | 包含内容 |
|------|---------|
| Added | 新功能 |
| Changed | 对已有功能的变更 |
| Deprecated | 标记即将移除的功能 |
| Removed | 本次移除的功能（通常对应 MAJOR） |
| Fixed | Bug 修复 |
| Security | 安全相关修复 |

## Release Notes 模板

发布说明面向使用者，讲清"对我有什么影响"：

```markdown
# Release v1.2.0

发布日期：2026-07-02

## 亮点
一句话总结本次发布最重要的 1-3 个变化。

## 新功能
- **部分退款**：现在可以按商品维度退款，不再只能整单退。

## 改进
- 折扣计算性能提升，大促场景下单延迟降低 40%。

## 修复
- 修复偶发的并发支付重复扣款问题。

## 破坏性变更（⚠️ 需注意）
- `GET /api/old-orders` 接口废弃，将于 2.0.0 移除。请迁移到 `GET /api/orders`。
  迁移指南：[链接]

## 升级指南
1. 备份数据库
2. 运行 `php artisan migrate`（新增 2 张表）
3. 更新配置项 `REFUND_MODE=partial`
4. 部署新版本

## 已知问题
- 在 PHP 7.4 下退款回调偶有超时，建议升级到 8.1+。
```

## 发布流程

```
1. 确认发布分支（main 或 release 分支）代码就绪
2. 跑发布前检查清单（BLOCK 项全过）
3. 更新 CHANGELOG.md
4. 打 tag（git tag -a）
5. 推送 tag
6. 触发部署（gsd-deploy）
7. 部署成功后，在代码托管平台创建 Release（附 release notes）
8. 通知相关方
9. 保留发布后的观察窗口（24-48 小时）
```

## 发布后

- [ ] 观察 24 小时，监控核心指标（见 gsd-observability）
- [ ] 收集用户反馈
- [ ] 如发现重大问题，按 gsd-incident 处理，必要时回滚
- [ ] 发布满一周无重大问题，标记为"稳定"

## 与 gsd 工作流的衔接

- `/gsd-complete-milestone` 归档里程碑 → 触发本 skill 做版本发布
- 本 skill 打 tag → 触发 `gsd-deploy` 部署
- 发布出问题 → 触发 `gsd-incident`
