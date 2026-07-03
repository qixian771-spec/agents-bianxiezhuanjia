# Git 工作流规范

> Git 是团队协作的基石。统一的工作流让分支不乱、历史可读、变更可追溯。本规范定义分支模型、提交规范、PR 流程和版本管理。

---

## 一、分支模型

采用 **GitHub Flow 简化版**：主干 + 功能分支，简单清晰，适合中小团队。

### 1.1 分支类型

| 分支 | 命名 | 生命周期 | 用途 |
|------|------|---------|------|
| 主干 | `main` | 永久 | 始终可部署的稳定版本 |
| 功能分支 | `feat/<jira-id>-<short-desc>` | 临时 | 开发新功能 |
| 修复分支 | `fix/<jira-id>-<short-desc>` | 临时 | 修复 Bug |
| 重构分支 | `refactor/<short-desc>` | 临时 | 重构 |
| 热修分支 | `hotfix/<jira-id>-<short-desc>` | 临时 | 生产紧急修复 |
| 发布分支 | `release/<version>` | 临时（可选） | 版本发布准备 |

### 1.2 命名规范

```
feat/PROJ-123-user-login          ✅ 有 ID、有简述
fix/PROJ-456-payment-callback     ✅
refactor/extract-auth-middleware  ✅ 重构无 ID 可用简述
hotfix/PROJ-789-login-500         ✅

feature/user                       ❌ 太笼统
bugfix                             ❌ 无描述
zhangsan/branch1                   ❌ 用人名而非功能
feat_login                         ❌ 用连字符分段，不用下划线
```

### 1.3 分支生命周期

```
1. 从 main 拉出功能分支
       │
2. 开发、提交（可多次）
       │
3. 同步 main 最新代码（rebase 或 merge）
       │
4. 推送、提 PR
       │
5. Code Review
       │
6. 通过后合入 main，删除功能分支
```

**原则**：
- 功能分支寿命尽量短（< 1 周），长了易冲突
- 合入后**立即删除**功能分支，避免分支堆积
- `main` 分支**禁止直接 push**，必须走 PR

---

## 二、Commit 规范

好的 commit 历史是系统的"编年史"，能让你追溯任何变更的来龙去脉。

### 2.1 Commit Message 格式

采用 **Conventional Commits** 规范：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 2.2 Type 类型

| Type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(auth): 支持微信扫码登录` |
| `fix` | Bug 修复 | `fix(payment): 修复并发支付重复扣款` |
| `docs` | 文档变更 | `docs: 更新部署说明` |
| `style` | 格式调整（不影响逻辑） | `style: 统一缩进` |
| `refactor` | 重构（不改功能） | `refactor(order): 提取价格计算策略` |
| `perf` | 性能优化 | `perf(user): 用户列表改分页查询` |
| `test` | 测试相关 | `test(order): 补充并发下单测试` |
| `chore` | 构建/工具/依赖 | `chore: 升级 lodash 修复漏洞` |
| `ci` | CI 配置 | `ci: 添加缓存加速构建` |
| `revert` | 回滚 | `revert: 撤销 feat(auth) 某次提交` |

### 2.3 各字段规则

**Subject（标题）**：
- 不超过 50 字符
- 用祈使句、现在时："add" 而非 "added" / "adds"
- 不加句号结尾
- 首字母大写（英文）或直接写（中文）
- 说明"做了什么"，而非"我做了什么"

**Scope（范围，可选）**：
- 标注影响的模块，如 `auth`、`order`、`payment`

**Body（正文，可选）**：
- 解释"为什么"做这个变更（What 和 How 看 diff 就知道）
- 每行不超过 72 字符
- 一个变更点一段

**Footer（脚注，可选）**：
- 关联 Issue：`Closes #123` / `Refs #456`
- 标记破坏性变更：`BREAKING CHANGE: xxx`
- 标注贡献者：`Co-authored-by: name <email>`

### 2.4 示例

```
feat(payment): 支持退款部分金额

之前只能整单退款，业务需要支持按商品维度退款。
本次新增 partial refund 接口，复用现有支付通道。

Closes PROJ-789
```

```
fix(auth): 修复 token 过期后未清除本地缓存

token 过期时只清了内存状态，未清 localStorage，
导致用户重新登录后仍携带旧 token 触发 401。
```

```
refactor(order): 用策略模式替代折扣计算的 if-else

随着折扣类型增加到 6 种，if-else 已难以维护。
抽取 DiscountStrategy 接口，每种折扣一个实现类。
行为不变，全部测试通过。
```

### 2.5 Commit 粒度

**一个 commit 做一件事**。判断标准：这个 commit 的 message 能用一句话说清吗？

```
✅ 好的粒度
feat(auth): 添加微信登录
fix(auth): 修复 token 刷新竞态
test(auth): 补充 token 过期测试

❌ 坏的粒度
feat: 添加微信登录、修复 token bug、更新依赖、改了点样式
```

**不要攒一堆改动一次性提交**。开发过程中频繁小步提交，最后用 `git rebase -i` 整理成清晰的 commit 序列。

### 2.6 禁止的做法

- ❌ `git commit -m "update"` / `-m "fix"` / `-m "wip"`（无意义信息）
- ❌ 一个 commit 包含多个不相关功能
- ❌ 提交后强行 push 覆盖他人代码（`--force` 到共享分支）
- ❌ 提交大量注释掉的代码（用 git 历史保留，别留尸体）
- ❌ 提交密钥、配置文件、大文件

---

## 三、Pull Request 流程

