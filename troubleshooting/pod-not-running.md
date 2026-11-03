# 排查：Pod 起不来

目标：**3 条命令定位到原因**，不靠猜。

## 固定三板斧

```bash
kubectl get pod <pod> -o wide            # ① STATUS + 落在哪个节点
kubectl describe pod <pod>               # ② 看 Events（最底下几条是最近的）
kubectl logs <pod> --previous            # ③ 上一次崩溃的日志（CrashLoop 必看）
```

`--previous` 是关键：容器已经重启过的话，普通 `logs` 看的是新容器，崩溃原因只在上一份日志里。

> ⚠️ Events 默认只保留 1 小时。超过 1 小时的现场，describe 里会是空的，得去看监控或 kube-apiserver 日志。

## 按 STATUS 分诊

| STATUS | 含义 | 下一步 |
|---|---|---|
| `Pending` | 还没调度上，或调度了但起不来 | 看 Events 有没有 `FailedScheduling` |
| `ContainerCreating` | 已调度，正在建容器 | 多半是 CNI / 存储卷 |
| `ImagePullBackOff` | 镜像拉不下来 | 见下 |
| `CrashLoopBackOff` | 起来了又退出，反复重启 | `logs --previous` |
| `OOMKilled` | 内存超限被杀 | 见下 |
| `CreateContainerConfigError` | 引用的 ConfigMap/Secret 不存在 | `kubectl get cm,secret -n <ns>` |
| `Init:Error` / `Init:CrashLoopBackOff` | init 容器失败 | `kubectl logs <pod> -c <init容器名>` |
| `Terminating` 卡住 | 删不掉 | 多半是 finalizer 或节点失联 |
| `Unknown` | 节点失联 | `kubectl get node` 看 NotReady |

---

## Pending

```bash
kubectl describe pod <pod> | grep -A10 Events
```

- **有 `FailedScheduling`** → 调度问题，事件文案直接给答案，见 [笔记 01](../notes/01-kube-scheduler.md) 的对照表。
- **没有任何事件** → 调度器可能压根没跑：
  ```bash
  kubectl -n kube-system get pods -l component=kube-scheduler
  ```
- **节点 NotReady** → 是节点的问题，不是 Pod 的。

## ImagePullBackOff

先区分两个：`ErrImagePull` 是第一次拉取失败，`ImagePullBackOff` 是失败后退避重试。**原因看前者，后者只是状态。**

```bash
kubectl describe pod <pod> | grep -i "failed to pull"
```

常见原因按概率排序：
1. 镜像 tag 写错 / 不存在（最容易犯）
2. 私有仓库没配 `imagePullSecrets`
3. 网络不通（国内拉 `gcr.io` / `k8s.gcr.io` 是重灾区 → 换镜像源）
4. `imagePullPolicy: Always` 但仓库限流（Docker Hub 429）

```bash
# 在节点上手动试一次，最快
docker pull / crictl pull <image>
```

## CrashLoopBackOff

**它本身不是错误，是"重启退避时间"**（10s → 20s → 40s … 上限 5 分钟）。真正的死因只有一处：

```bash
kubectl logs <pod> --previous
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state.waiting.reason}{"\t"}exit={.lastState.terminated.exitCode}{"\n"}{end}'
```

常见 exit code：

| code | 含义 |
|---|---|
| 0 | 正常退出 —— **主进程跑完就退出了**，容器没有前台进程。常见于用 shell 脚本当 entrypoint 但没 `exec`/`tail -f` |
| 1 或 2 | 应用自身报错，看日志 |
| 127 | 命令找不到（entrypoint 写错 / 镜像里没这个二进制） |
| 137 | 被 SIGKILL = **OOM** 或 `kubectl delete --force` |
| 139 | 段错误 |
| 143 | SIGTERM，正常终止流程 |

**exit 0 那个很常见也很坑**：容器设计上就是要长期运行的进程，写了个一次性脚本，K8s 就一直重启它。

## OOMKilled

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
# OOMKilled
```

要点：

- 触发的是 `limits.memory`，不是 `requests`。**没设 limits 的 Pod 不会被 OOMKilled**（它会被节点整体 OOM 时挑中杀掉，reason 不是 OOMKilled）。
- 判断"是设小了还是真的泄漏"：看 `kubectl top pod` 的内存曲线，持续爬升 = 泄漏，稳定在高位 = 就是需要这么多。
- Java 应用经典坑：JVM 堆按**节点内存**算默认值，不认 cgroup limits，必须显式 `-Xmx` 且小于 limits（一般留 25% 给堆外）。
- Exit code 一定是 137。

## ContainerCreating 卡住

```bash
kubectl describe pod <pod> | grep -A5 Events
```

- `failed to set up sandbox ... network plugin` → CNI 没装好 / 没分配 IP
- `MountVolume.SetUp failed ... volume not bound` → PVC 没绑上（看 `kubectl get pvc`）
- `failed to sync secret/cache` → 引用的 Secret 不存在

---

## 通用定位技巧

```bash
# 看 Pod 的"上次"状态 —— 自愈类程序要读的就是这里
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'

# 一次性看所有异常 Pod（比一条条看快得多）
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Completed

# 进了容器调试（镜像里没 shell 时用 ephemeral 容器）
kubectl debug -it <pod> --image=busybox:1.36 --target=<container>
```

## 和 KubeStasis 的关系

P2 的自愈引擎要识别的正是这几个信号，读取路径已经确定：

| 故障 | 读取位置 |
|---|---|
| CrashLoop | `containerStatuses[].state.waiting.reason == "CrashLoopBackOff"` |
| OOM | `containerStatuses[].lastState.terminated.reason == "OOMKilled"` |
| 镜像拉不动 | `containerStatuses[].state.waiting.reason == "ImagePullBackOff"` |
| 调度失败 | `status.phase == Pending` + Events 里 `reason=FailedScheduling` |

注意前三个读的是**容器状态**，第四个读的是 **Events**——两者 API 不同，P2 得分别实现。
