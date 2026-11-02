# 排查：Service 不通

## 先做一件事：把问题范围砍一半

```bash
kubectl port-forward pod/<pod-name> 8080:8080
curl localhost:8080
```

- **直连 Pod 都不通** → 是应用的问题，跟 Service 没关系，出门左转 [Pod 起不来](pod-not-running.md)。
- **直连通、走 Service 不通** → 确定是 Service 这一层，继续往下。

这一步能省掉大量无意义的折腾。

---

## 第 1 站：Endpoints（80% 的问题在这暴露）

```bash
kubectl get endpoints <svc>
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
```

**Endpoints 是空的，Service 一定不通**，而且 Service 自己不会报任何错。

原因基本就是两个：
1. `spec.selector` 和 Pod 的标签对不上（一个字母之差就没后端）
2. Pod 不 Ready（readinessProbe 失败也会从 Endpoints 里摘掉，哪怕容器在跑）

```bash
# 用 Service 的 selector 去反查，看能不能命中
kubectl get pods -l <key>=<value> --show-labels

# 看哪些 Pod 没 Ready
kubectl get pods -o wide | grep -v "1/1"
```

> 我的项目里踩过一次：Service 的 selector 必须和 Deployment 的 `spec.template.metadata.labels` 完全一致。
> 修法是让两边共用同一个 `selectorLabels()` 函数，从源头杜绝。

## 第 2 站：DNS

```bash
kubectl run dns --rm -it --image=busybox:1.36 --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local
```

- `NXDOMAIN` → Service 名字/命名空间写错，或者跨命名空间访问没带后缀
- 解析出 IP 但和 `kubectl get svc` 的 ClusterIP 不一致 → 有同名 Service 在别的 namespace（**短名只在同 namespace 内有效**）
- CoreDNS 挂了：`kubectl -n kube-system get pods -l k8s-app=kube-dns`

## 第 3 站：真的在集群内 curl

**不要从节点上 curl ClusterIP 来判断**。节点网络和 Pod 网络不是一回事，结果不可信。必须在 Pod 里测：

```bash
kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sS -o /dev/null -w '%{http_code}\n' -m 5 http://<svc>.<ns>.svc.cluster.local:<port>
```

**ClusterIP ping 不通是正常现象**，不是故障（iptables 只做 DNAT，没有进程回应 ICMP）。用 TCP 测，别用 ping。详见 [笔记 02](../notes/02-service-and-kube-proxy.md)。

## 第 4 站：kube-proxy 与转发规则

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide   # 每个节点都该有一个
kubectl -n kube-system logs <kube-proxy-pod> | tail -30
```

在目标节点上看规则（iptables 模式）：

```bash
sudo iptables -t nat -L KUBE-SERVICES -n | grep <clusterIP>
sudo iptables -t nat -S | grep <clusterIP>
```

IPVS 模式：

```bash
sudo ipvsadm -Ln | grep -A5 <clusterIP>
```

## 第 5 站：端口映射对不对

```yaml
ports:
  - port: 80          # Service 暴露的端口
    targetPort: 8080  # Pod 实际监听的端口
```

**`targetPort` 不写时默认等于 `port`**——容器监听 8080、Service 只写了 `port: 80`，就会转发到 80 上，连接被拒。这是最常见的配置错误之一。

```bash
# 确认容器到底监听哪个端口
kubectl exec <pod> -- netstat -tlnp   # 或 ss -tlnp
```

## 第 6 站：NetworkPolicy

一旦 namespace 下配了 NetworkPolicy，**默认是白名单**：没有规则允许就是拒绝，而且是静默丢包。

```bash
kubectl get networkpolicy -n <ns>
```

排查时临时删掉所有 NetworkPolicy 试一次，通了就是策略问题。

---

## 专项

### NodePort 访问不到

- 端口范围必须在 30000–32767
- `externalTrafficPolicy: Local` 时，**只有跑了 Pod 的节点才会响应**，打到没 Pod 的节点上就是连接超时。这是设计如此，不是 bug。
- 节点安全组/防火墙没放行该端口

### Ingress 404 / 503

- 503 = Ingress 找到了 Service 但没有健康后端 → 回到第 1 站查 Endpoints
- 404 = 路径/域名匹配不上，看 Ingress 的 `rules` 和 `pathType`（`Prefix` 和 `Exact` 行为差别很大）

### Headless Service（`clusterIP: None`）

没有 VIP，DNS 直接返回所有 Pod IP。`nslookup` 应该返回多条 A 记录；只返回一条或没有，说明后端没 Ready。

## 速查表

| 现象 | 第一嫌疑 |
|---|---|
| Endpoints 空 | selector 不匹配 / Pod 不 Ready |
| DNS 解析失败 | 名字或 namespace 写错 / CoreDNS 挂了 |
| 连上就断（connection refused） | `targetPort` 不对 |
| 超时无响应 | NetworkPolicy / kube-proxy 没跑 |
| 时通时不通 | 部分 Pod 不 Ready，被负载均衡随机命中 |
| ping 不通 | **正常现象**，用 curl 测 |
