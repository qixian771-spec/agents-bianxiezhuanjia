# 测试工程化指南

> 没有测试的代码是"一次性代码"——你不敢改，因为不知道改了会不会炸。测试不是额外的负担，而是**让代码可安全演进的保险绳**。本指南从零搭建团队的测试体系。

---

## 一、测试的价值与心态

### 1.1 测试到底测什么

测试回答三个问题：

1. **这段代码做了我期望的事吗？**（正确性）
2. **改动之后，原来能用的还能用吗？**（回归保护）
3. **这段代码的设计是否合理？**（可测试性）——难以测试的代码，往往耦合过重

### 1.2 误区澄清

| 误区 | 真相 |
|------|------|
| "测试浪费时间" | 不写测试，调试和返工的时间是写测试的数倍 |
| "测试是 QA 的事" | 开发者必须写单元测试，QA 专注集成/E2E/探索性测试 |
| "测试覆盖率越高越好" | 覆盖率是底线，**测试质量**比覆盖率重要。100% 覆盖率也可能测得很烂 |
| "以后再补测试" | 以后永远不会补。代码一上线就成"历史包袱"，测试要随代码一起写 |
| "改代码太麻烦，先注释掉测试" | 测试挂了说明代码行为变了，先搞清楚是 bug 还是预期变更，而不是消灭报警 |

---

## 二、测试金字塔

### 2.1 经典分层

```
            ╱╲
           ╱  ╲          E2E 测试（少量）
          ╱    ╲         模拟真实用户操作，慢、贵、易碎
         ╱──────╲
        ╱        ╲       集成测试（适量）
       ╱          ╲      测模块间协作，中速
      ╱────────────╲
     ╱              ╲    单元测试（大量）
    ╱                ╲   测单个函数/类，快、稳、多
   ╱──────────────────╲
```

**比例建议**：单元 70% / 集成 20% / E2E 10%。

### 2.2 各层职责

| 层 | 测什么 | 速度 | 稳定性 | 数量 |
|----|--------|------|-------|------|
| **单元测试** | 单个函数/类的逻辑，隔离所有外部依赖 | 极快（ms 级） | 高 | 大量 |
| **集成测试** | 多个模块协作、与 DB/缓存的真实交互 | 中（百 ms 级） | 中 | 适量 |
| **E2E 测试** | 完整用户流程，从入口到出口 | 慢（秒级） | 低（易碎） | 少量，只覆盖核心流程 |

### 2.3 为什么金字塔这样分

- **单元测试快且稳**：能频繁跑（每次保存就跑），快速反馈
- **E2E 测试慢且易碎**：依赖多、环境复杂、UI 变动就挂，跑多了拖慢开发
- **倒金字塔（E2E 多、单元少）是反模式**：测试慢、维护成本高、一改一片红

---

## 三、单元测试

### 3.1 单元测试的"单元"

一个单元 = 一个函数 / 一个方法 / 一个类。**隔离**它的所有外部依赖（DB、网络、文件、时间、随机数），只测它的纯逻辑。

### 3.2 好单元测试的特征（FIRST 原则）

- **F**ast：快，毫秒级。慢了说明有外部依赖没隔离
- **I**ndependent：测试之间互不依赖，可任意顺序运行
- **R**epeatable：可重复运行，结果稳定（不依赖时间、网络、随机）
- **S**elf-validating：自动判断通过/失败，不需人工看输出
- **T**imely：在代码之前或同时写，不要拖到后面补

### 3.3 测试结构（AAA 模式）

```python
def test_withdraw_insufficient_balance_raises():
    # Arrange - 准备
    account = Account(balance=100)
    
    # Act - 执行
    with pytest.raises(InsufficientBalance):
        account.withdraw(150)
    
    # Assert - 断言
    assert account.balance == 100  # 余额不变
```

### 3.4 测试命名

测试名要说明**"在什么场景下，调用什么，期望什么结果"**：

```python
# ❌ 看不出测什么
def test_user(): ...
def test_login_1(): ...

# ✅ 场景清晰
def test_login_with_valid_credentials_returns_token(): ...
def test_login_with_wrong_password_raises_invalid_credentials(): ...
def test_login_with_expired_account_raises_account_expired(): ...
```

