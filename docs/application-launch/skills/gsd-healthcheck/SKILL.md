---
description: "Health check design: liveness/readiness probes, health endpoints, dependency cascade, degradation"
argument-hint: "[--type <liveness|readiness|startup>] [--depth <shallow|deep>]"
---

# gsd-healthcheck — 健康检查

> 部署完成后，"服务起来了"不等于"服务能用"。本 skill 设计健康检查体系，让编排系统（k8s / 负载均衡 / 监控）能准确判断服务真实状态，避免流量打到还没准备好的实例。

## 何时使用

- 新服务上线，需要设计健康检查端点
- 已有服务但只有简单的 `/ping`，想升级到真正的健康检查
- 部署后验证服务是否真正就绪
- 排查"实例起来了但流量打过去报错"的问题

## 三种探针的区别（关键概念）

| 探针类型 | 回答的问题 | 失败后果 | 检查深度 |
|---------|-----------|---------|---------|
| **Liveness（存活）** | 进程还活着吗？ | 重启实例 | 浅，只查自身 |
| **Readiness（就绪）** | 能接流量了吗？ | 摘除流量，不重启 | 深，查依赖 |
| **Startup（启动）** | 启动完成了吗？ | 阻塞 liveness 直到启动完成 | 一次性 |

**最常见的错误**：把 liveness 和 readiness 搞混，liveness 里检查 DB 连接——结果 DB 抖动一下，所有实例都被重启，雪崩。

## 健康端点设计

### 端点分层

```
GET /health/live      ← 存活探针（浅检查）
GET /health/ready     ← 就绪探针（深检查）
GET /health/startup   ← 启动探针（一次性）
GET /health           ← 综合健康（人类查看 + 监控）
```

### Liveness 端点（浅检查）

只检查"进程自身是否正常"，不查外部依赖：

```php
// Laravel 示例
Route::get('/health/live', function () {
    return response()->json(['status' => 'ok'], 200);
});
```

```python
# Flask 示例
@app.route('/health/live')
def live():
    return jsonify({'status': 'ok'}), 200
```

**检查项**：
- [ ] 进程在运行
- [ ] 事件循环未阻塞（可选，超时检测）
- [ ] 内存未耗尽（可选）

**不要检查**：数据库、缓存、外部 API——这些是 readiness 的事。

### Readiness 端点（深检查）

检查"依赖是否可用，能否正常服务"：

```php
Route::get('/health/ready', function () {
    $checks = [
        'database' => checkDatabaseConnection(),
        'cache' => checkCacheConnection(),
        'queue' => checkQueueWorkerAlive(),
    ];

    $allOk = !in_array(false, $checks, true);

    return response()->json([
        'status' => $allOk ? 'ok' : 'degraded',
        'checks' => $checks,
    ], $allOk ? 200 : 503);
});
```

**检查项**（按依赖优先级）：
- [ ] 数据库连接可用（执行 `SELECT 1`）
- [ ] 缓存连接可用（Redis PING）
- [ ] 消息队列可达
- [ ] 关键外部依赖可用（超时设短，1-2 秒）

### 综合健康端点

供人类和监控系统查看完整状态：

```json
{
  "status": "degraded",
  "version": "1.2.0",
  "uptime": "3d 4h",
  "checks": {
    "database": { "status": "ok", "latency_ms": 5 },
    "cache": { "status": "ok", "latency_ms": 1 },
    "queue": { "status": "fail", "error": "connection refused" },
    "disk": { "status": "ok", "free_gb": 12.5 }
  }
}
```

## 探针配置（以 k8s 为例）

### Liveness 配置原则

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10      # 启动后等 10 秒再开始探
  periodSeconds: 10            # 每 10 秒探一次
  failureThreshold: 3          # 连续失败 3 次才重启
  timeoutSeconds: 2            # 超时 2 秒算失败
```

**initialDelay 要大于启动时间**。启动慢的服务用 startup probe 替代。

### Readiness 配置原则

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5             # 就绪检查可以频繁些
  failureThreshold: 2          # 连续 2 次失败就摘流量
  timeoutSeconds: 3            # 依赖检查可能慢，给 3 秒
```

### Startup 配置（慢启动服务）

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30         # 允许 30 次失败（5 分钟）
  periodSeconds: 10
```

启动期间只探 startup，不探 liveness，避免慢启动被误杀。

## 依赖健康级联

当下游依赖挂了，健康检查应**优雅降级**而非全盘失败：

| 依赖状态 | 服务状态 | 行为 |
|---------|---------|------|
| 所有依赖正常 | healthy | 正常服务 |
| 非核心依赖挂（如统计上报） | degraded | 继续服务，告警 |
| 核心依赖挂（如 DB） | unhealthy | 摘流量，告警 |

```php
function checkReadiness(): array {
    $checks = [];

    // 核心依赖：挂了就摘流量
    $checks['database'] = checkDb();
    $checks['cache'] = checkCache();

    // 非核心依赖：挂了降级但不摘流量
    $checks['analytics'] = ['critical' => false, ...] = checkAnalytics();

    $criticalOk = collect($checks)
        ->filter(fn($c) => $c['critical'])
        ->every(fn($c) => $c['status'] === 'ok');

    return [
        'status' => $criticalOk ? 'ok' : 'unavailable',
        'checks' => $checks,
    ];
}
```

## 健康检查的陷阱

### 1. 健康检查本身拖垮服务

**问题**：readiness 检查 DB，每 5 秒一次，高并发下 DB 连接被健康检查占满。

**解决**：
- 健康检查用独立连接池（或短连接）
- 检查查询极简（`SELECT 1`，不要查业务表）
- 加缓存：3-5 秒内的检查结果复用，避免高频打 DB

```python
_health_cache = {'result': None, 'ts': 0}

@app.route('/health/ready')
def ready():
    if time.time() - _health_cache['ts'] < 3:
        return _health_cache['result']  # 3 秒内复用
    # 实际检查
    result = do_real_check()
    _health_cache['result'] = result
    _health_cache['ts'] = time.time()
    return result
```

### 2. 健康检查通过但服务实际不可用

**问题**：DB 连得上但表锁了，`SELECT 1` 通过，业务请求全超时。

**解决**：readiness 偶尔做一次"真实业务查询"（如查一条测试记录），频率低（每分钟一次）。

### 3. 健康检查导致雪崩

**问题**：liveness 检查依赖，依赖抖动 → 所有实例重启 → 流量集中到剩余实例 → 雪崩。

**解决**：liveness 永远只查自身，绝不查依赖。

## 部署后健康验证清单

部署完成后，逐项验证：

- [ ] `/health/live` 返回 200
- [ ] `/health/ready` 返回 200 且 checks 全 ok
- [ ] 就绪探针通过后，流量已切入（看监控 QPS）
- [ ] 实例数符合预期，无重启记录
- [ ] 健康端点响应时间 < 500ms
- [ ] 综合健康端点显示 version 正确

## 与 gsd 工作流的衔接

- `gsd-deploy` 部署后 → 触发本 skill 验证健康
- 健康检查异常 → 触发 `gsd-deploy` 回滚
- 持续不健康 → 触发 `gsd-incident`
- 健康指标 → 反馈到 `gsd-observability` 仪表盘
