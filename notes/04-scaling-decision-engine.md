# 04 · 伸缩决策引擎：为什么 HPA 会震荡

> 对应掘金：《伸缩决策引擎》｜状态：草稿｜最后更新 2026-11-02
> 源码：`kubestasis/internal/scaler/engine.go`（本文行号对应该文件）

## 一句话

伸缩不难，**不震荡**才难。难点不在"算几个副本"，而在于：算出来的值会改变下一轮的输入，于是系统自己跟自己打架。

```
流量上升 → 扩容 → 单副本负载下降 → 又算出该缩 → 缩完负载又上升 → 又扩…
```

这就是震荡（flapping）。副本数在 3↔5 之间反复横跳，每次都要重建 Pod、重新预热连接，实际体验比不扩容还差。

## 三道闸

我的引擎是三道独立的闸，按顺序卡：

```
原始 desired（由指标算出）
  → ① 上下限钳制（min/max）
  → ② 迟滞带（差值太小不动）
  → ③ 冷却窗口（冷却期内不许缩容）
  → ④ 缩容渐进（单次最多 -20%）
  → 最终 desired
```

### ① 迟滞带（hysteresis）

```go
hysteresis := int32(math.Max(1, float64(current)*0.1))
withinBand := desired-current <= hysteresis && current-desired <= hysteresis
backToMin := desired == policy.MinReplicas && current > policy.MinReplicas
if withinBand && !backToMin { return None }
```

差值在 ±10%（至少 ±1）之内就不动作。**10% 这个带宽必须随当前副本数放大**——写死成 1 的话，20 副本时 ±1 的抖动也会被当真事。

**`backToMin` 这个例外是踩出来的**：不加它，3 副本时算出 desired=2 会被迟滞带判定为"差值 1，在带内"→ 不缩 → **永远缩不回 minReplicas**。所以"回落到下限"必须放行，否则最小副本数形同虚设。

### ② 冷却窗口（cooldown）——只拦缩容

```go
if e.inCooldown(app) && desired < current { return None }
```

**扩容不拦，缩容才拦。** 这是不对称设计，理由是代价不对称：多给几个副本最多浪费点资源；少给几个副本会直接导致超时和 5xx。宁可多花点钱，不能拿可用性冒险。

冷却起点记在 `status.lastScalingAt`，由 `updateStatus` 在真的落地伸缩动作时写入。注意：必须是**落地时**写，不是决策时写——决策了但写入失败，冷却窗口不该开始计时。

### ③ 缩容渐进——`ceil` 那个坑

```go
if desired < current {
    step := int32(math.Ceil(float64(current) * 0.2))   // 每次最多缩 20%
    if step < 1 { step = 1 }
    if floor := current - step; desired < floor { desired = floor }
}
```

**这里有个纯数学陷阱，写反了就永远缩不动：**

| current | `ceil(current*0.8)` ❌ | `current - ceil(current*0.2)` ✅ |
|---|---|---|
| 2 | ceil(1.6) = **2**，缩不动 | 2 − ceil(0.4) = **1** ✓ |
| 3 | ceil(2.4) = **3**，缩不动 | 3 − ceil(0.6) = **2** ✓ |
| 4 | ceil(3.2) = **4**，缩不动 | 4 − ceil(0.8) = **3** ✓ |
| 5 | ceil(4.0) = 4 ✓ | 5 − ceil(1.0) = 4 ✓ |
| 10 | ceil(8) = 8 ✓ | 10 − ceil(2) = 8 ✓ |

直觉写法 `目标 = ceil(当前 × 0.8)` 在小副本数时会被 `ceil` 拉回原值，**2/3/4 副本时一次都缩不动**。必须先算"步长"再减，且步长至少为 1。

扩容为什么一步到位？因为扩容的紧迫性远高于平滑性——流量已经压上来了，分三次扩等于扛三次超时。

## 决策与执行分离

```go
type MetricsSource interface {
    DesiredReplicas(target MetricTarget, current int32) (replicas int32, ok bool)
}
```

`Decide()` 是**纯决策**（不碰集群），`Apply()` 才写。好处是 `Decide` 可以零依赖单测：造一个 fake 的 `MetricsSource`，断言返回的 `Decision` 就行，不需要 envtest、不需要 etcd。

`Apply` 的写法也值得说：

```go
// 改的是 CR 的 spec.replicas，不是直接改 Deployment
target := d.ToReplicas
app.Spec.Replicas = &target
c.Update(ctx, app)
```

**为什么不直接 patch Deployment？** 因为 `ensureDeployment` 每轮都会按 `spec.replicas` 重写 Deployment。你改 Deployment → 下一轮被 spec 覆盖 → 改了个寂寞，还会引发两方互相打架的诡异抖动。

声明式的核心就这一句：**控制器只负责维护"期望状态"，把期望写进 CR，让下游控制器（Deployment controller）去收敛实际状态。** 谁也别越级直接操作别人的对象。

## 我修掉的一个"看起来在工作"的 bug

原代码：

```go
for _, m := range policy.Metrics {
    signals[m.Type] = current        // 每个信号都等于当前副本数
    if signals[m.Type] > desired { desired = signals[m.Type] }
}
```

Prometheus 还没接进来时，为了让代码能编译，把信号填成了当前副本数。后果：`desired == current` 恒成立 → `Decide()` **永远返回 None，一次都不会伸缩**。

可怕的是它不报错、不崩溃、日志干净，`kubectl describe app` 看起来一切正常。**这类"静默失效"是最难查的一类 bug**——它伪装成"当前不需要伸缩"。

修法：没有指标源就明确说出来。

```go
if e.source == nil {
    return Decision{Action: ActionNone, Reason: "no metrics source configured (P1)"}
}
```

同样地，单个信号取不到数据时是 `continue` 跳过，**不能当成 0**：

```go
want, ok := e.source.DesiredReplicas(m, current)
if !ok { continue }   // 该信号暂时没数据，跳过而不是当成 0
```

当成 0 会让一个挂掉的 Prometheus（或还没采集到的新指标）瞬间把副本缩到 min。

## 待补（P1）

- `PrometheusSource` 实现 `MetricsSource`：QPS 用 `rate(http_requests_total[5m])`，P99 用 `histogram_quantile(0.99, ...)`
- `Decide` 的表驱动单测：迟滞带边界、冷却窗口、ceil 缩容那 5 行数据直接变测试用例
- 冷却窗口的时间注入已经留好口子（`nowFunc func() time.Time`），单测里可以瞬间推进时间

## 参考

- HPA 算法（对比用）：https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details
- K8s 社区关于 HPA 稳定性的讨论：`--horizontal-pod-autoscaler-downscale-stabilization`（默认 5min，也是冷却窗口的思路）
