# DN Pod 调度失败分析：restore-2dev 集群

**日期**: 2026-04-22  
**集群**: restore-2dev (dev 环境)  
**问题**: DN Pod 持续 Pending，无法调度

---

## 完整链路：Profile → NodeGroup → ASG → 节点 → Pod 调度

```
Provider CRD (aliyun)
  └─ profiles: 定义 profile 名 → 机型 → resources
       dn.c4m16 → ecs.g7.xlarge (4C/16G) → limits: 3C/14Gi

Cluster CRD (restore-2dev)
  └─ dnSets.profile: dn.c4m16

unit-agent: makeDnSetNodeGroup()
  └─ GetProfileByName("dn.c4m16", provider)
  └─ 生成 NodeGroup: dn_restore-2dev_dn_c4m16
       └─ ScalingGroup: instanceType=ecs.g7.xlarge, autoScaling=true
  └─ 调用阿里云 API 创建 ASG → 扩出节点
  └─ 给节点打标签: cluster-pool=restore-2dev, profile=dn.c4m16

matrixone-operator:
  └─ 从 Cluster CRD 读取 dnSets.managed.nodeSelector
  └─ 注入 Pod: nodeSelector={cluster-pool=restore-2dev, profile=dn.c4m16}
  └─ 注入 Pod: tolerations=[cluster-pool, profile]

scheduler:
  └─ 匹配节点 → 检查资源 → 调度/失败
```

---

## Provider CRD 中的 DN Profile 定义

```bash
kubectl get provider aliyun -o yaml  # on controller cluster
```

| Profile | 机型 | 节点总资源 | 可分配 | Pod limits |
|---------|------|-----------|--------|------------|
| `dn.standard` | `ecs.g7.4xlarge` | 16C/64G | ~15C/60G | 15C/58Gi |
| `dn.small` | `ecs.r7.xlarge` | 4C/32G | ~3.8C/30G | 3C/28Gi |
| `dn.c4m16` | `ecs.g7.xlarge` | 4C/16G | ~3.8C/14.3G | 3C/14Gi |

---

## 根因

`dn.c4m16` 的 Pod limits (14Gi) 与节点可分配内存 (14.3Gi) 之间只有 ~300Mi 余量。系统 Pod 占用 ~441Mi 后，剩余 ~13.9Gi < 14Gi，**差约 100Mi 导致调度失败**。

```
节点 cn-hangzhou.10.3.95.48:
  Capacity:    4C / 15.1Gi
  Allocatable: 3.8C / 14.3Gi
  已用:        395m / 441Mi
  剩余:        3.4C / ~13.9Gi
  DN 需要:     3C / 14Gi  ← 内存不够
```

注意：**不是没有 scaling group**。unit-agent 已经正确创建了 ASG 并扩出了 `ecs.g7.xlarge` 节点，也打上了正确的标签和 taint。问题纯粹是机型内存太小。

对比 freetier-01 的 DN 使用 `dn.small` (ecs.r7.xlarge 4C/32G)，内存充裕。

---

## 节点实际状态

```
$ kubectl get nodes -l cluster-pool=restore-2dev
NAME                     PROFILE         STATUS
cn-hangzhou.10.1.19.33   log.standard    Ready
cn-hangzhou.10.2.214.26  log.standard    Ready
cn-hangzhou.10.3.95.43   log.standard    Ready
cn-hangzhou.10.3.95.46   proxy.standard  Ready
cn-hangzhou.10.3.95.47   proxy.standard  Ready
cn-hangzhou.10.3.95.48   dn.c4m16        Ready  ← DN 节点已扩出，但内存不够
cn-hangzhou.10.x.x.x     cn.s8c32g       Ready
```

---

## 解决方案

### 方案 A：改模板 DN profile 为 dn.small（推荐）

修改 `mocloud-tester/src/data/moc/templates/root-cluster-template.yaml.j2`：

```yaml
# 改前
  dnSets:
    - ...
      profile: dn.c4m16

# 改后
  dnSets:
    - ...
      profile: dn.small
```

- `dn.small` → `ecs.r7.xlarge` (4C/32G)，内存充裕
- 与 freetier-01 一致
- 影响范围：仅新建集群

### 方案 B：改 Provider CRD 中 dn.c4m16 的机型

```bash
kubectl edit provider aliyun  # on controller cluster
```

把 `dn.c4m16` 的 instanceTypes 从 `ecs.g7.xlarge` (4C/16G) 改为 `ecs.g7.2xlarge` (8C/32G)。

- 影响所有使用 `dn.c4m16` 的集群
- 成本增加（机型更大）
- 不推荐

### 方案 C：改 dn.c4m16 的 resources limits

把 limits 从 14Gi 降到 12Gi（与 requests 一致），留出更多余量。

- 需要改 Provider CRD
- 可能影响 DN 性能
- 不推荐

---

## 相关资源

- Provider CRD: `kubectl get provider aliyun -o yaml` (controller cluster)
- Unit CRD: `kubectl get unit unit-cn-hangzhou -o yaml` (controller cluster)
- unit-agent 扩容逻辑: `third/unit-agent/pkg/extension/providers/aliyun/profile.go` → `makeDnSetNodeGroup()`
- Profile 查找: `third/unit-agent/pkg/extension/providers/aliyun/alicomm/utils.go` → `GetProfileByName()`
- operator nodeSelector 注入: `third/matrixone-operator/pkg/controllers/mocluster/controller.go` → `setPodSetDefault()`
