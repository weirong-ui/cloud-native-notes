# 05 · controller-gen 与 CRD 的 8 个坑（血泪版）

> 全部来自 2026-11-02 把 kubesentry 从"不能编译"救到"CI 全绿"的实际过程，每条都带真实报错。
> 状态：持续更新

---

## 坑 1：`+kubebuilder:default={}` 打在结构体上 → CRD 直接装不进去

```go
// 错
type HealthPolicy struct {
    Strategy HealthStrategy `json:"strategy"`
    // +kubebuilder:default={}
    Canary CanaryPolicy `json:"canary"`   // CanaryPolicy 里有 required 字段
}
```

`default={}` 会被原样写进 CRD：`default: {}`。而 `CanaryPolicy` 的子字段是 required 的，于是 apiserver 校验 CRD 时报错——**CRD 根本装不上，不是"默认值不对"，是连 apply 都失败**。

```bash
$ kubectl apply -f config/crd/bases
The CustomResourceDefinition "apps.apps.sentry.io" is invalid:
spec.versions[0].schema.openAPIV3Schema.properties[spec].properties[health].properties[canary].default:
Required value
```

**修法**：结构体上不要写 `default={}`。要默认值就写在**字段级**，并且字段本身是 optional：

```go
type CanaryPolicy struct {
    Enabled bool `json:"enabled"`
    // +kubebuilder:default=300
    ObservationSeconds int `json:"observationSeconds"`
    // +kubebuilder:default="0.01"
    ErrorRateThreshold string `json:"errorRateThreshold"`
}
```

判据：**`default` 只打在叶子字段（标量）上。** 打在结构体上等于让 controller-gen 替你猜一个完整对象，它猜不出来。

## 坑 2：status 字段设成 required → 值为 0 时 patch 报错

```go
// 错
CurrentReplicas int32 `json:"currentReplicas"`   // 无 omitempty → controller-gen 标为 required
```

Go 的 `omitempty` 会让 `0` 从序列化的 patch 里消失，但 CRD 说这个字段 required → 矛盾。

```bash
$ kubectl describe app sample
... .status.currentReplicas: Required value
```

表现很迷惑：**只有 Pod 还没起来（readyReplicas=0）时才报错，起来之后就正常了**，于是每个新 App 在创建初期疯狂重试，日志里全是这个，很容易被误判成"控制器有 bug"。

**修法**：status 字段一律 `+optional` + `omitempty`。status 本来就应该是"能填就填"的观测值，没有任何字段值得强制。

## 坑 3：`controller-gen paths=./...` 在 v0.16.5 下报 no Go files

```bash
$ go run sigs.k8s.io/controller-tools/cmd/controller-gen@v0.16.5 object paths=./...
Error: no Go files in ...
```

**修法**：写显式包路径。

```bash
paths=./api/v1alpha1 paths=./internal/controller paths=./internal/scaler paths=./internal/healer
```

Makefile 里用 `$(subst $(space),$(space)paths=,$(PKG_INTERNAL))` 拼出来，别手写死。

## 坑 4：`Metrics: ctrl.Options{}.Metrics` —— 自引用

```go
// 错：ctrl.Options{}.Metrics 取的是零值，而且这种自引用写法本身就不合法
mgr, _ := ctrl.NewManager(cfg, ctrl.Options{
    Metrics: ctrl.Options{}.Metrics,
})
```

controller-runtime v0.19 里 `Metrics` 的类型是 `metricsserver.Options`。正确写法：

```go
Metrics: metricsserver.Options{BindAddress: metricsAddr},
```

（想关掉就 `BindAddress: "0"`。）

**顺带一个更隐蔽的**：同一个文件里 `Healer` 字段没注入，`r.Healer` 是 nil。当时没 panic 只是因为 `Inspect`/`Apply` 两个方法恰好不解引用接收者——**一旦 P2 真正实现它们，立刻 nil panic**。这种"现在没炸是因为还没实现"的债，越早还越好。

## 坑 5：Windows 的 CRLF 让 `gofmt -l .` 全仓库报脏

