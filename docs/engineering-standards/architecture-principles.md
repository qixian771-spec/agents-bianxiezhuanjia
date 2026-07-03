# 架构设计原则与实践

> 本文档面向 Tech Lead 与骨干开发者，提供架构设计的核心原则、决策框架、常用模式和反模式清单。目标：让系统**可演进、可维护、可测试**，告别"改一处动全身"的耦合泥潭。

---

## 一、架构的本质

架构不是画几张图、分几个层。架构是**在约束下做权衡决策**：用什么样的结构组织代码，使得系统能在**当前需求**下正确工作，同时**未来变化**时能以可控成本演进。

好的架构让"添加新功能"是**加代码**，而非**改代码**。

> 架构师的核心能力不是"会用多少模式"，而是"知道什么时候不用模式"。

---

## 二、六大设计原则（SOLID +）

### 2.1 单一职责原则（SRP）

**一个模块/类/函数只有一个变化的理由。**

判断方法：问自己"这个类会因为什么而修改？"如果答案是多个互不相关的理由，说明职责过重。

```php
// ❌ UserManager 既管持久化又管通知，两个变化理由
class UserManager {
    public function create(array $data) { /* 存 DB */ }
    public function sendWelcomeEmail($user) { /* 发邮件 */ }
    public function exportToCsv() { /* 导出 */ }
}

// ✅ 按职责拆分，各自独立变化
class UserRepository { public function create(array $data) {} }
class UserNotifier { public function sendWelcome($user) {} }
class UserExporter { public function toCsv() {} }
```

### 2.2 开闭原则（OCP）

**对扩展开放，对修改关闭。** 新增功能时，应该通过新增代码实现，而非修改已有代码。

```python
# ❌ 每加一种折扣就要改这个函数（修改关闭被破坏）
def calculate_price(order, discount_type):
    if discount_type == "vip":
        return order.total * 0.8
    elif discount_type == "new_user":
        return order.total * 0.9
    # 每次加新折扣都要改这里...

# ✅ 通过新增策略类扩展，不动原代码
class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, total: Decimal) -> Decimal: ...

class VipDiscount(DiscountStrategy):
    def apply(self, total): return total * Decimal("0.8")

# 新增折扣 = 新增一个类，不改任何旧代码
```

### 2.3 里氏替换原则（LSP）

**子类必须能替换父类而不破坏程序正确性。** 子类不应削弱父类的契约。

经典反例：正方形继承长方形。长方形有独立的宽高，正方形强行让宽高联动，违反了长方形"宽高独立变化"的契约。

```typescript
// ❌ 子类违反父类契约
class Bird { fly() {} }
class Penguin extends Bird {
  fly() { throw new Error("企鹅不会飞"); }  // 父类说能飞，子类说不能
}

// ✅ 重新建模契约
class Bird {}
class FlyingBird extends Bird { fly() {} }
class Penguin extends Bird {}  // 不继承飞行能力
```

### 2.4 接口隔离原则（ISP）

**不要强迫调用方依赖它不用的方法。** 接口要小而专注。

```php
// ❌ 胖接口，实现类被迫实现用不到的方法
interface Worker {
    public function work();
    public function eat();
    public function sleep();
}
class Robot implements Worker {
    public function work() { /* ... */ }
    public function eat() { /* 机器人不吃饭，被迫空实现 */ }
    public function sleep() { /* 被迫空实现 */ }
}

// ✅ 接口拆分，按需实现
interface Workable { public function work(); }
interface Eatable { public function eat(); }
class Robot implements Workable { public function work() {} }
```

### 2.5 依赖倒置原则（DIP）

**高层模块不依赖低层模块，二者都依赖抽象。** 抽象不依赖细节，细节依赖抽象。

这是**依赖注入**的理论基础。业务逻辑不应直接 new 一个具体的数据库/缓存/外部服务，而应依赖接口，由外部注入实现。

