# 02 · Service 只是一个抽象：ClusterIP 为什么 ping 不通

> 对应掘金：《Service 与 kube-proxy》｜状态：已发布｜最后更新 2026-11-02

## 一句话

**Service 没有实体。** 它是一组 iptables/IPVS 规则的"名字"；真正决定转发到哪的，是 `EndpointSlice` 里的 IP 列表。`kubectl get svc` 看到的 ClusterIP 在任何网卡上都找不到。

## 三层对象，别搞混

```
Service (stable virtual IP + port)
   ↓ 由 endpoint-controller 根据 selector 实时筛选
EndpointSlice (真实后端 IP:Port 列表)     ← 出问题先查这里
   ↓ 由 kube-proxy 监听
iptables/IPVS 规则（每个节点一份）
```

`endpoint-controller` 的工作方式：拿 `service.spec.selector` 去匹配 **Running 且 Ready** 的 Pod，把命中的写进 EndpointSlice。所以：

> **selector 写错 → Endpoints 为空 → Service 一定不通**，且 Service 本身不会报任何错。

这是我在 KubeSentry 里真实踩到的：Service 的 `spec.selector` 必须和 Deployment 的 `spec.template.metadata.labels` **完全一致**，一个字母之差就没有后端，而 `kubectl get svc` 看起来完全正常。

源码位置（`internal/controller/app_controller.go:224`）：

```go
want.Spec.Selector = selectorLabels(app)   // Service 的 selector
...
want.Spec.Template.Labels = selectorLabels(app)  // Deployment 的 pod 标签
```

两边共用同一个 `selectorLabels()` 函数，从源头杜绝不一致。这不是为了优雅，是为了不被自己坑。

## ClusterIP 为什么 ping 不通

iptables 规则做的是 **DNAT**（改目标地址），只在有 TCP/UDP 流量经过时才生效。ICMP 包确实被 DNAT 了，但没有任何进程会回应它——ClusterIP 背后没有网卡、没有协议栈。

**所以 `ping <clusterIP>` 不通是正常现象，不代表 Service 坏了。** 验证必须用四层：

```bash
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sS -o /dev/null -w '%{http_code}\n' http://<service>.<ns>.svc.cluster.local:<port>
```

## kube-proxy 三种模式

| 模式 | 机制 | 现状 |
|---|---|---|
| userspace | 流量进用户态再转发 | **已废弃**，别用 |
| iptables | 纯内核 DNAT，规则链式匹配 | 默认，节点规模 <1000 够用 |
| IPVS | 内核哈希表 + ipset | 大规模集群，支持更多调度算法（rr/lc/dh/sh/sed/nq） |

iptables 模式的链结构（值得真的看一眼）：

```
PREROUTING / OUTPUT
  └─ KUBE-SERVICES                # 所有 Service 入口
      └─ KUBE-SVC-XXXX            # 单个 Service，按 statistic 模块随机挑一条 SEP
          └─ KUBE-SEP-XXXX        # 单个 Endpoint，做 DNAT 到 Pod IP
```

```bash
# 真机上看到这条链
sudo iptables -t nat -L KUBE-SERVICES -n | head -20
# 看某个 Service 对应哪条链
sudo iptables -t nat -S | grep <clusterIP>
```

**规模问题**：iptables 是 O(n) 链式匹配，Service 上千之后规则更新会明显变慢（一次变更要重写整表）。这是 IPVS 存在的理由——它是 O(1) 查表。

## 四种 Service 类型

| 类型 | 用途 | 注意 |
|---|---|---|
| ClusterIP | 集群内访问 | 默认。ping 不通是正常的 |
| NodePort | 每个节点开同一个高位端口（30000-32767） | 会同时分配 ClusterIP；节点多时端口暴露面大 |
| LoadBalancer | 云厂商 LB | 云上才有效，裸机会一直 Pending |
| ExternalName | DNS CNAME 到外部域名 | 没有 selector，没有 Endpoints，不经过 kube-proxy |

外加 **`clusterIP: None`（Headless）**：不分配 VIP，DNS 直接返回所有 Pod IP。StatefulSet 靠它拿到稳定的 Pod 域名（`pod-0.svc.ns.svc.cluster.local`）。

## Service 没有负载均衡"策略"

`iptables` 模式下 kube-proxy 是**随机**选后端的（statistic 模块的 random 模式），不是轮询。想要会话保持只能靠 `sessionAffinity: ClientIP`，它底层是给 iptables 加 `recent` 模块做超时记录。

真正的流量治理（权重/熔断/重试）不在 Service 层，那是 Service Mesh 的事 → 已进 [`parking-lot.md`](../parking-lot.md)。

## 我怎么验证的

```bash
# 1. 先看有没有后端（90% 的问题在这一步暴露）
kubectl get endpointslices -l kubernetes.io/service-name=<svc> -o yaml

# 2. 看 Service 的 selector 能不能命中 Pod
kubectl get pods -l <key>=<value> --show-labels

# 3. 看 DNS 解析对不对
kubectl run dns --rm -it --image=busybox:1.36 --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local

# 4. 直接在 Pod 里 curl（不要从节点上 curl，节点网络和 Pod 网络不是一回事）
```

## 我为项目记的

KubeSentry 的 `ensureService` 只做 ClusterIP + 单端口映射，`spec.type` 只在创建时设置一次（`app_controller.go:221`）——否则每轮 Reconcile 都会把用户手动改成 NodePort 的字段覆盖回去。这是 Operator 的通用原则：**mutate 函数里只写你负责的字段，用户可能改的字段要么不动，要么只在创建时设一次。**

## 参考

- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/reference/networking/virtual-ips/
