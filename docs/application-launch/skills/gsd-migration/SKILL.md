---
description: "Database migration: forward/backward scripts, large table changes, zero-downtime migration"
argument-hint: "[--direction <up|down>] [--safe-mode] [--table <name>]"
---

# gsd-migration — 数据库迁移

> 数据库变更是部署中风险最高的一环。一条写错的迁移能让全表锁死、服务宕机。本 skill 定义安全的迁移流程，核心原则：**可前向、可回滚、不停机**。

## 何时使用

- 部署涉及数据库结构变更（加表、加列、改类型、加索引）
- 数据迁移（清洗、转换、搬迁）
- 紧急修复需要改 schema
- 回顾已有迁移是否规范

## 迁移铁律（BLOCK 级，违反必出事）

1. **迁移脚本必须有 up 和 down**——能上能下，否则不算完成
2. **大表变更必须分步**——不能一条 ALTER 锁全表
3. **迁移在部署前跑，不在部署时跑**——避免部署和迁移互相阻塞
4. **迁移脚本先在 staging 验证**——生产不是第一次跑的地方
5. **不在迁移里删生产数据**——先标记废弃，给迁移期
6. **迁移脚本纳入版本控制**——和代码一起 review

## 迁移分类与策略

### 1. 加列（低风险）

```php
// Laravel — 加可空列，不锁表
Schema::table('orders', function (Blueprint $table) {
    $table->string('remark')->nullable()->after('status');
});

// down
Schema::table('orders', function (Blueprint $table) {
    $table->dropColumn('remark');
});
```

**要点**：新列必须 `nullable` 或有 `default`，否则老代码插入会报错。

### 2. 删列（高风险，需分步）

**绝不能一步删**——老代码还在用这列，直接删会报错。分四步：

| 步骤 | 时机 | 动作 | 代码是否改 |
|------|------|------|-----------|
| 1 | 第一版部署 | 列改 nullable（如非空），停止写入该列 | 代码停止写该列 |
| 2 | 第一版上线后 | 数据备份（防误删） | — |
| 3 | 第二版部署 | 确认无代码引用该列 | 代码移除该列引用 |
| 4 | 第二版上线观察期后 | 删除该列 | — |

### 3. 改列类型（高风险）

**不能直接 ALTER**——会锁表 + 数据可能丢失。零停机做法：

```
1. 加新列（新类型）
   ALTER TABLE orders ADD COLUMN amount_decimal DECIMAL(10,2);

2. 双写：代码同时写旧列和新列
   // 写入时
   $order->amount = $value;
   $order->amount_decimal = $value;

3. 回填：把旧列数据迁移到新列
   UPDATE orders SET amount_decimal = CAST(amount AS DECIMAL(10,2)) WHERE amount_decimal IS NULL;
   （分批跑，每次 1000 行，避免锁表）

4. 读切换：代码改成读新列
   $order->amount_decimal  // 不再读 amount

5. 停写旧列：代码停止写 amount

6. 删旧列：观察期后删除
```

### 4. 加索引（中风险）

```php
// Laravel — MySQL 加索引会锁表（小表可接受）
Schema::table('orders', function (Blueprint $table) {
    $table->index(['user_id', 'created_at']);
});
```

**大表加索引**（百万行以上）用在线 DDL 工具：
- MySQL：`pt-online-schema-change` 或 `gh-ost`
- PostgreSQL：`CREATE INDEX CONCURRENTLY`（不锁表）

```sql
-- PostgreSQL 不锁表加索引
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at);
```

### 5. 加表（低风险）

直接建，无风险。但注意：
- 外键约束可能阻塞——生产环境建议**不用外键约束**，靠应用层保证
- 初始数据用 seeder 单独跑，别混在迁移里

## 大表变更分步法（核心方法论）

任何涉及大表（10 万行以上）的变更，按"**先扩后缩**"模式：

```
兼容期（兼容新旧 schema）
   │
   ├─ Step 1: 扩展 schema（加列/加表/加索引），老代码继续工作
   ├─ Step 2: 部署新代码，双写双读
   ├─ Step 3: 回填历史数据
   ├─ Step 4: 切读到新 schema
   ├─ Step 5: 停写旧 schema
   └─ Step 6: 缩减 schema（删旧列），部署最终代码

每一步都是独立部署，独立验证，独立可回滚。
```

**核心思想**：schema 变更和代码变更解耦，永远保持"当前代码 + 当前 schema"能正常运行。

## 迁移脚本规范

### 命名

```
YYYYMMDDHHMMSS_create_orders_table.php        # 创建表
YYYYMMDDHHMMSS_add_remark_to_orders.php       # 加列
YYYYMMDDHHMMSS_drop_legacy_amount_column.php  # 删列（分步的最后一步）
```

### up 和 down 必须对称

```python
# Alembic 示例
def upgrade():
    op.add_column('orders', sa.Column('remark', sa.String(255), nullable=True))

def downgrade():
    op.drop_column('orders', 'remark')
```

**down 必须真的能回滚**——不只是写个空函数。回滚后 schema 应回到 upgrade 前的状态。

### 迁移里不要做的事

```python
# ❌ 在迁移里做数据业务逻辑
def upgrade():
    op.add_column('orders', sa.Column('total', sa.Decimal(10, 2)))
    # 不要在迁移里算业务数据
    for order in get_all_orders():
        order.total = calculate_complex_total(order)  # 这应在数据回填脚本里

# ✅ 迁移只改结构，数据回填单独脚本
def upgrade():
    op.add_column('orders', sa.Column('total', sa.Decimal(10, 2), nullable=True))
# 数据回填用 backfill_total.py 单独跑
```

## 迁移执行流程

### 部署前（在 staging 验证）

- [ ] 迁移脚本在 staging 跑通（含 down 回滚）
- [ ] 在 staging 模拟生产数据量压测（大表必须）
- [ ] 估算生产执行时间（大表迁移可能几十分钟）
- [ ] 确认执行时段（避开业务高峰）

### 执行迁移

```bash
# Laravel
php artisan migrate --force          # 生产强制执行
php artisan migrate:rollback          # 回滚上一批

# 确认当前版本
php artisan migrate:status
```

- [ ] 备份相关表（大表变更前必做）
- [ ] 执行 up 迁移
- [ ] 验证 schema 符合预期
- [ ] 跑业务冒烟测试

### 回滚

- [ ] 执行 down 迁移
- [ ] 验证 schema 回到变更前
- [ ] 确认代码与回滚后的 schema 兼容（这是为什么要分步的原因）

## 迁移风险自检清单

- [ ] 是否有 down 脚本？down 真的能回滚吗？
- [ ] 涉及的表有多大？会不会锁表？
- [ ] 新列是 nullable 或有 default 吗？
- [ ] 删列是否给了迁移期（不是一步删）？
- [ ] 改类型是否用了双写双读？
- [ ] 大表加索引用了并发/在线工具吗？
- [ ] 迁移在 staging 验证过吗？
- [ ] 执行前备份了吗？
- [ ] 执行时段避开了高峰吗？

## 与 gsd 工作流的衔接

- `/gsd-plan-phase` 规划阶段 → 识别涉及的迁移，提前设计分步方案
- `/gsd-execute-phase` 执行阶段 → 迁移脚本随代码一起提交、review
- `gsd-deploy` 部署前 → 先跑本 skill 验证迁移
- 迁移导致故障 → 触发 `gsd-incident`，down 脚本是回滚手段之一