```python
# ❌ 高层直接依赖低层具体实现，无法替换、无法测试
class OrderService:
    def __init__(self):
        self.repo = MysqlOrderRepository()      # 写死 MySQL
        self.notifier = EmailNotifier()          # 写死邮件

    def place(self, order):
        self.repo.save(order)
        self.notifier.send(order.user)

# ✅ 依赖抽象，可替换、可测试
class OrderService:
    def __init__(self, repo: OrderRepository, notifier: Notifier):
        self.repo = repo
        self.notifier = notifier

# 测试时注入 Mock，生产时注入真实实现
service = OrderService(InMemoryRepo(), MockNotifier())
```

### 2.6 依赖倒置的进阶：依赖方向

**依赖方向应指向稳定方向。** 稳定的核心域逻辑不依赖易变的基础设施（DB、框架、外部 API）。

```
易变 ←─────────────────→ 稳定
基础设施 → 应用服务 → 领域服务 → 领域模型
(DB/框架)    (用例)     (业务)    (核心)
                          ↑
                  依赖方向指向这里
```

---

## 三、分层架构

### 3.1 经典四层架构

```
┌──────────────────────────────────┐
│  表现层 (Controller / API)        │  接收请求、返回响应、参数校验
├──────────────────────────────────┤
│  应用层 (Service / UseCase)       │  编排业务流程、事务边界
├──────────────────────────────────┤
│  领域层 (Domain Model)            │  核心业务逻辑、领域规则
├──────────────────────────────────┤
│  基础设施层 (Repository / Client) │  DB、缓存、外部 API、框架
└──────────────────────────────────┘
```

**依赖规则：上层依赖下层，下层不感知上层。领域层是核心，不依赖任何层。**

### 3.2 各层职责红线

| 层 | 应该做 | 禁止做 |
|----|--------|-------|
| 表现层 | 参数校验、调用应用层、格式化响应 | 业务逻辑、直接访问 DB |
| 应用层 | 编排领域对象、事务管理、调用基础设施 | 业务规则判断（应在领域层） |
| 领域层 | 业务规则、状态变更、领域事件 | DB 操作、框架 API、IO |
| 基础设施层 | 持久化、外部调用、技术细节 | 业务规则 |

**最常见的违规**：业务逻辑写在 Controller 里，或领域层直接操作 DB。前者让 Controller 臃肿且无法测试，后者让领域层耦合技术细节。

### 3.3 跨层调用规范

- 上层调用下层：通过接口（DIP），不直接 new
- 下层需要通知上层：用**事件 / 回调**，不反向依赖
- 同层之间：尽量无依赖，必要时通过上层编排

---

## 四、模块化与边界

### 4.1 模块划分原则

按**业务领域**划分模块，而非按技术类型。一个"用户"模块内部包含自己的 Controller、Service、Repository、Model，而不是把所有 Controller 堆一起、所有 Service 堆一起。

```
# ❌ 按技术分层目录（模块边界模糊，改一个功能要跳多个目录）
src/
  controllers/   (UserController, OrderController, ...)
  services/      (UserService, OrderService, ...)
  models/        (User, Order, ...)

# ✅ 按业务领域分模块（高内聚，改一个功能只动一个目录）
src/
  user/
    controller.py
    service.py
    repository.py
    model.py
  order/
    controller.py
    service.py
    ...
```

### 4.2 模块间通信

- **同模块内**：直接调用
- **跨模块**：通过明确的接口 / API，不直接 import 对方内部实现
- **避免循环依赖**：A 依赖 B，B 就不能依赖 A；必要时抽取公共模块 C

### 4.3 共享内核

多个模块需要的通用逻辑（如通用工具、基类、值对象）放在 `shared` / `common` 模块。但**警惕共享内核膨胀**——它很容易变成"什么都往里塞"的垃圾桶。每加一个东西都要问：是真的多个模块都需要，还是只是图方便。

---

## 五、常用架构模式

### 5.1 仓库模式（Repository）

隔离领域层与数据访问。领域层面对 `OrderRepository` 接口，不关心背后是 MySQL 还是 MongoDB。

```php
interface OrderRepository {
    public function find(string $id): ?Order;
    public function save(Order $order): void;
}

class MysqlOrderRepository implements OrderRepository {
    // 具体实现细节
}
```

**注意**：Repository 返回的是**领域对象**，不是数据传输对象（DTO）或数组。

