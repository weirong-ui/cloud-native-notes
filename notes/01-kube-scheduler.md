# 01 · kube-scheduler 是怎么选中一台节点的

> 对应掘金：[《K8s 调度器深度笔记：调度流程、亲和性、污点/容忍、抢占》](https://juejin.cn/post/7702808313616236571)｜状态：已发布｜最后更新 2026-11-02

## 一句话

调度器只干两件事：**过滤出能放的机器**，再**给能放的机器打分**。它从不"启动"Pod，只把 `pod.spec.nodeName` 写回 apiserver——真正起容器的是那台机器上的 kubelet。

最后这一步很关键：**调度 = 写一次 `nodeName` 字段**。想明白这点，"调度失败"的排查路径就清楚了：不是机器没收到指令，是调度器压根没写。

---

## 调度周期 vs 绑定周期

| | 调度周期（Scheduling Cycle） | 绑定周期（Binding Cycle） |
|---|---|---|
| 并发 | **串行**，全局一次一个 Pod | 每个 Pod 一个 goroutine，可并行 |
| 终点 | 到 `Reserve` 为止 | `PreBind` → `Bind` → `PostBind` |
| 为什么分 | 决策要基于一致的资源快照 | 绑定只是写一次 API，慢 IO 不该阻塞下一个 Pod |

`Reserve` 是分界线：过了它就算"已预定"，即使还没绑定。若绑定失败，走 `Unreserve` 回滚。

## 扩展点（Scheduling Framework，1.19+）

老版本叫 Predicate / Priority，现在统一叫 Plugin，分 11 个扩展点：

```
QueueSort      # 决定待调度队列顺序（默认按优先级 + 创建时间）
├─ PreFilter   # 全局预处理，如 PodTopologySpread 预算
├─ Filter      # 过滤：不满足直接淘汰（NodeResourcesFit / NodeAffinity / TaintToleration）
├─ PostFilter  # 全被淘汰时才触发 —— 抢占（Preemption）在这里
├─ PreScore    # 打分前的准备
├─ Score       # 打分 0~100（NodeResourcesBalancedAllocation / ImageLocality / InterPodAffinity）
├─ NormalizeScore
├─ Reserve     # 预定资源（调度周期结束）
├─ Permit      # 放行/拒绝/等待（可以做 Gang Scheduling）
├─ PreBind     # 绑定前，如等待 Volume 挂载
├─ Bind        # 真正写 nodeName
└─ PostBind
```

**注意 `PostFilter`**：只有当 `Filter` 把**所有**节点都淘汰完，才会走到它。也就是说你配了抢占不等于一定会抢占——只有彻底无处可放才考虑赶人。

## 为什么调度器要维护自己的 cache

调度周期基于 **scheduler cache 的快照**（`nodeInfo`），不是每次都查 apiserver 的。原因：

1. 调度周期串行，若每次 Filter 都打 API，QPS 直接压垮 apiserver；
2. 更重要的是**一致性**：快照让同一次调度内所有 Filter/Score 看到的是同一时刻的集群视图，不会出现"前半程看到 4 核、后半程看到 2 核"的撕裂判断。

代价是快照可能过期（节点刚被别的调度器占用），此时 `Reserve` 后的 `Bind` 可能失败 → 触发重新调度。这是设计上的取舍，不是 bug。

## 我怎么验证的

```bash
# 1. 看调度器在想什么（Pending 时第一件事就是看它）
kubectl describe pod <pod> | grep -A15 Events
# 典型输出：0/3 nodes are available: 3 Insufficient cpu.

# 2. 打开调度器日志，能看到每个节点被淘汰的原因
kubectl -n kube-system logs deploy/kube-scheduler -f

# 3. 想知道"如果现在调度，会选中谁"（不真的调度）
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
```

想看每个扩展点的打分明细，给 kube-scheduler 加 `--v=5`，日志里会出现每个 plugin 对每个节点的分数。

## 排障：Pending 且 reason=FailedScheduling

`kubectl describe pod` 的 Events 里那句话就是答案，常见的有：

| 事件文案 | 含义 | 方向 |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | 可分配资源不够 | 看 requests 是不是写大了，或节点真的满了 |
| `node(s) had untolerated taint` | 节点有污点，Pod 没配 toleration | master 节点默认有 `node-role.kubernetes.io/control-plane:NoSchedule` |
| `didn't match Pod's node affinity/selector` | nodeSelector / affinity 不匹配 | 检查标签 |
| `didn't match pod anti-affinity rules` | 反亲和把节点全排除了 | 常见于副本数 > 节点数还配了硬反亲和 |
| `had volume node affinity conflict` | PV 在特定 zone，Pod 被调度到别的 zone | 云盘跨可用区挂载不了 |
| `0/N nodes are available: N node(s) were unschedulable` | 节点被 cordon 了 | `kubectl uncordon <node>` |

**一个容易忽略的**：requests 决定调度，limits 不决定。Pod 的 `resources.requests` 写 0 或者不写，调度器认为它不占资源——这是集群被"撑爆"的常见原因。

## 我为项目记的

调度本身和 KubeSentry 暂时没有交集，但**"调度失败"是 P2 自愈要识别的信号之一**：一个 Pod 长时间 `Pending` 且 `reason=Unschedulable`，不应该只是等着，而应该上报——这属于"资源类故障"，和 CrashLoop 的处理路径完全不同（扩容救不了调度失败）。记在这里，P2 时回来取。

## 参考

- 官方：https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/
- 调度框架设计：https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework
