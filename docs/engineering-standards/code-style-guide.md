# 代码规范通用指南

> 本文档定义跨语言通用的编码规范。各技术栈专规在此基础上补充语言/框架特定的细则。当专规与本文档冲突时，以专规为准。

---

## 一、命名规范

好的命名是代码可读性的 80%。一个准确的名字能让读者无需看实现就理解意图。

### 1.1 总原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **见名知意** | 名字要表达"是什么"而非"怎么做" | `active_users` ✅ / `data1` ❌ |
| **避免缩写** | 除非是领域通用缩写（URL、ID、HTTP），否则写全词 | `calculate_total` ✅ / `calc_tot` ❌ |
| **避免否定** | 用肯定式，否定式命名增加理解负担 | `is_valid` ✅ / `is_not_invalid` ❌ |
| **保持一致** | 同一概念在整个项目用同一个词 | 别一会 `fetch` 一会 `get` 一会 `retrieve` |

### 1.2 各类标识符命名约定

| 标识符类型 | 约定 | 示例 |
|-----------|------|------|
| 变量、函数 | 小驼峰（JS/TS/Java）或蛇形（Python/PHP/SQL） | `userName` / `user_name` |
| 常量 | 全大写蛇形 | `MAX_RETRY_COUNT` |
| 类、组件 | 大驼峰 | `UserController` / `UserCard` |
| 私有成员 | 前缀下划线或语言约定 | `_internal_cache` / `#privateField` |
| 布尔值 | `is/has/can/should` 开头 | `is_active` / `has_permission` |
| 事件处理 | `on` + 事件名 / `handle` + 事件名 | `onClick` / `handleSubmit` |
| 集合 | 复数名 | `users` 而非 `userList`（除非确需强调类型） |

### 1.3 命名反模式

```javascript
// ❌ 无意义命名
const data = getUsers();          // 什么 data？
const temp = calculate(x);        // temp 永远是坏名字
const flag = true;                // flag 是什么的 flag？

// ❌ 误导性命名
const userList = getUsers().filter(u => u.active);  // 这是 list 还是 filtered？

// ✅ 准确命名
const activeUsers = getUsers().filter(u => u.active);
const retryCount = 3;
const shouldRefresh = true;
```

### 1.4 函数命名

函数名应当是**动词或动宾结构**，回答"做什么"：

```python
# ✅ 动宾结构，意图清晰
def fetch_user_profile(user_id): ...
def validate_email_format(email): ...
def send_password_reset_notification(user): ...

# ❌ 名词作函数名，看不出是动作
def user_profile(user_id): ...        # 是获取？创建？更新？
def email(email): ...                  # 完全不知所云
```

**特殊动词的语义约定**（全项目统一）：

| 动词 | 语义 | 返回 |
|------|------|------|
| `get` / `fetch` | 读取，从内存/DB 获取 | 数据 |
| `find` | 查找，可能找不到 | 数据或 null |
| `create` | 新建并持久化 | 新对象 |
| `build` | 新建但不持久化 | 新对象 |
| `update` | 修改已有 | 修改后的对象 |
| `delete` / `remove` | 删除 | void 或 bool |
| `is` / `has` / `can` | 判断 | bool |
| `to` | 转换 | 转换后的对象 |

---

## 二、代码格式

格式统一靠工具，不靠人。每个项目必须配置格式化工具并纳入 pre-commit 和 CI。

### 2.1 必配工具

| 技术栈 | 格式化工具 | 规范检查 |
|--------|-----------|---------|
| JS/TS/Vue/React | Prettier | ESLint |
| PHP | PHP CS Fixer / Laravel Pint | PHPStan |
| Python | Black + isort | Ruff / flake8 |
| 通用 | EditorConfig | — |

### 2.2 基础格式约定

以下约定是跨语言的基线，各技术栈工具配置应遵循：

- **缩进**：2 空格（JS/TS/前端）或 4 空格（Python/PHP），由 EditorConfig 统一
- **行宽**：不超过 120 字符（含前端模板）
- **行尾**：LF，文件末尾保留空行
- **引号**：JS/TS 用单引号，PHP 用单引号，Python 用双引号（遵循各栈专规）
- **分号**：JS/TS 必须加分号（ASI 陷阱），PHP 必须加分号，Python 无分号
- **尾逗号**：多行结构保留尾逗号（diff 友好）

```javascript
// ✅ 尾逗号，diff 友好
const config = {
  host: 'localhost',
  port: 3306,
  database: 'app',           // ← 尾逗号
};
```

### 2.3 空行规则

- 函数之间：1 个空行
- 逻辑块之间：1 个空行（用空行表达"段落"）
- 类内方法之间：1 个空行
- **禁止**连续 2 个以上空行
- 文件开头和结尾不留空行

```python
def load_config():
    path = find_config_file()
    return parse(path)


def parse(path):               # ← 1 个空行分隔函数
    with open(path) as f:
        return json.load(f)
```

---

## 三、注释规范

注释是给"未来的读者"看的。好的注释解释**为什么**，坏的注释重复**是什么**。

### 3.1 注释分层

| 类型 | 用途 | 何时写 | 何时删 |
|------|------|-------|-------|
| **文档注释** | API/函数/类的契约说明 | 公开接口必写 | 接口废弃时 |
| **意图注释** | 解释"为什么这样做" | 有非显然的业务/技术原因时 | 原因消失时 |
| **TODO/FIXME** | 标记待办 | 临时方案、已知缺陷 | 完成时立即删 |
| **废弃注释** | 标注过时代码 | 代码已废弃但暂不能删 | 真正删除后 |

### 3.2 注释反模式

```javascript
// ❌ 重复代码的注释（代码已经说清楚了）
let count = 0;  // 设置 count 为 0

// ❌ 过时的注释（代码改了注释没改）
// 返回所有用户
function getUsers() {
  return users.filter(u => u.active);  // 实际只返回活跃用户
}

// ❌ 被注释掉的代码（用 Git，别留尸体）
// const oldMethod = () => { ... }

// ✅ 解释意图的注释
// 用批处理而非逐条，因为 DB 在异地，单次往返 50ms
await bulkInsert(records);
```

### 3.3 TODO 规范

```python
# TODO(zhangsan 2026-08-01): 当用户量过万后改为分页加载
# FIXME(lisi 2026-07-15): 并发场景下有竞态，需加分布式锁
```

格式：`TODO/FIXME(负责人 期望解决日期): 说明`。**无负责人、无日期的 TODO 视为无效，CR 不予通过。**

### 3.4 文档注释要素

函数的文档注释至少回答：

1. **做什么**（一句话）
2. **参数**：含义、类型、约束
3. **返回**：含义、类型、可能为空的情况
4. **异常**：会抛什么异常、什么条件下
5. **副作用**：是否修改入参、是否读写外部状态（如果有）

```python
def transfer_balance(from_id: str, to_id: str, amount: Decimal) -> Transfer:
    """
    在两个账户间转账。

    Args:
        from_id: 转出账户 ID，必须存在且余额充足
        to_id: 转入账户 ID，必须存在
        amount: 转账金额，必须为正

    Returns:
        转账记录对象

    Raises:
        AccountNotFound: 账户不存在
        InsufficientBalance: 余额不足
        ValueError: 金额非正

    Note:
        操作是原子的，失败时双方余额不变。
    """
```

---

## 四、文件与目录结构

### 4.1 文件组织原则

- **单一职责**：一个文件只做一件事。一个类一个文件（Java/TS 类），一组相关函数一个文件（脚本/工具）
- **按职责分目录**：不要按"类型"堆砌（所有 controller 一坨），按业务领域聚合
- **文件长度**：软上限 300 行，硬上限 500 行。超过 500 行考虑拆分
- **函数长度**：软上限 40 行，超过 40 行考虑抽取子函数。**一个函数只做一件事**

### 4.2 文件头部

每个源文件应有一个简短头部，说明：

- 文件用途（一句话）
- （可选）所属模块/业务域

```javascript
/**
 * 用户认证中间件 - 校验 JWT 并注入当前用户到请求上下文
 * @module auth/middleware
 */
```

### 4.3 import / use 顺序

统一顺序，分组排列，组间空行：

1. 标准库 / 语言内置
2. 第三方依赖
3. 项目内部模块（@/ 开头或相对路径）
4. 类型定义（TS）

```python
# 1. 标准库
import os
import json
from datetime import datetime

# 2. 第三方
from flask import request, g
from sqlalchemy import orm

# 3. 项目内部
from app.core.exceptions import BusinessError
from app.models.user import User
```

---

## 五、错误处理

错误处理是代码质量的重灾区。原则：**错误必须被处理，不能被吞掉**。

### 5.1 错误处理三原则

1. **快速失败（Fail Fast）**：发现错误立即抛出，不要带着错误状态继续往下走
2. **明确语义**：用业务异常而非裸技术异常上抛，调用方能理解发生了什么
3. **边界兜底**：在最外层（Controller / API 边界 / 任务入口）统一捕获并转为友好响应

### 5.2 异常分层

```
技术异常（DB 连接失败、空指针）
        ↑ 包装
业务异常（余额不足、权限不够）
        ↑ 统一捕获
边界处理（HTTP 响应 / 日志 / 监控）
```

