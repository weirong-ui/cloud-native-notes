# 03 · Reconcile 循环：幂等、Finalizer 时序与 status 回写

> 对应掘金：[《从零写一个 K8s Operator：kubebuilder 初始化到第一个 Reconcile》](https://juejin.cn/post/7703105289100951561)｜状态：已发布｜最后更新 2026-11-02
> 源码：`kubestasis/internal/controller/app_controller.go`（本文行号对应该文件）

## 一句话

Reconcile 是一个**不带状态的函数**：给你一个名字，你去查现状、和期望比对、把差异写回去。它可能被并发调用、可能被打断、可能重复执行同一件事——所以唯一的要求是**幂等**。

```go
func (r *AppReconciler) Reconcile(ctx, req ctrl.Request) (ctrl.Result, error)
//                                          ↑ 只有一个 NamespacedName，没有对象
```

签名里没有对象，这是刻意的：controller-runtime 希望你**每次都重读最新对象**，而不是信任任何缓存里的副本。

---

## 1. 为什么必须是幂等的

Reconcile 会被这些事触发：CR 变更、被 Own 的资源变更（`Owns()`）、`RequeueAfter` 到点、控制器重启后全量 List、甚至一次失败重试。

所以 **mutate 函数里只能写"期望值"，不能写"增量"**。反例：

```go
// 错：每执行一次副本数就翻倍
want.Spec.Replicas = ptr.To(*want.Spec.Replicas * 2)

// 对：不管执行多少次，结果都是 spec 说的值
want.Spec.Replicas = &replicas
```

`controllerutil.CreateOrUpdate` 的用法（源码 190 行）：

```go
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, want, func() error {
    want.Labels = selectorLabels(app)
    want.Spec.Replicas = &replicas
    want.Spec.Template.Spec.Containers = []corev1.Container{{...}}
    return controllerutil.SetControllerReference(app, want, r.Scheme)
})
```

一个细节：**`want.Labels` 要打在 Deployment 本身上，不只是 pod template 上**。前者决定 `kubectl get deploy -l ...` 能不能查到，后者决定 Service 的 selector 能不能命中。踩过一次。

## 2. Finalizer 时序（顺序错了就删不掉）

Finalizer 的本质：给对象加一个 `metadata.finalizers` 条目，**有 finalizer 的对象被删除时只会变成"待删除"（`deletionTimestamp` 被置位），不会真消失**，直到 finalizer 被摘掉。

正确时序（源码 105-123 行）：

```go
// A. 对象带 deletionTimestamp → 说明用户已发起删除
if !app.DeletionTimestamp.IsZero() {
    if controllerutil.ContainsFinalizer(&app, appFinalizer) {
        //      ① 先做清理动作（导出审计 / 摘外部资源）
        controllerutil.RemoveFinalizer(&app, appFinalizer)  // ② 再摘 finalizer
        if err := r.Update(ctx, &app); err != nil { ... }   // ③ 最后写回
    }
    return ctrl.Result{}, nil
}

// B. 首次见到 → 加 finalizer
if !controllerutil.ContainsFinalizer(&app, appFinalizer) {
    controllerutil.AddFinalizer(&app, appFinalizer)
    r.Update(ctx, &app)
    return ctrl.Result{RequeueAfter: time.Second}, nil  // 加完就返回，下一轮从头走
}
```

两个坑：

1. **顺序必须是"先清理再摘"**。反过来写的话，如果清理动作失败，finalizer 已经没了，对象直接消失，清理永远补不回来。
2. **加完 finalizer 要立刻 return**。不 return 的话，同一轮里接着用这个对象做后续操作，而 `Update` 之后 `resourceVersion` 已经变了，后面的 `Status().Patch()` 大概率报 conflict。让它下一轮重新进来，最省心。

**经典事故**：加了 finalizer 但忘了在删除分支里摘 → 对象永远卡在 `Terminating`，`kubectl delete` 不报错也不动。生产上只能 `kubectl patch` 手动摘。

## 3. status 必须反映"观测到的真实状态"

这是我在这个项目里修掉的最值得记的一个 bug。原代码：

```go
app.Status.CurrentReplicas = *app.Spec.Replicas   // 错
```

读的是 spec（期望值），写进 status（实际值）。后果：`kubectl get app` 显示的数字**恒等于你写的期望值**——Pod 全挂了也显示 2/2。排障时这比不显示更糟，因为它主动误导你。

修法（源码 249 行）：

```go
ready := deploy.Status.ReadyReplicas   // 从被管理的 Deployment 读真实状态
app.Status.CurrentReplicas = ready
app.Status.DesiredReplicas = desired
```

**判据**：status 里的每一个字段，都应该能追溯到一个"观测动作"。凡是直接从 spec 抄过来的，都是假的。

## 4. Status 用 Patch，不用 Update

```go
patch := client.MergeFrom(app.DeepCopy())
// ...改 app.Status...
r.Status().Patch(ctx, app, patch)
```

三个原因：

1. `Status().Update()` 会整对象覆盖，容易把别人刚改的 spec 冲掉；
2. `MergeFrom(deepcopy)` 自动算出最小 diff，减少 conflict；
3. 没有 status 子资源时，`Status()` 会报错——所以 CRD 必须开 `+kubebuilder:subresource:status`。

**一个真实踩到的坑**：如果 status 字段在 CRD 里被标成 `required`，而它的值是 `0`，`omitempty` 会把 0 从 patch 里剔除，apiserver 随即报 `Required value`。结果是每个 App 在 Pod 起来之前一直报错重试。
→ status 的字段**一律 `+optional` + `omitempty`**。

## 5. ObservedGeneration 是干嘛的

```go
app.Status.ObservedGeneration = app.Generation
```

`metadata.generation` 在 spec 每次变更时 +1。把它记进 status，别人（和你自己）才能判断：**这个 status 是不是已经反映了我刚才那次修改**。

判据：`status.observedGeneration < metadata.generation` → 控制器还没处理到最新一版 spec，`kubectl wait` 应该继续等。不写这个字段，用户无法区分"控制器还没看到"和"控制器已经处理完了但还是这样"。

## 6. Conditions 要用 `meta.SetStatusCondition`

```go
meta.SetStatusCondition(&app.Status.Conditions, metav1.Condition{
    Type:               "Available",
    Status:             metav1.ConditionTrue,
    Reason:             "MinimumReplicasAvailable",
    Message:            fmt.Sprintf("%d/%d replicas ready", ready, desired),
    ObservedGeneration: app.Generation,
})
```

不要手写 `append`。`SetStatusCondition` 帮你处理两件事：**已存在同 type 的条件时去重覆盖**，以及**只在 status 真的变化时才更新 `LastTransitionTime`**。手写的话，后者极易出错——条件被反复刷新成"刚转换"，看事件时间线会完全错乱。

标准条件类型（跟着 K8s 社区约定走，别自创）：`Available` / `Progressing` / `Ready` / `Degraded`。

## 7. Owns() 让链路自己收敛

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&appsv1alpha1.App{}).
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.Service{}).
    Complete(r)
