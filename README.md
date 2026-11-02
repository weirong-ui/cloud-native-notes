# cloud-native-notes

> 云原生学习笔记库。**只记"我在项目里真的用到 / 真的踩到"的东西**，不抄文档，不背八股。
> 主线项目：[weirong-ui/kubesentry](https://github.com/weirong-ui/kubesentry)（K8s Operator：自愈 + 智能弹性伸缩）

每一篇笔记都按同一套格式写：

```
机制是什么 → 源码/命令在哪 → 我怎么验证的 → 我踩过的坑
```

凡是"想学但当前项目用不上"的，一律进 [`parking-lot.md`](parking-lot.md)，不占注意力。

---

## 目录

### 原理笔记（`notes/`）

| # | 笔记 | 对应文章 | 关联项目 |
|---|---|---|---|
| 01 | [kube-scheduler 是怎么选中一台节点的](notes/01-kube-scheduler.md) | 掘金《调度器》 | — |
| 02 | [Service 只是一个抽象：ClusterIP 为什么 ping 不通](notes/02-service-and-kube-proxy.md) | 掘金《Service 与 kube-proxy》 | `ensureService` |
| 03 | [Reconcile 循环：幂等、Finalizer 时序与 status 回写](notes/03-operator-reconcile.md) | 掘金《第一个 Reconcile》 | `internal/controller` |
| 04 | [伸缩决策引擎：为什么 HPA 会震荡](notes/04-scaling-decision-engine.md) | 掘金《伸缩决策引擎》 | `internal/scaler` |
| 05 | [controller-gen 与 CRD 的 8 个坑（血泪版）](notes/05-controller-gen-pitfalls.md) | — | P0 全流程 |

### 排查手册（`troubleshooting/`）

面向"线上出事了，我在 5 分钟内要定位"的场景，每篇都是**判定路径 + 命令**，不是概念讲解。

- [Pod 起不来](troubleshooting/pod-not-running.md) — Pending / ImagePullBackOff / CrashLoopBackOff / OOMKilled
- [Service 不通](troubleshooting/service-unreachable.md) — Endpoints 空 / kube-proxy / DNS / NetworkPolicy
- [etcd 满了](troubleshooting/etcd-full.md) — `mvcc: database space exceeded` 的处置顺序

### 其他

- [`parking-lot.md`](parking-lot.md) — 想学但 11 月不碰的清单（eBPF、多集群、Service Mesh……）
- [`commands.md`](commands.md) — 高频命令速查

---

## 状态

| 项 | 值 |
|---|---|
| 笔记数 | 5 篇原理 + 3 篇排查 |
| 最近更新 | 2026-11-02 |
| 写作原则 | 项目当场要用才记；每个结论都能用一条命令验证 |

## License

[CC BY-NC-SA 4.0](LICENSE) — 转载请署名，禁止商用。