```python
# ❌ 吞掉异常，问题被掩盖
def get_user(user_id):
    try:
        return db.find(user_id)
    except Exception:
        return None        # 调用方分不清是"不存在"还是"出错了"

# ❌ 裸技术异常上抛，调用方无法理解
def withdraw(user_id, amount):
    if balance < amount:
        raise Exception("error")     # 什么 error？

# ✅ 业务异常，语义明确
class InsufficientBalance(BusinessError):
    pass

def withdraw(user_id, amount):
    if balance < amount:
        raise InsufficientBalance(
            f"余额 {balance} 不足，需 {amount}",
            code="BALANCE_INSUFFICIENT"
        )
```

### 5.3 不要忽略异常

捕获了异常就必须处理。如果确实要忽略（如清理资源的失败），必须注释说明原因：

```javascript
try {
  await closeConnection();
} catch (e) {
  // 连接即将销毁，关闭失败可忽略，资源会被 GC 回收
  logger.debug('Connection close failed, will be GCed', e);
}
```

**禁止空 catch（即捕获后什么都不做）。** 至少要记录日志。

### 5.4 防御式编程的度

- **公共 API 边界**：校验所有入参，不信任调用方
- **内部函数**：用类型系统/断言保证前置条件，不重复校验
- **不要过度防御**：每个函数都校验同一组参数是冗余，靠类型和契约保证

---

## 六、日志规范

日志是排查问题的眼睛。原则：**线上出问题，靠日志能还原现场**。

### 6.1 日志级别

| 级别 | 使用场景 | 生产环境 |
|------|---------|---------|
| ERROR | 影响功能的异常，需人工介入 | 保留 + 告警 |
| WARN | 可恢复的异常、降级、接近阈值 | 保留 |
| INFO | 关键业务节点（登录、下单、支付） | 保留，控制量 |
| DEBUG | 调试细节、中间状态 | 默认关闭，按需开启 |

### 6.2 日志内容要素

一条好的日志应包含：

- **时间**（框架自动）
- **级别**（框架自动）
- **上下文标识**：用户 ID、请求 ID、业务 ID——用于串联链路
- **事件描述**：发生了什么
- **关键数据**：相关的参数、结果

```python
# ❌ 无上下文，无法排查
logger.info("user login")
logger.error("something wrong")

# ✅ 有上下文，可串联
logger.info("user login success", extra={
    "user_id": user.id,
    "request_id": request.id,
    "ip": request.remote_addr,
})
```

### 6.3 日志禁忌

- **禁止打印敏感信息**：密码、token、身份证号、完整卡号
- **禁止在循环里打 INFO 以上日志**（会产生日志风暴）
- **禁止拼接字符串构造日志**（用结构化日志，便于检索）
- **异常日志要带堆栈**：`logger.error("xxx failed", exc_info=True)` / `logger.error(e.stack)`

---

## 七、函数设计

### 7.1 函数单一职责

一个函数只做一件事。判断标准：**能否用一句话描述这个函数在做什么**？如果需要"并且"，说明职责过重，该拆。

```javascript
// ❌ 一个函数干三件事
function processOrder(order) {
  // 校验
  if (!order.items.length) throw new Error('empty');
  // 计算价格
  let total = order.items.reduce((s, i) => s + i.price * i.qty, 0);
  // 发邮件
  sendEmail(order.user.email, `订单 ${total} 元`);
}

// ✅ 拆分，各自单一职责
function validateOrder(order) { ... }
function calculateTotal(order) { ... }
function notifyOrderConfirmed(order, total) { ... }
```

### 7.2 参数设计

- **参数数量 ≤ 3**：超过 3 个考虑封装成对象
- **禁止布尔标志参数**：布尔参数意味着函数干两件事，拆成两个函数
- **有默认值的参数放后面**

```python
# ❌ 布尔标志 + 4 个参数
def create_report(data, include_chart, send_email, format, cc_list): ...

# ✅ 配置对象 + 拆分
def create_report(data, options: ReportOptions): ...
def send_report(report, recipients): ...
```

### 7.3 避免副作用

函数要么**计算并返回**（纯函数），要么**执行操作**（命令），不要既返回又修改状态。被调用方不应惊讶于函数"顺手改了点东西"。

```javascript
// ❌ 假装是查询，实则修改了入参
function getTotal(cart) {
  cart.total = cart.items.reduce(...);   // 偷偷改了 cart！
  return cart.total;
}

// ✅ 纯查询
function calculateTotal(cart) {
  return cart.items.reduce(...);          // 不碰 cart
}
```

### 7.4 提前返回，减少嵌套

用 guard clause（卫语句）替代深层嵌套：