```

`Owns` 的含义：这些资源带 `ownerReference` 指向 App，它们变化时**反查到 App 并触发它的 Reconcile**。效果：

- 有人手贱删了 Service → 下一轮自动重建；
- Deployment 的副本数被 HPA 改了 → App 的 status 立刻跟上，不用等 `RequeueAfter`。

配合末尾的 `return ctrl.Result{RequeueAfter: 15 * time.Second}`，做到"事件驱动 + 周期兜底"双保险。

## 8. 失败就返回 err，别自己重试

```go
if err != nil {
    r.emitEvent(&app, corev1.EventTypeWarning, EventReasonError, "ensure deployment: "+err.Error())
    return ctrl.Result{}, err   // 交给 controller-runtime 的指数退避
}
```

controller-runtime 自带 workqueue 的限速重试（默认 5ms → 1000s 指数退避）。在 Reconcile 里写 `for` 循环重试是最常见的反模式——会堵死 worker，让整个控制器对其他对象也失去响应。

## 我怎么验证的（实测输出）

在 Kubernetes v1.36.1（Docker Desktop）上：

```bash
$ kubectl apply -f config/samples/
$ kubectl wait --for=condition=Available app/sample
$ kubectl get app sample
NAME     PHASE    READY   DESIRED   IMAGE
sample   Stable   2       2         nginx:1.27-alpine

$ kubectl get app sample -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}){.message}{"\n"}{end}'
Available=True (MinimumReplicasAvailable) 2/2 replicas ready
Progressing=False (ReconcileComplete) all replicas are up to date
```

## 参考

- https://book.kubebuilder.io/cronjob-tutorial/controller-overview.html
- controller-runtime：`pkg/builder`、`pkg/controller/controllerutil`