`gofmt` 认为 `\r\n` 不符合格式，于是每个文件都被列出来，你改了半天以为自己写错了。

**修法**：一次性统一 + 永久约束。

```bash
printf '* text=auto eol=lf\n*.png binary\n*.jpg binary\n' > .gitattributes
git add --renormalize .
```

批量转换（Python）：

```python
import os
for root, dirs, files in os.walk('.'):
    if '.git' in root.split(os.sep): continue
    for f in files:
        p = os.path.join(root, f)
        d = open(p, 'rb').read()
        if b'\r\n' in d: open(p, 'wb').write(d.replace(b'\r\n', b'\n'))
```

## 坑 6：job 级 `if: hashFiles(...)` → 整份 workflow 一个 job 都不创建

```yaml
# 错：hashFiles() 只在 step 上下文可用
jobs:
  test:
    if: ${{ hashFiles('go.mod') != '' }}
```

后果不是"这个 job 被跳过"，而是**整份 workflow 校验失败**：

```bash
$ curl .../actions/runs/41439643551/jobs
{"total_count": 0}     # 一个 job 都没有
# run 本身 conclusion=failure，耗时 0 秒
```

我最初误判成"job 被门禁挡住了"，实际是连创建都没创建。**判断方法就是查 jobs 数**：`total_count=0` 一定是 workflow 语法/语义错误，不是条件不满足。

**修法**：把条件挪到 step 级。

```yaml
jobs:
  test:
    steps:
      - name: 检查是否有 go.mod
        id: gomod
        run: echo "has=$([ -f go.mod ] && echo true || echo false)" >> $GITHUB_OUTPUT
      - name: 跑测试
        if: steps.gomod.outputs.has == 'true'
        run: go test ./...
```

## 坑 7：`gcr.io/distroless` 国内拉不到

```bash
$ docker build .
=> ERROR [stage-1 1/2] FROM gcr.io/distroless/static-debian12:nonroot
```

**修法**：Dockerfile 里把运行时镜像做成 build-arg，两边各取所需。

```dockerfile
ARG RUNTIME_IMAGE=gcr.io/distroless/static-debian12:nonroot
FROM ${RUNTIME_IMAGE}
```

```bash
# CI 用官方源，本地用镜像源
docker build --build-arg RUNTIME_IMAGE=m.daocloud.io/gcr.io/distroless/static-debian12:nonroot .
```

同理 `GOPROXY` 也做成 build-arg（国内 `https://goproxy.cn,direct`）。

## 坑 8：Dockerfile `COPY` 指向还不存在的目录

```dockerfile
COPY pkg/ pkg/        # pkg/miniinformer 还是 P4，目录压根不存在
COPY go.mod go.sum ./ # 没跑 kubebuilder init 时也不存在
```

构建直接失败。这其实是个**意外的好处**——镜像构建会替你发现 README 里的超前承诺。P0 时我把这两行删了，等 P4 真正有 `pkg/` 再加回来。

---

## 附：两个非 Go 但同样浪费了我时间的

- **Windows 上 `git clone` 报证书吊销失败** → `git -c http.schannelCheckRevoke=false clone ...` 或 `GIT_SSL_NO_VERIFY=1`。curl 同理加 `--ssl-no-revoke`。
- **Docker Desktop 集群里同 tag 镜像被 containerd 缓存**：改了代码重新 `docker build -t kubesentry:e2e` 再部署，集群用的还是旧镜像。验证时**换 tag**（`e2e2`、`e2e3`）最省事，别去研究 imagePullPolicy。
- **Windows 没装 `make`**：Makefile 本身没问题，只是本地跑不了。要么用等价的底层命令，要么装一个 GNU make。

---

## 一条总原则

这 8 个坑里有 5 个（坑 1、2、4、6、8）**本地看代码完全看不出来，只有真的 `kubectl apply` / 真的推到 CI 才会炸**。所以：

> **Operator 项目的验证必须在真集群上做，不能靠"我觉得应该能跑"。**

代价是多花 15 分钟起个 kind，收益是提前发现一堆会在面试官面前爆炸的问题。
