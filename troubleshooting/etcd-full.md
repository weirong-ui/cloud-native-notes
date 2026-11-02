# 排查：etcd 满了（`mvcc: database space exceeded`）

## 现象

集群突然**什么都写不进去**，`kubectl apply` / `create` / `delete` 全部报错，但 `get` 是好的：

```
Error from server: etcdserver: mvcc: database space exceeded
```

原因：etcd 达到配额后进入 **NOSPACE alarm**，主动切换到只读模式保护自己。这是它的设计行为，不是崩溃。

**所以顺序很重要：先确认是 alarm，再释放空间，最后解除 alarm。** 不 compact 就直接 disarm，etcd 会立刻再次进 alarm。

---

## 第 1 步：确认

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

看 `DB SIZE` 是否接近配额（默认 2GiB，`--quota-backend-bytes`）。

```bash
etcdctl ... alarm list
# memberID:x alarm:NOSPACE
```

## 第 2 步：压缩历史版本（compact）

etcd 会保留 key 的所有历史 revision，光删对象不释放空间——**必须手动 compact**。

```bash
# 取当前最新 revision
rev=$(etcdctl ... endpoint status --write-out="json" | jq -r '.[0].Status.header.revision')

# 压缩到当前（保留最后 1000 个 revision 做缓冲）
etcdctl ... compact $((rev-1000))
# compacted revision xxx
```

## 第 3 步：整理碎片（defrag）

compact 只是标记删除，磁盘文件不会变小，**defrag 才是真正回收**。

```bash
etcdctl ... defrag
```

⚠️ 三条注意事项：

1. **defrag 会阻塞该成员**，期间它不响应请求。必须**一个一个来**，不能并发。
2. **leader 最后做**，否则可能触发重新选举。
3. 大 DB（几十 GB）的 defrag 可能要几分钟，别中断。

## 第 4 步：解除 alarm

```bash
etcdctl ... alarm disarm
etcdctl ... alarm list    # 应为空
```

再验证一下能写了：

```bash
kubectl create ns etcd-health-check && kubectl delete ns etcd-health-check
```

---

## 防复发

kubeadm 起的集群，给 etcd 静态 Pod 加自动压缩参数：

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    - --auto-compaction-retention=1      # 保留 1 小时历史
    - --quota-backend-bytes=4294967296   # 配额提到 4GiB
```

改完 kubelet 会自动重启 etcd 静态 Pod。**改之前先备份**：

```bash
etcdctl ... snapshot save /var/backups/etcd-$(date +%F).db
```

> 为什么默认不自动压缩：etcd 的 watch 依赖历史 revision，压缩太激进会让慢消费者 watch 断流。
> 1 小时是个比较安全的折中。

## 常见的"吃空间"元凶

| 元凶 | 特征 | 处理 |
|---|---|---|
| 高频写 CR 的 Operator 没清理 | 某个 CRD 对象数暴涨 | 查 `kubectl get <crd> -A \| wc -l` |
| Event 堆积（默认只留 1h，但量大的集群仍可观） | — | 一般不用管 |
| 反复创建删除的 Job/Pod | `kubectl get events` 能看到 | 修应用逻辑 |
| 大量 revision 的 ConfigMap/Secret 被频繁更新 | 常见于配置热更新写成 update 而非 patch | 改用 patch |

排查"谁在吃空间"：

```bash
# 按 key 前缀统计（需要连到 etcd）
etcdctl ... get "" --prefix --keys-only | awk -F/ '{print $2"/"$3}' | sort | uniq -c | sort -rn | head
```

## 如果 etcd 已经起不来了

配额打满到无法启动的极端情况：临时调大 `--quota-backend-bytes` 让它先起来，**起来后立刻 compact + defrag**，再把参数改回去。不要长期靠放大配额掩盖问题——那只是把爆炸时间往后推。

---

## 我的备注

这条笔记目前和 KubeSentry 没有直接关系，我记它是因为：Operator 本身就是 etcd 的重度写入方（每轮 Reconcile 的 status patch 都会产生新 revision）。**如果将来 KubeSentry 定义的 CR 数量上千，它就是 etcd 空间问题的贡献者之一。** 到时候回来看第 2 步。
