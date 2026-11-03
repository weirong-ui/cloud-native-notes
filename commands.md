# 高频命令速查

> 只记我实际用过的，不追求全。格式：`命令` → 用来干什么。

## 排障三板斧

```bash
kubectl get pod <p> -o wide              # STATUS + 节点
kubectl describe pod <p>                 # Events（默认只留 1 小时）
kubectl logs <p> --previous              # 上一次崩溃的日志（CrashLoop 必看）
```

## 筛选与批量

```bash
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Completed
kubectl get events --sort-by=.lastTimestamp | tail -30
kubectl get events --field-selector reason=FailedScheduling
kubectl get deploy,svc -l app.kubernetes.io/managed-by=kubestasis
```

## jsonpath 取字段（写脚本时比 -o yaml 干净）

```bash
# Pod 的容器状态（自愈程序要读的就是这个）
kubectl get pod <p> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state.waiting.reason}{"\t"}{.lastState.terminated.reason}{"\n"}{end}'

# 自定义资源的所有 condition
kubectl get app <n> -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}) {.message}{"\n"}{end}'

# 节点可分配资源
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
```

## 网络验证

```bash
# 在集群内 curl（别在节点上 curl ClusterIP 下结论）
kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sS -o /dev/null -w '%{http_code}\n' -m 5 http://<svc>.<ns>.svc.cluster.local:<port>

# 绕过 Service 直连 Pod —— 用来把问题范围砍一半
kubectl port-forward pod/<p> 8080:8080

# DNS
kubectl run dns --rm -it --image=busybox:1.36 --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local

# iptables 转发规则（节点上）
sudo iptables -t nat -S | grep <clusterIP>
```

## 进容器 / 调试

```bash
kubectl exec -it <p> -- sh
kubectl debug -it <p> --image=busybox:1.36 --target=<container>   # 镜像里没 shell 时
kubectl exec <p> -- netstat -tlnp                                  # 确认真实监听端口
```

## CRD / Operator 开发

```bash
# 生成 DeepCopy（controller-gen v0.16.5 必须用显式包路径，不能用 ./...）
go run sigs.k8s.io/controller-tools/cmd/controller-gen@v0.16.5 \
  object:headerFile="hack/boilerplate.go.txt" paths=./api/v1alpha1

# 生成 CRD / RBAC
go run sigs.k8s.io/controller-tools/cmd/controller-gen@v0.16.5 \
  paths=./api/v1alpha1 paths=./internal/controller \
  crd:allowDangerousTypes=true rbac:roleName=manager-role \
  output:crd:artifacts:config=config/crd/bases output:rbac:artifacts:config=config/rbac

# 渲染 + 部署（不依赖外部 kustomize 二进制）
kubectl kustomize config/default | sed 's|IMG_PLACEHOLDER|<img>|g' | kubectl apply -f -

# 等待条件（比 sleep 可靠）
kubectl wait --for=condition=Available app/<n> --timeout=240s

# 卡在 Terminating：手动摘 finalizer
kubectl patch app <n> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

## kind

```bash
kind create cluster --name ks
kind load docker-image <img> --name ks     # 把本地镜像塞进集群，省掉推仓库
kind delete cluster --name ks
```

## Windows 专有

```bash
# git clone 报证书吊销失败（本机 schannel 的问题）
git -c http.schannelCheckRevoke=false clone <url>
GIT_SSL_NO_VERIFY=1 git clone <url>
curl --ssl-no-revoke <url>

# gofmt 全仓库报脏 = CRLF 问题
printf '* text=auto eol=lf\n' > .gitattributes && git add --renormalize .

# 没有 make：用上面的 go run / kubectl kustomize 等价命令
```

## etcd

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
etcdctl ... alarm list / compact <rev> / defrag / alarm disarm
```