```python
# ❌ 箭头型嵌套
def process(user):
    if user:
        if user.is_active:
            if user.has_permission:
                do_work()
            else:
                raise NoPermission()
        else:
            raise InactiveUser()
    else:
        raise UserNotFound()

# ✅ 卫语句，扁平
def process(user):
    if not user:
        raise UserNotFound()
    if not user.is_active:
        raise InactiveUser()
    if not user.has_permission:
        raise NoPermission()
    do_work()
```

---

## 八、重复代码

### 8.1 三次法则

- 第一次写：直接写
- 第二次类似：可以容忍重复
- 第三次类似：**必须抽象**

### 8.2 抽象的陷阱

不是所有重复都该抽象。错误抽象比重复更糟。抽象前问自己：

1. 这几处重复的**本质**是否相同？（还是只是表面相似）
2. 抽象后，未来变化是否会让抽象变得臃肿？
3. 抽象的收益是否大于增加的间接层成本？

如果不确定，**先保留重复，等第三次且本质相同时再抽象**。

---

## 九、魔法值

代码中不应出现裸字面量。所有"有业务含义"的常量必须有名字。

```javascript
// ❌ 魔法值，不知道 86400 是什么
if (Date.now() - lastLogin > 86400000) { ... }

// ❌ 即便注释了，散落各处难维护
if (Date.now() - lastLogin > 86400000) { /* 一天 */ }

// ✅ 命名常量，集中管理
const ONE_DAY_MS = 24 * 60 * 60 * 1000;
if (Date.now() - lastLogin > ONE_DAY_MS) { ... }
```

例外：`0`、`1`、`-1`、`""`、`true/false`、循环索引等"语言级"字面量可直接使用。

---

## 十、配置与环境

### 10.1 配置分离

- **代码**：业务逻辑，进版本库
- **配置**：环境相关（数据库地址、密钥），不进版本库，用 `.env` / 环境变量
- **密钥**：绝不进版本库，用密钥管理服务

### 10.2 配置层级

```
默认配置（代码内）
   ↑ 覆盖
环境配置（.env / config/{env}.yaml）
   ↑ 覆盖
启动参数（命令行 / 环境变量）
```

### 10.3 12-Factor 实践

- 配置通过环境变量注入，代码不区分"开发/测试/生产"
- 同一份代码构建产物可部署到任意环境
- 环境差异通过环境变量体现，而非代码分支

---

## 十一、版本控制与依赖

- **锁定依赖版本**：使用 lockfile（package-lock.json / composer.lock / poetry.lock），提交到版本库
- **定期更新依赖**：至少每月一次检查更新，及时修复安全漏洞
- **谨慎引入新依赖**：引入前评估维护活跃度、体积、许可证、是否可替代
- **禁止使用已废弃/不维护的包**

---

## 十二、性能与资源

### 12.1 数据库

- **N+1 查询**：禁止在循环里查 DB，用预加载 / 批量查询
- **大结果集**：用分页或游标，禁止一次性加载全表
- **索引**：WHERE / JOIN / ORDER BY 涉及的字段建索引，但避免过度索引

### 12.2 资源释放

- 文件、连接、锁用完即释放，用 try-finally 或语言提供的 RAII / with 语法
- 长连接要有超时和重连机制

### 12.3 异步与并发

- IO 密集型操作用异步（网络、磁盘、DB）
- 避免在请求处理中做阻塞调用
- 共享状态修改必须加锁或用并发安全结构

---

## 十三、安全基线

- **输入校验**：所有外部输入（用户输入、API 参数、文件内容）必须校验
- **输出转义**：渲染到 HTML 时转义，防 XSS
- **参数化查询**：禁止字符串拼接 SQL，用 ORM 或预编译语句
- **认证授权**：每个敏感操作必须校验身份和权限，不只靠前端隐藏
- **敏感数据**：传输用 HTTPS，存储用加密，日志不泄露
- **依赖漏洞**：CI 中跑依赖扫描（npm audit / composer audit / pip-audit）

---

## 十四、自检清单

提交代码前，对照以下清单自检（也是 Code Review 的基础）：

- [ ] 命名是否准确、一致、无缩写？
- [ ] 是否有被注释掉的死代码？
- [ ] 是否有魔法值未提取常量？
- [ ] 函数是否过长（>40 行）或参数过多（>3）？
- [ ] 是否有深层嵌套（>3 层）未用卫语句优化？
- [ ] 异常是否被正确处理（无空 catch、无吞异常）？
- [ ] 日志是否包含上下文、是否泄露敏感信息？
- [ ] 是否有明显的 N+1 查询或循环内 IO？
- [ ] 新增的公开接口是否有文档注释？
- [ ] 是否有对应的测试？

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
