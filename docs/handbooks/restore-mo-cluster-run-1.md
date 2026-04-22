# Restore MO Cluster Run #1 运行记录

**Run URL**: https://github.com/xzxiong/workflow-test/actions/runs/24764485382  
**Job**: https://github.com/xzxiong/workflow-test/actions/runs/24764485382/job/72455404910  
**触发方式**: workflow_dispatch  
**触发时间**: 2026-04-22 06:49 UTC (14:49 CST)  
**分支**: `dev-2-dev`  
**状态**: ❌ Failed  
**总耗时**: 31m 27s

---

## 输入参数

| 参数 | 值 |
|------|-----|
| env | `dev` |
| RESTORE_CLUSTER | `restore-2dev` |
| RESTORE_CLUSTER_VERSION | `v3.0.0-a37aaf836-2026-04-21` |
| RESTORE_S3_BUCKET | `moc-backup-test` |
| RESTORE_S3_BUCKET_FILE_PATH | `restore-dev2dev-20260422050052` |
| MO_IMAGE_REPOSITORY | (vars) |
| RESTORE_MOC_PLUGIN_TAG | `0.11.0-17e89d6-2024-12-04` |

---

## 执行步骤

| # | Step | 状态 | 说明 |
|---|------|------|------|
| 1 | Set up job | ✅ | |
| 2 | Checkout | ✅ | mocloud-tester `xzxiong` 分支 |
| 3 | Set Up Python3.11 | ✅ | |
| 4 | Install Python Dependencies | ✅ | |
| 5 | **Restore MO Cluster** | ❌ | 超时 |
| 6 | Upload Logs | ✅ | |

---

## 失败原因

```
06:49:41 - Start to restore cluster with data in moc-backup-test/restore-dev2dev-20260422050052
07:20:26 - ERROR - Timeout to achieve the conditions by 'cluster_is_active'
Exception: Timeout to achieve the conditions by 'cluster_is_active' with args: (K8sClient, 'restore-2dev', False)
```

**集群创建成功但未能在 30 分钟内达到 Active 状态。**

### 根因链

1. **DN Pod 调度延迟**：`dn.c4m16` profile 对应 `ecs.g7.xlarge` (4C/16G)，节点可分配内存 14.3Gi，DN Pod limits 14Gi，系统 Pod 占用后剩余不足，导致 DN Pending ~30 分钟
2. **CN Pod 雪崩**：cnSets replicas=2 + cnPool maxIdle=1，节点不足时 CN 启动 OOM，operator 不断重建，堆积 235+ 僵尸 Pod
3. **集群无法 Active**：DN 未 Ready → 集群状态停留在 Creating → 超过 `CLUSTER_BECOMES_ACTIVE_TIMEOUT` (30min) → 抛异常

### 时间线

| 时间 (UTC) | 事件 |
|------------|------|
| 06:49:30 | Workflow 开始 |
| 06:49:41 | Cluster CRD 创建成功，开始等待 Active |
| 06:49:41 ~ 07:20:26 | 等待集群 Active（30 分钟） |
| 07:20:26 | 超时，抛出异常 |

---

## 已采取的修复措施

| 修复 | PR | 说明 |
|------|-----|------|
| dn.c4m16 limits 对齐 requests | [gitops#4530](https://github.com/matrixorigin/gitops/pull/4530) | limits cpu:2/mem:12Gi，适配 ecs.g7.xlarge |
| CN replicas 缩减 | [mocloud-tester#761](https://github.com/matrixorigin/mocloud-tester/pull/761) | replicas 2→1, minReplicas 2→1, maxIdle 1→0 |

---

## 后续

待上述 PR 合入后重新测试。预期：
- DN Pod 可在 ecs.g7.xlarge 上正常调度（limits 12Gi < 可分配 14.3Gi）
- CN 只需 1 个 Pod，不会雪崩
- 集群应在 10-15 分钟内达到 Active