### 3.1 PR 标题

PR 标题用 commit message 格式的标题部分：

```
✅ feat(auth): 支持微信扫码登录
✅ fix(payment): 修复并发支付重复扣款
❌ update（看不出做了什么）
❌ WIP（未完成就别提 PR，或标 [WIP] 前缀）
```

### 3.2 PR 流程

```
1. 功能开发完成，自检通过
2. rebase 到 main 最新（保持线性历史）
3. push 功能分支
4. 提 PR，目标分支 main
5. 填写 PR 描述（见 CR 手册附录模板）
6. 指定审查者（按 CR 手册分配规则）
7. 等待审查，响应反馈
8. 通过后，Squash Merge 合入 main
9. 删除功能分支
```

### 3.3 合并策略

推荐 **Squash Merge**：

- 功能分支的多个零碎 commit 压成一个干净的 commit
- main 历史线性、清晰，每个 commit 是一个完整功能
- 保留 PR 编号关联（`(#123)`）

```
# 合并后 main 历史示例
abc1234 feat(auth): 支持微信扫码登录 (#123)
def5678 fix(payment): 修复并发支付重复扣款 (#122)
ghi9012 refactor(order): 提取价格计算策略 (#121)
```

### 3.4 同步主干变更

开发期间 main 有新提交时，两种方式同步：

| 方式 | 命令 | 特点 |
|------|------|------|
| **Rebase**（推荐） | `git rebase main` | 历史线性，但会改写功能分支 commit |
| **Merge** | `git merge main` | 保留合并记录，但历史有分叉 |

Rebase 后需 `git push --force`（仅限自己的功能分支，**绝不对共享分支 force push**）。

### 3.5 冲突解决

1. `git rebase main` 出现冲突
2. 手动解决冲突文件
3. `git add <resolved-files>`
4. `git rebase --continue`（不要用 `git commit`）
5. 重复直到 rebase 完成
6. `git push --force-with-lease`（比 `--force` 安全，避免覆盖他人提交）

**解决冲突的原则**：
- 理解双方的意图，不要机械保留一方删除另一方
- 冲突复杂时，拉对方一起解决
- 解决后跑一遍测试确认没破坏

---

## 四、版本管理

### 4.1 语义化版本（SemVer）

版本号 `MAJOR.MINOR.PATCH`：

| 位 | 何时递增 | 含义 |
|----|---------|------|
| MAJOR | 不兼容的 API 变更 | 破坏性变更 |
| MINOR | 向后兼容的新功能 | 加功能 |
| PATCH | 向后兼容的 Bug 修复 | 修 Bug |

```
1.0.0 → 1.0.1   修了个 Bug
1.0.1 → 1.1.0   加了新功能
1.1.0 → 2.0.0   改了 API，不兼容旧版
```

### 4.2 Tag 规范

发布版本时打 tag：

```bash
git tag -a v1.2.0 -m "Release v1.2.0: 新增退款功能"
git push origin v1.2.0
```

格式：`v<MAJOR>.<MINOR>.<PATCH>`，如 `v1.2.0`。

### 4.3 CHANGELOG

每个版本维护 CHANGELOG，记录变更：

```markdown
## [1.2.0] - 2026-07-02

### Added
- 支持部分金额退款
- 新增微信扫码登录

### Fixed
- 修复并发支付重复扣款问题

### Changed
- 折扣计算改用策略模式（内部重构，API 不变）

### Deprecated
- `GET /api/old-orders` 标记废弃，将于 2.0.0 移除
```

---

## 五、热修流程（Hotfix）

生产环境出紧急 Bug 时的流程：

```
1. 从 main 拉出 hotfix 分支
2. 最小化修复（只改必要的，别顺手重构）
3. 补回归测试
4. 提 PR，加急审查（至少 1 人 + Tech Lead）
5. 合入 main
6. 打新版本 tag，部署
7. （如果 release 分支存在）同步回 release 分支
```

**热修原则**：最小变更、最快恢复、事后补完整测试。别在热修里塞其他改动。

---

## 六、敏感信息防护

### 6.1 禁止入库的内容

- 密钥、token、密码（哪怕注释说"测试用的"）
- `.env` 文件（只提交 `.env.example`）
- 私有配置
- 大文件、二进制文件（用 LFS 或对象存储）
- 数据库导出、日志

### 6.2 .gitignore 必备项

每个项目 `.gitignore` 至少包含：

```
# 环境配置
.env
.env.local
.env.*.local

# 依赖
node_modules/
vendor/
__pycache__/
*.pyc
.venv/

# 构建
dist/
build/
*.log

# IDE
.idea/
.vscode/
*.swp
.DS_Store

# 测试覆盖率
coverage/
.coverage
htmlcov/
```

### 6.3 误提交密钥的处理

如果不慎提交了密钥：

1. **立即吊销该密钥**（不是删 commit 就完事，历史已泄露）
2. 用 `git filter-branch` 或 BFG Repo-Cleaner 清除历史中的密钥
3. 强制 push（通知所有协作者重新 clone）
4. 生成新密钥，更新所有环境

---

## 七、Git 自检清单

提交前对照：

- [ ] 分支命名符合规范？
- [ ] Commit message 符合 Conventional Commits？
- [ ] 每个 commit 是单一职责？
- [ ] 没有提交死代码 / 注释掉的代码？
- [ ] 没有提交密钥 / .env / 大文件？
- [ ] rebase 到 main 最新，无冲突？
- [ ] 本地测试通过？

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
