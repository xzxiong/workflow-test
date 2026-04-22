# LogSet Failover 恢复手册：还原被 Reserve 的 Log Ordinal

## 背景

matrixone-operator 在检测到 LogSet Pod 持续故障（默认 10 分钟）后，会自动执行 failover：将故障 Pod 的 ordinal 加入 Kruise StatefulSet 的 `reserveOrdinals`，跳过该序号并创建新序号的 Pod 替代。

例如：`default-log-2` 故障 → operator 将 2 加入 `reserveOrdinals` → Kruise 创建 `default-log-3` 替代。

**关键事实：operator 只会添加 `reserveOrdinals`，永远不会自动移除。** 恢复需要手动操作。

## Operator Failover 逻辑（源码分析）

源码位置：`pkg/controllers/logset/controller.go` Repair()

```
触发条件: Observe() 中 StoresFailedFor(storeFailureTimeout) 返回非空
         ↓
Repair():
  1. FailoverEnabled == false → 跳过
  2. len(FailedStores) > minorityLimit → 多数派故障，等人工介入
  3. len(ReserveOrdinals) >= minorityLimit → failover 次数已达上限，停止
  4. 取第一个故障 store，获取其 ordinal
  5. FailedPodStrategy == Orphan → 给 pod 加 finalizer + label
     FailedPodStrategy == Delete (默认) → 直接 reserve
  6. Upsert ordinal 到 sts.Spec.ReserveOrdinals
  7. 更新 gossip config
```

关键参数：
- `minorityLimit = LogShardReplicas / 2`（3 副本时 limit=1，即最多自动 failover 1 个）
- `storeFailureTimeout` 默认 10 分钟
- `FailedPodStrategy` 默认 Delete

## Kruise StatefulSet ReserveOrdinals 语义

`replicas` 表示期望的**可用 Pod 数量**，`reserveOrdinals` 中的序号被跳过：

| replicas | reserveOrdinals | 实际 Pod |
|----------|----------------|----------|
| 3 | [] | 0, 1, 2 |
| 3 | [2] | 0, 1, 3 |
| 4 | [2] | 0, 1, 3, 4 |

**扩 replicas 不会恢复被 reserve 的序号，只会在末尾追加新序号。**

## 诊断

### 1. 确认当前状态

```bash
# 查看 Kruise StatefulSet 的 reserveOrdinals
kubectl -n <ns> get statefulset.apps.kruise.io <name>-log \
  -o jsonpath='{.spec.reserveOrdinals}' && echo

# 查看 LogSet 状态
kubectl -n <ns> get logset.core.matrixorigin.io <name> \
  -o jsonpath='{.status}' | python3 -m json.tool

# 查看实际 Pod
kubectl -n <ns> get pods -l matrixorigin.io/component=LogSet
```

### 2. 确认 operator 日志

```bash
# 查看 failover 相关日志
kubectl -n mo-system logs <operator-pod> | grep -i "<ns>" | grep -iE "failover|reserve|repair|ordinal"
```

## 恢复操作

### 场景 A：让替代 Pod 正常工作（推荐）

如果替代 Pod（如 log-3）已经或即将就绪，**不需要恢复原 ordinal**。0/1/3 和 0/1/2 功能完全等价。

等待替代 Pod 就绪即可：
```bash
kubectl -n <ns> get pod <name>-log-3 -w
```

### 场景 B：恢复原 ordinal（移除 reserveOrdinals）

适用于：替代 Pod 无法启动、或有特殊原因需要恢复原序号。

#### 前提条件

- 原 ordinal 对应的节点/存储已恢复
- 当前没有其他 failover 正在进行

#### 步骤

**Step 1：移除 reserveOrdinals**

```bash
# 移除整个 reserveOrdinals 字段（适用于只有一个被 reserve 的情况）
kubectl -n <ns> patch statefulset.apps.kruise.io <name>-log \
  --type json -p '[{"op":"remove","path":"/spec/reserveOrdinals"}]'
```

如果有多个 reserve 只想移除特定的，需要先查看数组索引：
```bash
# 查看当前 reserveOrdinals
kubectl -n <ns> get statefulset.apps.kruise.io <name>-log \
  -o jsonpath='{.spec.reserveOrdinals}'
# 输出如 [2]，索引 0 对应 ordinal 2

# 移除索引 0 的元素
kubectl -n <ns> patch statefulset.apps.kruise.io <name>-log \
  --type json -p '[{"op":"remove","path":"/spec/reserveOrdinals/0"}]'
```

**效果**：Kruise 会自动缩掉多余的高序号 Pod（如 log-3），并创建被释放的序号 Pod（如 log-2）。

**Step 2：确认 Pod 调度和启动**

```bash
kubectl -n <ns> get pods -l matrixorigin.io/component=LogSet -w
```

等待新 Pod 变为 Running + Ready。

**Step 3：确认 operator 自动更新 gossip config**

operator 在下一次 reconcile 循环中会调用 `updateGossipConfig()`，自动将 gossip seeds 更新为正确的 Pod 列表。无需手动操作。

验证：
```bash
kubectl -n <ns> get cm <name>-log-gossip -o yaml
```

确认 `gossip-seed-addresses` 包含恢复后的 Pod 地址。

**Step 4：确认 LogSet 状态恢复**

```bash
kubectl -n <ns> get logset.core.matrixorigin.io <name> \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

等待 `Ready` condition 变为 `True`。

### 场景 C：Orphan 策略下的恢复

如果 `FailedPodStrategy` 为 `Orphan`，被故障的 Pod 会被加上 finalizer `matrixorigin.io/confirm-deletion` 和 label `matrixorigin.io/action-required=True`。

除了执行场景 B 的步骤外，还需要清理孤儿 Pod：

```bash
# 查找孤儿 Pod
kubectl -n <ns> get pods -l matrixorigin.io/action-required=True

# 移除 finalizer 让 Pod 被删除
kubectl -n <ns> patch pod <orphaned-pod> --type json \
  -p '[{"op":"remove","path":"/metadata/finalizers"}]'
```

## 为什么 patch StatefulSet 不会被 operator 覆盖？

分析 operator 代码中所有写 `reserveOrdinals` 的路径：

| 代码路径 | 是否写 reserveOrdinals |
|---------|----------------------|
| `Observe()` → `syncStatefulSetSpec()` | ❌ 只改 PVC retention policy |
| `Observe()` → `syncReplicas()` | ❌ 只改 replicas |
| `Observe()` → `Update()` | ❌ 只提交 Observe 中的 diff，不含 reserveOrdinals |
| `Scale()` | ❌ 只改 replicas + gossip |
| `Repair()` | ✅ Upsert ordinal 到 reserveOrdinals |

**只有 `Repair()` 会写 `reserveOrdinals`**，而 `Repair()` 只在 `StoresFailedFor(timeout)` 返回非空时触发。只要恢复后的 Pod 正常运行，`Repair()` 不会被触发，patch 不会被覆盖。

## 风险提示

1. **恢复期间的短暂不可用**：移除 reserveOrdinals 后，Kruise 会先缩掉高序号 Pod（如 log-3），再创建恢复的 Pod（如 log-2）。期间 LogSet 可能短暂只有 2 个副本。
2. **PVC 复用**：如果原 ordinal 的 PVC 仍存在（`PVCRetentionPolicy: Retain`），新 Pod 会复用旧 PVC 及其数据。如果 PVC 已删除（`PVCRetentionPolicy: Delete`），会创建新 PVC。
3. **再次 failover**：如果恢复后的 Pod 再次故障超过 `storeFailureTimeout`，operator 会再次将其 reserve。