### 5.2 服务层模式（Service Layer）

封装业务用例的编排。Controller 调 Service，Service 编排领域对象和 Repository。Service 是**事务边界**。

```python
class PlaceOrderService:
    def __init__(self, order_repo, inventory_repo, payment_client):
        ...

    @transactional
    def execute(self, cmd: PlaceOrderCommand) -> Order:
        # 1. 校验库存
        inventory = self.inventory_repo.get(cmd.sku)
        if inventory.stock < cmd.quantity:
            raise OutOfStock()
        # 2. 创建订单（领域逻辑在 Order 里）
        order = Order.create(cmd.user_id, cmd.sku, cmd.quantity, cmd.price)
        # 3. 扣库存
        inventory.deduct(cmd.quantity)
        # 4. 发起支付
        payment = self.payment_client.charge(order)
        order.markPaid(payment.id)
        # 5. 持久化
        self.order_repo.save(order)
        return order
```

### 5.3 事件驱动

当 A 的操作需要触发 B 的后续处理，且 B 不在 A 的核心职责内，用事件解耦。

```php
// 下单后发邮件、加积分 —— 这些不是下单的核心，用事件解耦
class PlaceOrderService {
    public function execute($cmd) {
        $order = ...; // 创建订单
        $this->events->dispatch(new OrderPlaced($order));  // 发布事件
        return $order;
    }
}

class SendOrderEmailListener {
    public function handle(OrderPlaced $event) { /* 发邮件 */ }
}

class AwardPointsListener {
    public function handle(OrderPlaced $event) { /* 加积分 */ }
}
```

**好处**：下单逻辑不关心后续有哪些副作用，新增副作用只需加 Listener，不改下单代码（OCP）。

### 5.4 策略模式

同一操作有多种算法/规则，用策略模式替代 if-else 链。

### 5.5 工厂模式

对象的创建逻辑复杂（多步骤、依赖注入、根据条件选实现）时，用工厂封装。调用方不应关心"怎么 new"。

---

## 六、架构决策框架

遇到架构选择时，不要凭感觉，用以下框架决策。

### 6.1 决策四问

1. **这个决策的可逆性如何？** 不可逆的决策（如数据库选型）要慎重调研；可逆的（如某个库的使用）快速试错
2. **变化的频率和方向？** 哪里会变、怎么变，朝变化方向做抽象，稳定方向保持简单
3. **团队能力是否支撑？** 再优雅的架构，团队驾驭不了也是灾难
4. **是否有过度设计？** 当前需求是否真的需要这个复杂度？

### 6.2 抽象的时机

| 场景 | 是否抽象 |
|------|---------|
| 第一次出现的需求 | 否，直接写 |
| 第二次类似的重复 | 可暂不抽象，但标记待观察 |
| 第三次且本质相同 | 是，抽象 |
| 预期的未来需求（YAGNI） | 否，等真出现再说 |
| 跨模块的通用能力 | 是，但放 shared 谨慎 |

### 6.3 微服务 vs 单体

**不要一上来就微服务。** 微服务解决的是组织规模问题（团队太多，单体协作冲突），不是技术问题。小团队、早期项目用**模块化单体**：一个部署单元，但内部按模块严格划分边界，未来需要拆分时代价可控。

### 6.4 同步 vs 异步

- **默认同步**：简单、易调试、强一致
- **用异步当**：调用方不关心即时结果、操作耗时长、需要削峰、有最终一致性容忍度
- **异步的代价**：调试难、需处理幂等/重试/补偿/顺序，不要为了"快"而滥用

---

## 七、架构反模式清单

以下反模式在 CR 中应直接打回：

### 7.1 贫血模型

领域对象只有 getter/setter，业务逻辑全在 Service。这等于把数据和逻辑分离，失去了面向对象的意义。

```php
// ❌ 贫血：Order 只是数据袋，逻辑在外部
class Order {
    public $status;  // public 字段，随便改
}
class OrderService {
    public function pay(Order $order) {
        if ($order->status === 'cancelled') throw ...;
        $order->status = 'paid';   // 状态变更逻辑散落各处
    }
}

// ✅ 充血：状态变更封装在 Order 内，不变式得到保护
class Order {
    private string $status;  // private，外部无法直接改

    public function pay(): void {
        if ($this->status === 'cancelled') {
            throw new OrderCancelled();
        }
        $this->status = 'paid';
    }
}
```

