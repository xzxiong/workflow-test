# CN Replicas 为什么是 4：restore-2dev 集群分析

**日期**: 2026-04-22  
**集群**: restore-2dev

---

## 现象

`cnset restore-2dev-s8c32g` 的 replicas 为 4，导致大量 CN Pod 创建，节点资源不足后 OOM，堆积 235+ 个僵尸 Pod。

---

## Replicas 计算来源

模板 `root-cluster-template.yaml.j2` 中定义了两个组件共用 `cn.s8c32g` profile：

### 1. cnSets（共享 CN）

```yaml
cnSets:
- name: shared
  profile: cn.s8c32g
  replicas: 2                    # ← 2 个共享 CN 副本
  scalingConfig:
    minReplicas: 2
    maxReplicas: 10
    policy: AutoScaling
```

共享 CN 通过 **CNClaim** 从 cnPool 中申请 CN Pod。2 个 replicas = 2 个 CNClaim。

### 2. cnPool（CN 池）

```yaml
cnPools:
- name: s2
  profile: cn.s8c32g
  poolStrategy:
    scaleStrategy:
      maxIdle: 1                 # ← 池中保持 1 个空闲 CN
    updateStrategy:
      reclaimTimeout: 24h0m0s
```

cnPool 需要满足：已绑定的 CNClaim 数 + maxIdle 个空闲 CN。

### 3. 计算过程

```
cnSet replicas = CNClaim 数 + maxIdle + 补偿
               = 2 (shared claims) + 1 (maxIdle) + 1 (operator 补偿)
               = 4
```

operator 的补偿逻辑：当 CN Pod 启动失败（OOM/ContainerStatusUnknown）时，CNClaim 无法绑定到健康的 CN，operator 认为 pool 中可用 CN 不足，增加 replicas 试图补偿。

---

## 雪崩过程

```
1. operator 创建 4 个 CN Pod（2 claim + 1 idle + 1 补偿）
2. 节点只有 1 个 cn.s8c32g（ecs.i3g.2xlarge 8C/32G）
3. 第 1 个 CN 启动，连接 DN 执行恢复操作 → OOM
4. Pod 状态变为 ContainerStatusUnknown
5. CNClaim 无法绑定 → operator 认为需要更多 CN
6. operator 创建新 CN Pod → 同样 OOM
7. 循环重复，16 分钟内堆积 235+ 个僵尸 Pod
```

---

## 资源瓶颈

| 维度 | cn.s8c32g 节点 (ecs.i3g.2xlarge) | CN Pod 需求 |
|------|------|------|
| CPU 可分配 | 7.8C | 最大 6C/Pod |
| 内存可分配 | 30.5Gi | 最大 30Gi/Pod |
| 可容纳 CN 数 | **1 个** | 需要 4 个 |

一个节点只能放 1 个 CN Pod，但需要 4 个。autoscaler 应该扩更多节点，但扩容速度跟不上 operator 创建 Pod 的速度。

---

## 解决建议

| 方案 | 说明 |
|------|------|
| 减少 cnSets replicas | `replicas: 1`, `minReplicas: 1` 降低初始需求 |
| 减少 maxIdle | `maxIdle: 0` 不预留空闲 CN |
| 限制 cnSet maxReplicas | 防止 operator 无限补偿 |
| 等节点扩容完成再创建集群 | 避免 Pod 和节点扩容竞争 |

最小化配置（测试环境）：

```yaml
cnSets:
- name: shared
  profile: cn.s8c32g
  replicas: 1
  scalingConfig:
    minReplicas: 1
    maxReplicas: 3
    policy: AutoScaling

cnPools:
- name: s2
  profile: cn.s8c32g
  poolStrategy:
    scaleStrategy:
      maxIdle: 0
```

这样 replicas = 1 (claim) + 0 (idle) = **1**，只需 1 个节点。