### 3.5 一个测试只测一件事

```javascript
// ❌ 一个测试测多个场景，挂了不知道哪个断言失败
test('user validation', () => {
  expect(validate({name: ''})).toBe(false);     // 名字空
  expect(validate({name: 'a'.repeat(200)})).toBe(false);  // 名字超长
  expect(validate({age: -1})).toBe(false);      // 年龄负数
  expect(validate({name: 'Tom', age: 20})).toBe(true);    // 正常
});

// ✅ 每个场景一个测试，失败定位清晰
test('rejects empty name', () => {
  expect(validate({name: ''})).toBe(false);
});
test('rejects name longer than 100', () => {
  expect(validate({name: 'a'.repeat(101)})).toBe(false);
});
```

### 3.6 隔离依赖的手段

| 手段 | 用途 |
|------|------|
| **Mock** | 替换依赖对象，控制其返回值，验证是否被调用 |
| **Stub** | 替换依赖对象，只提供预设返回，不验证调用 |
| **Fake** | 提供简化但可用的实现（如内存版 Repository） |
| **Spy** | 包裹真实对象，记录调用情况 |

```typescript
// Mock 示例：控制 Notifier 的行为，验证是否被调用
const notifier = { send: jest.fn().mockResolvedValue(true) };
const service = new OrderService(notifier);

await service.place(order);

expect(notifier.send).toHaveBeenCalledWith(expect.objectContaining({
  userId: order.userId
}));
```

### 3.7 不要 Mock 一切

过度 Mock 会让测试变成"测 Mock 而非测代码"。原则：

- **隔离外部副作用**（DB、网络、文件、时间）：必须 Mock
- **测自己的逻辑**：用真实实现
- **值对象/纯函数**：用真实实现，不 Mock

### 3.8 测试的可维护性

- **避免测试私有方法**：通过公开接口间接测试私有逻辑
- **避免断言实现细节**：断言"结果对不对"，而非"内部调了几次某方法"（除非调用次数是契约）
- **共享 setUp 提取**：重复的测试数据提取成工厂函数
- **避免测试间共享状态**：每个测试独立准备数据，不依赖前一个测试

---

## 四、集成测试

### 4.1 集成测试测什么

- 模块间协作是否正确（Service + Repository + DB）
- 与真实基础设施的交互（DB 查询、缓存读写、消息队列）
- 事务边界是否正确

### 4.2 数据库集成测试

用**真实数据库**（而非 Mock），但用测试专用实例：

```python
# pytest 示例：每个测试用事务回滚隔离
@pytest.fixture
def db_session():
    session = create_test_session()
    session.begin()
    yield session
    session.rollback()  # 测试结束回滚，互不污染

def test_create_user_persists(db_session):
    repo = UserRepository(db_session)
    repo.create(User(name="Tom"))
    
    found = repo.find_by_name("Tom")
    assert found is not None
```

**原则**：
- 用真实数据库（建议用 Docker 起测试实例，与生产同类型）
- 每个测试独立数据，测试间不共享、不依赖
- 用事务回滚或每次清表来隔离

### 4.3 API 集成测试

测完整 HTTP 请求 → 响应链路：

```php
// Laravel 示例
public function test_create_order_returns_201_and_persists(): void
{
    $response = $this->postJson('/api/orders', [
        'sku' => 'SKU001',
        'quantity' => 2,
    ]);

    $response->assertStatus(201)
        ->assertJsonStructure(['id', 'sku', 'quantity', 'total'])
        ->assertJsonPath('quantity', 2);

    $this->assertDatabaseHas('orders', ['sku' => 'SKU001']);
}
```

---

## 五、E2E 测试

### 5.1 E2E 测什么

只测**核心业务流程**，如：注册 → 登录 → 下单 → 支付 → 查看订单。不测每个按钮、每个字段。

### 5.2 E2E 工具

| 技术栈 | 工具 |
|--------|------|
| Web | Playwright（推荐）/ Cypress |
| API | 各语言 HTTP 客户端 + 断言 |

### 5.3 E2E 原则