### 7.2 上帝类 / 上帝服务

一个类/服务什么都知道、什么都管。特征：超过 1000 行、几十个方法、被到处依赖。一旦修改，影响面巨大。**拆分它**，按职责拆成多个聚焦的服务。

### 7.3 过度抽象

为每个可能的变化都建一层抽象，导致调用链冗长、跳转无数层才能看到真正逻辑。**抽象应有具体收益**（多实现、可测试、解耦），无收益的抽象是负债。

### 7.4 循环依赖

A 依赖 B，B 依赖 A。编译报错或运行时死锁的常见元凶。解法：抽取公共部分 C，让 A、B 都依赖 C。

### 7.5 跨层穿透

Controller 直接操作 Model / DB，绕过 Service 层。短期快，长期业务逻辑无处安放，Controller 膨胀。**每层做好自己的事**。

### 7.6 紧耦合于框架

业务逻辑里到处是框架特有 API（如 Laravel Facade、Spring 注解）。换框架代价巨大。**领域层保持框架无关**，框架只在基础设施层和表现层出现。

### 7.7 配置硬编码

环境差异（地址、密钥、阈值）写死在代码里，靠分支管理环境。用配置中心或环境变量。

### 7.8 忽略幂等性

可重试的操作（消息消费、支付回调、API 调用）不做幂等，重试导致重复创建/扣款。**所有可重试接口必须幂等**。

---

## 八、可演进性设计

### 8.1 向后兼容

API 变更要向后兼容：
- **新增字段**：可选，老客户端忽略即可
- **删除/重命名字段**：先标记废弃，给迁移期，再删除
- **语义变更**：宁可加新接口，不改老接口语义

### 8.2 特性开关

大功能上线用特性开关控制灰度，而非长期开分支。开关要有明确的移除时间点，避免开关堆积。

### 8.3 数据库迁移

- 迁移脚本必须可前向、可回滚
- 大表变更分多步：先加列 → 双写 → 迁移数据 → 切读 → 删旧列
- 永远不在迁移中删除生产数据，先标记废弃

---

## 九、架构文档

每个系统/模块应有以下架构文档（轻量即可）：

1. **架构概览图**：模块划分、依赖关系、外部依赖
2. **关键决策记录（ADR）**：为什么选 A 不选 B，当时的权衡是什么
3. **数据流图**：核心数据如何在系统内流转
4. **部署拓扑**：部署在哪、如何伸缩、容灾策略

ADR 模板：

```markdown
# ADR-001: 选择 PostgreSQL 作为主数据库

## 状态
已采纳 (2026-07-02)

## 背景
需要支持事务、复杂查询、JSON 字段...

## 决策
选择 PostgreSQL 10+

## 权衡
- 选 PostgreSQL：成熟、功能全、社区活跃
- 不选 MySQL：JSON 支持弱于 PG
- 不选 MongoDB：业务强一致需求多

## 后果
- 需要运维 PG 能力
- 未来如需水平拆分需额外方案
```

---

## 十、架构自检清单

设计评审 / CR 时对照：

- [ ] 业务逻辑是否在领域层，而非 Controller / Repository？
- [ ] 依赖方向是否指向稳定方向（领域层）？
- [ ] 是否有跨层穿透（Controller 直连 DB）？
- [ ] 是否有循环依赖？
- [ ] 新增功能是"加代码"还是"改一堆代码"？
- [ ] 抽象是否有真实收益，还是为了抽象而抽象？
- [ ] 可重试操作是否幂等？
- [ ] 公共逻辑是否合理放置（shared 是否在膨胀）？
- [ ] 是否有 ADR 记录关键决策？
- [ ] 模块边界是否清晰，跨模块是否走明确接口？

---

_文档版本：v1.0 ｜ 最后更新：2026-07-02 ｜ 维护者：Senior Developer_
