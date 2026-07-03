---
description: "Observability: logging standards, metrics, distributed tracing, dashboards, alert thresholds"
argument-hint: "[--pillar <logs|metrics|traces>] [--scope <service|system>]"
---

# gsd-observability — 可观测性

> 上线之后，系统对你是黑盒——除非你装了"眼睛"。本 skill 建立三大可观测性支柱：日志、指标、链路追踪。没有可观测性的系统不是"稳定运行"，是"瞎着运行"，出问题你才知道，而且是用户告诉你的。

## 何时使用

- 新服务上线，需要建立监控
- 已有服务但监控缺失，想补齐
- 排查问题时发现日志不够、指标没有、链路断裂
- 想设计仪表盘和告警体系

## 三大支柱

```
        可观测性
       ╱    │    ╲
   日志   指标   链路追踪
  (What   (How    (Where
  happened) fast)  it went)
```

| 支柱 | 回答的问题 | 工具示例 | 保留期 |
|------|-----------|---------|-------|
| **日志 Logs** | 发生了什么？细节 | ELK / Loki / 云日志 | 7-30 天 |
| **指标 Metrics** | 多快？多少？趋势 | Prometheus / 云监控 | 3-12 个月 |
| **链路 Traces** | 请求走了哪？卡在哪？ | Jaeger / 云链路 | 1-7 天 |

三者缺一不可：只有日志不知趋势，只有指标不知细节，没有链路定位不了跨服务瓶颈。

## 一、日志

### 日志级别与使用

| 级别 | 场景 | 生产环境 |
|------|------|---------|
| ERROR | 影响功能的异常 | 保留 + 告警 |
| WARN | 可恢复的异常、降级 | 保留 |
| INFO | 关键业务节点 | 保留，控制量 |
| DEBUG | 调试细节 | 默认关，按需开 |

详见工程化标准 code-style-guide.md 第六章。本 skill 补充可观测性相关要求。

### 结构化日志（必须）

日志必须是机器可解析的结构化格式，不是给人读的自由文本：

```python
# ❌ 自由文本，无法检索
logger.info(f"User {user_id} logged in from {ip}")

# ✅ 结构化，可检索可聚合
logger.info("user_login", extra={
    "event": "login",
    "user_id": user_id,
    "ip": ip,
    "request_id": request.id,
    "duration_ms": 120,
})
```

### 必须携带的上下文

每条业务日志至少包含：

| 字段 | 用途 |
|------|------|
| `timestamp` | 时间（框架自动） |
| `level` | 级别（框架自动） |
| `request_id` | 请求唯一 ID，串联同一请求的所有日志 |
| `trace_id` | 链路 ID，串联跨服务调用（见链路追踪） |
| `user_id` | 用户 ID（如已登录） |
| `event` | 事件类型（login / order_create / payment_callback） |

**request_id 是排查生命线**：从入口生成，透传到所有下游调用，这样一条请求的所有日志能被一键捞出。

```php
// Laravel 中间件：注入 request_id
public function handle($request, Closure $next)
{
    $requestId = $request->header('X-Request-ID') ?: Str::uuid();
    Log::withContext(['request_id' => $requestId]);
    $response = $next($request);
    return $response->header('X-Request-ID', $requestId);
}
```

### 日志禁忌

- **禁止打印敏感信息**：密码、token、身份证、完整卡号（脱敏后可）
- **禁止在循环内打 INFO 以上日志**（日志风暴）
- **禁止打印大对象**（整个 request body、大数组）
- **异常日志必须带堆栈**：`logger.error(e.stack)` / `logger.error("x", exc_info=True)`

## 二、指标（Metrics）

### 四类核心指标（USE + RED 方法）

#### 基础设施层 — USE 方法

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| **Utilization** 使用率 | CPU / 内存 / 磁盘 / 连接池占比 | > 80% 持续 5 分钟 |
| **Saturation** 饱和度 | 队列长度、等待数 | 持续增长 |
| **Errors** 错误率 | 硬件/系统错误 | > 0 |

#### 服务层 — RED 方法

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| **Rate** 请求率 | 每秒请求数 QPS | 突增/突降 50% |
| **Errors** 错误率 | 5xx 占比 | > 1% 持续 2 分钟 |
| **Duration** 延迟 | P50 / P95 / P99 | P99 > 基线 2 倍 |

### 必须采集的业务指标