- **少而精**：覆盖核心流程即可，细节交给单元/集成测试
- **独立测试环境**：用专用的 E2E 环境，不污染开发/生产数据
- **容忍不稳定**：E2E 易碎，失败先重试确认，不急于禁用
- **数据自包含**：测试自己准备数据、自己清理，不依赖预置数据

---

## 六、TDD 与测试驱动

### 6.1 TDD 循环（红-绿-重构）

```
写一个失败的测试  →  写最少代码让它通过  →  重构改善代码
   （红）              （绿）              （重构）
```

### 6.2 是否强制 TDD

**不强制全员 TDD**，但鼓励在以下场景尝试：
- 逻辑复杂的算法/规则
- Bug 修复（先写复现测试，再修复）
- 新功能的核心逻辑

TDD 的真正价值不是"先写测试"，而是**倒逼出可测试的设计**——当你发现某段代码难以先写测试，往往说明它耦合过重。

---

## 七、测试覆盖率

### 7.1 覆盖率目标

| 类型 | 目标 | 说明 |
|------|------|------|
| 行覆盖率 | ≥ 80% | 最低要求 |
| 分支覆盖率 | ≥ 70% | 更能反映逻辑完整性 |
| 核心域逻辑 | ≥ 90% | 业务核心，零容忍 |
| 基础设施/胶水代码 | 不强求 | Controller、配置等，集成测试覆盖即可 |

### 7.2 覆盖率的正确用法

- **作为下限**：低于阈值说明有大量未测代码，需补
- **不作为上限**：100% 覆盖率不代表测得好，可能全是无效断言
- **关注未覆盖部分**：看哪些分支没测到，是否是风险点

### 7.3 覆盖率工具

| 技术栈 | 工具 |
|--------|------|
| JS/TS | Jest / Vitest + c8 / istanbul |
| PHP | PHPUnit + Xdebug / PCOV |
| Python | pytest-cov / coverage.py |

---

## 八、各技术栈测试实践速查

### 8.1 PHP / Laravel

```bash
# 运行测试
php artisan test
php artisan test --coverage   # 覆盖率
```

- **单元测试**：PHPUnit，`tests/Unit/`
- **集成测试**：Laravel 自带的 TestCase，`tests/Feature/`
- **Mock**：Mockery（Laravel 内置）
- **数据库**：用 `RefreshDatabase` trait，每个测试用事务隔离
- **工厂**：用 Factory 生成测试数据，别手写

```php
class OrderTest extends TestCase
{
    use RefreshDatabase;

    public function test_place_order_creates_record(): void
    {
        $user = User::factory()->create();
        $sku = Product::factory()->create(['stock' => 10]);

        $response = $this->actingAs($user)
            ->postJson('/api/orders', ['sku' => $sku->id, 'quantity' => 2]);

        $response->assertStatus(201);
        $this->assertDatabaseHas('orders', ['user_id' => $user->id]);
        $this->assertEquals(8, $sku->fresh()->stock);  // 库存扣减
    }
}
```

### 8.2 JavaScript / TypeScript

```bash
# 运行测试
npm test
npm run test:coverage
```

- **单元测试**：Vitest（推荐，快）/ Jest
- **组件测试**：Vue Test Utils / React Testing Library
- **E2E**：Playwright
- **Mock**：vi.mock / jest.mock

```typescript
// Vitest 示例
import { describe, it, expect, vi } from 'vitest';
import { createUser } from './user-service';

describe('createUser', () => {
  it('throws when email is invalid', () => {
    expect(() => createUser({ email: 'bad' })).toThrow(InvalidEmail);
  });

  it('calls repository with normalized email', async () => {
    const repo = { save: vi.fn().mockResolvedValue({ id: 1 }) };
    await createUser({ email: '  TOM@X.COM ' }, repo);
    expect(repo.save).toHaveBeenCalledWith(
      expect.objectContaining({ email: 'tom@x.com' })
    );
  });
});
```

### 8.3 Python

```bash
# 运行测试
pytest
pytest --cov=app --cov-report=html
```

- **框架**：pytest（推荐，优于 unittest）
- **Mock**：unittest.mock / pytest-mock
- **Fixture**：pytest fixture 管理测试数据和环境
- **异步测试**：pytest-asyncio

