# Restore MO Cluster 配置详解

## 概述

`restore-mo-cluster.yaml` workflow 基于已拷贝到 S3 的备份数据，创建一个恢复集群。核心逻辑在 `mocloud-tester/src/upgrade/restore.py` 的 `MORestore.restore_cluster()` 方法中，通过渲染 `root-cluster-template.yaml.j2` 模板生成 Cluster CRD 并提交到 K8s Controller。

---

## Restore 场景下的针对性配置

### 1. `objectStorage.restored: true`

**最关键的标志位**，告诉 operator 该集群从备份恢复，而非全新创建。

```python
# restore.py:272
self.configs['cluster']['objectStorage']['restored'] = True
```

模板渲染结果：

```yaml
spec:
  managed:
    objectStorage:
      path: <bucket>/<restore-path>
      region: cn-hangzhou
      restored: true    # <-- restore 专有
```

**作用**：operator 检测到 `restored: true` 后，会跳过初始化流程（如 bootstrap SQL），直接从 S3 路径加载已有数据启动集群。

---

### 2. `objectStorage.path` 指向恢复数据路径

正常新建集群时 `objectStorage.path` 为空或自动生成；restore 时指向备份数据已拷贝到的目标路径。

```python
# restore.py:39-44
self.configs['cluster']['objectStorage']['bucket'] = restore_bucket   # e.g. moc-backup-test
self.configs['cluster']['objectStorage']['path'] = restore_filePath   # e.g. restore-dev2dev-20260422050052
```

模板渲染：

```yaml
spec:
  managed:
    objectStorage:
      path: moc-backup-test/restore-dev2dev-20260422050052
```

---

### 3. 跳过 `initJobs`（条件渲染）

模板中 initJobs 被 `{% if not cluster.objectStorage.restored %}` 包裹：

```yaml
{% if not cluster.objectStorage.restored %}
  initJobs:
  - jobType: sql
    name: loaddata-create-tables
    sqls: [...]
  - jobType: sql
    name: loaddata-load-nation
    sqls: [...]
  # ... TPCH SF1/SF10 数据加载 + Publication 创建
{% endif %}
```

**restore 时不执行 initJobs**，因为：
- 数据已在 S3 备份中，无需重新建表和 load data
- publication 等元数据也在备份里

---

### 4. Cluster Labels

```yaml
metadata:
  annotations:
    matrixone.cloud/enable-aksk-authorize: "true"
  labels:
    matrixone.cloud/role: root
    matrixone.cloud/cordoned: "Y"    # <-- 防止调度器分配新租户
```

`cordoned: "Y"` 表示集群处于隔离状态，不接受新租户分配。restore 集群通常用于测试/验证，不应被平台自动分配流量。

---

## Restore 后的额外操作

集群创建成功（Active）后，`MORestore.start()` 还会执行以下针对性操作：

| 步骤 | 代码位置 | 说明 |
|------|----------|------|
| 调整 Pod 副本数 | `restore.py:117-118` | `ob-sys-cn` 设为 0，`s2-pool-cn` 设为 3 |
| 等待所有 Pod Running | `restore.py:119` | 按 `ROOT_CLUSTER_INIT_PODS` 定义校验 |
| 触发 Global Checkpoint | `restore.py:121` | `GCKP.*-End` 确保恢复数据一致性 |
| 重启 local-system-service | `restore.py:123` | 刷新集群内部服务状态 |
| 更新 CU/Storage 统计时间 | `restore.py:125` | 停 metric-worker → 修改计费起算时间 → 重启 |

### Pod 副本预期

| 组件 | 副本数 | Label Selector |
|------|--------|---------------|
| LogSet | 3 | `matrixorigin.io/component=LogSet` |
| DNSet | 1 | `matrixorigin.io/component=DNSet` |
| local-service | 1 | `matrixone.cloud/component=local-service` |
| ProxySet | 2 | `matrixorigin.io/component=ProxySet` |
| shared CN | 2 | `matrixorigin.io/component=CNSet,matrixorigin.io/claimset=<cluster>-shared` |
| limited CN | 0 | `matrixorigin.io/component=CNSet,matrixorigin.io/claimset=<cluster>-limited` |
| s2-pool CN | 3* | `matrixorigin.io/component=CNSet,pool.matrixorigin.io/pool-name=<cluster>-s2` |
| ob-sys CN | 0* | `matrixorigin.io/component=CNSet,matrixorigin.io/claimset=<cluster>-ob-sys-ob-sys` |

> *标注的为 restore 模式（非 `--all`）下的调整值，原始模板中 s2-pool 为 6、ob-sys 为 1。

---

## 与新建集群的配置差异对比

| 配置项 | 新建集群 | Restore 集群 |
|--------|----------|-------------|
| `objectStorage.restored` | `false` | `true` |
| `objectStorage.path` | 空/自动生成 | 备份数据所在路径 |
| `initJobs` | 执行（建 TPCH 表 + load data） | 跳过 |
| `cordoned` label | `"Y"` | `"Y"` |
| Global Checkpoint | 不需要 | 必须执行 |
| CU/Storage 时间更新 | 不需要 | 必须执行 |

---

## 执行流程图

```
restore-mo-cluster workflow
    │
    ├─ 1. Checkout mocloud-tester (main)
    ├─ 2. Setup Python 3.11 + deps
    └─ 3. python src/upgrade/main.py restore --force
           │
           ├─ 读取 upgrade-config.yaml + 环境变量覆盖
           ├─ 设置 objectStorage.restored = true
           ├─ 设置 objectStorage.path = <restore_s3_bucket_file_path>
           ├─ 渲染 root-cluster-template.yaml.j2（无 initJobs）
           ├─ 提交 Cluster CRD 到 Controller
           ├─ 等待 Namespace Active
           ├─ 等待 Cluster Active (30min timeout)
           ├─ 等待所有 Pods Running (15min timeout)
           ├─ 触发 Global Checkpoint
           ├─ 重启 local-system-service
           ├─ 更新 CU/Storage 统计时间
           └─ 完成
```

---

## 环境变量

| 环境变量 | 来源 | 用途 |
|----------|------|------|
| `RESTORE_CLUSTER` | workflow input | 集群名（如 `restore-2dev`） |
| `RESTORE_CLUSTER_VERSION` | workflow input | MO 镜像版本 |
| `RESTORE_S3_BUCKET` | workflow input | 恢复数据所在 bucket |
| `RESTORE_S3_BUCKET_FILE_PATH` | workflow input | 恢复数据 S3 路径 |
| `RESTORE_ALIYUN_AK` | secret | 阿里云 AccessKeyId（访问 restore bucket） |
| `RESTORE_ALIYUN_SK` | secret | 阿里云 SecretAccessKey |
| `MO_IMAGE_REPOSITORY` | vars | MO 镜像仓库地址 |
| `RESTORE_MOC_PLUGIN_TAG` | vars | MOC plugin 镜像 tag |

---

## 已知问题与注意事项

1. **DN Pod 调度**：`dn.c4m16` profile 需要 4C/16G 节点，若节点资源不足会导致 Pending 超时（参见 [run-1 记录](restore-mo-cluster-run-1.md)）
2. **超时限制**：集群 Active 超时 30 分钟，Pods Running 超时 15 分钟，Global Checkpoint 超时 30 分钟
3. **重试机制**：数据恢复失败时会在 S3 path 后追加 `-retry` 后缀重试一次
4. **前置依赖**：需要先执行 `restore-mo-data` workflow 完成 S3 数据拷贝，获取输出路径后填入本 workflow