| 指标 | 含义 | 为什么重要 |
|------|------|-----------|
| 订单创建数 | 每分钟下单量 | 业务健康度，突降说明有问题 |
| 支付成功率 | 成功/总尝试 | 核心 KPI，直接关系收入 |
| 注册转化率 | 注册完成/开始注册 | 增长漏斗 |
| 活跃用户数 | 在线/活跃用户 | 容量规划依据 |

**业务指标比技术指标更早发现问题**——支付成功率掉 5% 时，CPU 还没报警。

### 黄金信号仪表盘

每个服务应有标准仪表盘，包含：

```
┌─────────────────────────────────────┐
│ 服务概览                              │
│  [QPS]  [错误率]  [P99延迟]  [实例数] │
├─────────────────────────────────────┤
│ 资源使用                              │
│  [CPU]  [内存]  [磁盘]  [连接池]      │
├─────────────────────────────────────┤
│ 依赖健康                              │
│  [DB延迟]  [缓存命中]  [队列积压]     │
├─────────────────────────────────────┤
│ 业务指标                              │
│  [下单量]  [支付成功率]  [在线用户]   │
└─────────────────────────────────────┘
```

## 三、链路追踪（Distributed Tracing）

### 为什么需要

日志告诉你"单个服务发生了什么"，链路告诉你"一个请求穿过 5 个服务时，时间花在哪了"。

```
用户请求 (1200ms 总耗时)
  ├─ API Gateway (5ms)
  ├─ 订单服务 (1100ms)  ← 瓶颈在这
  │    ├─ 查 DB (80ms)
  │    ├─ 调支付服务 (1000ms)  ← 卡在这
  │    └─ 写日志 (20ms)
  └─ 返回响应
```

没有链路追踪，你只知道"慢"，不知道"慢在哪"。有了链路，一眼定位是支付服务的问题。

### 接入要点

| 要点 | 说明 |
|------|------|
| **生成 trace_id** | 入口生成，透传所有下游 |
| **传播 header** | HTTP 用 `traceparent`，消息队列放消息头 |
| **Span 命名** | 用"服务.操作"，如 `order.query_db` |
| **记录关键事件** | DB 查询、外部调用、业务节点都打 span |
| **采样率** | 高流量 1-10%，低流量 100% |

```php
// Laravel 中传播 trace_id
class TraceMiddleware {
    public function handle($request, Closure $next) {
        $traceId = $request->header('traceparent') ?: generateTraceId();
        $span = tracer()->startSpan('order.handle_request');
        $span->setAttribute('user_id', auth()->id());

        $response = $next($request);

        $span->end();
        return $response->header('traceparent', $traceId);
    }
}
```

## 四、告警体系

### 告警分级

| 级别 | 触发条件 | 通知方式 | 响应 |
|------|---------|---------|------|
| **Critical** | 核心功能不可用 | 电话 + 短信 + IM | 7×24 立即 |
| **Warning** | 指标异常但有缓冲 | IM | 工作时间 |
| **Info** | 异常但不紧急 | IM 群汇总 | 看到就处理 |

### 告警规则设计原则

1. **可执行**：告警要告诉值班"做什么"，不只是"出问题了"
2. **低误报**：阈值基于历史数据，避免抖动误报
3. **去重**：同一根因的多个告警合并
4. **分级**：不是所有异常都打电话

### 告警消息模板

```
[Critical] 支付服务错误率飙升
服务：payment-service
指标：5xx 错误率 = 8.5%（阈值 1%）
持续时间：3 分钟
影响：约 15% 支付请求失败
建议：检查最近部署，准备回滚
仪表盘：https://monitor.example.com/d/payment
```

## 五、可观测性自检清单

新服务上线前：

- [ ] 结构化日志已接入，含 request_id / trace_id
- [ ] 日志级别合理，无敏感信息
- [ ] 核心指标（RED + USE）已采集
- [ ] 业务指标已采集
- [ ] 黄金信号仪表盘已创建
- [ ] 链路追踪已接入，trace_id 跨服务透传
- [ ] 告警规则已配置，阈值合理
- [ ] 告警通知渠道已验证
- [ ] 日志保留期符合要求

## 与 gsd 工作流的衔接

- `/gsd-plan-phase` 规划时 → 识别需要埋的指标和日志
- `/gsd-execute-phase` 执行时 → 日志和埋点随代码一起写
- `gsd-deploy` 部署后 → 验证监控数据正常上报
- `gsd-healthcheck` 的健康端点 → 作为监控数据源之一
- `gsd-incident` 故障时 → 依赖本 skill 的三支柱定位问题
- 故障复盘的监控改进 → 回流到本 skill 更新清单