```python
import pytest
from app.services.order import OrderService
from app.core.exceptions import OutOfStock

@pytest.fixture
def inventory_repo(mocker):
    repo = mocker.Mock()
    repo.get.return_value = mocker.Mock(stock=5)
    return repo

def test_place_order_out_of_stock_raises(inventory_repo):
    service = OrderService(inventory_repo, mocker.Mock())
    
    with pytest.raises(OutOfStock):
        service.place(user_id='u1', sku='s1', quantity=10)
```

### 8.4 Node.js

```bash
# 运行测试
npm test
```

- **框架**：Vitest / Jest
- **HTTP 测试**：supertest（测 Express/Fastify 路由）
- **Mock**：vi / jest.mock

---

## 九、CI 中的测试集成

### 9.1 测试在 CI 的位置

```
代码提交 → Lint → 类型检查 → 单元测试 → 集成测试 → 构建 → (人工) → E2E → 部署
```

### 9.2 CI 测试策略

| 阶段 | 跑什么 | 阻断合并 |
|------|-------|---------|
| PR 提交 | Lint + 类型检查 + 单元测试 + 集成测试 | 是 |
| 合并到主干 | 上述 + 覆盖率检查 | 是 |
| 部署前 | E2E 测试（staging 环境） | 是 |
| 定期（每日） | 全量测试 + 安全扫描 | 否（仅报告） |

### 9.3 加速 CI

- **并行执行**：测试文件分片并行跑
- **缓存依赖**：缓存 node_modules / vendor / pip cache
- **分层执行**：单元测试先跑（快反馈），集成测试后跑
- **失败优先**：一个 stage 失败立即停止后续，不浪费时间

---

## 十、测试反模式

### 10.1 测试实现细节

```javascript
// ❌ 断言内部调用顺序，重构就挂
expect(repo.save).toHaveBeenCalledTimes(1);
expect(repo.save).toHaveBeenCalledBefore(notifier.send);
expect(logger.info).toHaveBeenCalledWith('exact message');

// ✅ 断言可观察的结果
expect(result.id).toBeDefined();
expect(await repo.findById(result.id)).not.toBeNull();
```

### 10.2 没有断言的测试

```python
# ❌ 跑完没报错就算通过？可能逻辑根本没执行
def test_create_user():
    create_user("Tom")   # 没 assert！

# ✅ 明确断言
def test_create_user_returns_user_with_id():
    user = create_user("Tom")
    assert user.id is not None
```

### 10.3 测试依赖顺序

```javascript
// ❌ 测试 B 依赖 A 创建的数据，A 挂了 B 也挂，且单独跑 B 会失败
test('A creates user', () => { sharedUser = createUser(); });
test('B updates user', () => { updateUser(sharedUser.id); });

// ✅ 每个测试自包含
test('updates user', () => {
  const user = createUser();   // 自己准备
  updateUser(user.id);
  // ...
});
```

### 10.4 快乐路径测试狂魔

只测正常流程，不测异常、边界、并发。**Bug 往往藏在边界和异常路径**。

```python
# 只测正常流程是不够的
def test_withdraw_normal(): ...           # 余额够，正常扣款

# 还要测
def test_withdraw_insufficient_balance(): ...  # 余额不足
def test_withdraw_zero_amount(): ...            # 零金额
def test_withdraw_negative_amount(): ...        # 负金额
def test_withdraw_concurrent(): ...             # 并发扣款
```

### 10.5 注释掉失败的测试

测试红了，说明代码行为变了。**搞清楚是 bug 还是预期变更**：
- Bug → 修代码
- 预期变更 → 改测试

绝不能注释掉测试让它"绿"——那是把报警器拆了，火灾照烧。

---

## 十一、测试自检清单

写完一个功能，对照检查：

- [ ] 核心逻辑有单元测试？
- [ ] 异常路径和边界情况有测试？
- [ ] 测试命名说明了场景？
- [ ] 测试可独立运行（无顺序依赖）？
- [ ] 外部依赖已隔离（Mock/Fake）？
- [ ] 集成测试覆盖了 DB / API 交互？
- [ ] Bug 修复附带回归测试？
- [ ] 测试在 CI 中自动运行？
- [ ] 覆盖率达到目标？

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
