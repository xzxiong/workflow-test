# Restore Workflow 能力对比

## 三个 Workflow 定位

| Workflow | 定位 | 命令 |
|----------|------|------|
| `moc_upgrade_test.yaml` | mocloud-tester 原版，一体化流水线 | `restore --all --force` |
| `restore-mo-data.yaml` | 步骤 1：S3 数据拷贝 | `restore --only-data` |
| `restore-mo-cluster.yaml` | 步骤 2：基于已拷贝数据创建集群 | `restore --force` |

---

## 能力矩阵

| 能力 | moc_upgrade_test | restore-mo-data | restore-mo-cluster |
|------|:---:|:---:|:---:|
| S3 数据拷贝 | ✅ | ✅ | ❌ |
| 创建 root cluster | ✅ | ❌ | ✅ |
| 恢复 sys/ob-sys（`--all`） | ✅ | ❌ | ❌ |
| 等 Pod Running + Global Check | ✅ | ❌ | ✅ |
| 重启 local-system-service | ✅ | ❌ | ✅ |
| 更新 CU/Storage | ✅ | ❌ | ✅ |
| 创建实例 | ✅ | ❌ | ❌ |
| 恢复 MO 账号 | ✅ | ❌ | ❌ |
| 升级集群 | ✅ | ❌ | ❌ |
| 加载测试数据 | ✅ | ❌ | ❌ |
| 跑 TPCC/TPCH 测试 | ✅ | ❌ | ❌ |
| 清理集群 + S3 | ✅ | ❌ | ❌ |
| 支持 dev→dev | ❌ | ✅ | ✅ |
| 独立 Secret 隔离 | ❌ | ✅ | - |

---

## 使用流程

### 场景 1：完整恢复（拷数据 + 建集群）

```
restore-mo-data (拷数据)
    ↓ 输出 restore-bucket-paths
restore-mo-cluster (建集群)
    ↓ 填入 restore_s3_bucket_file_path
集群就绪
```

### 场景 2：只拷数据（不建集群）

```
restore-mo-data
    ↓
数据在 S3，后续手动处理
```

### 场景 3：数据已存在，只建集群

```
restore-mo-cluster
    ↓ 填入已有的 restore_s3_bucket_file_path
集群就绪
```

---

## 参数对比

### restore-mo-data.yaml

| Input | 必填 | 说明 |
|-------|:---:|------|
| backup_env | ✅ | 备份源环境（prod/dev） |
| env | ✅ | 恢复目标环境（dev/qa） |
| backup_s3_bucket | 否 | 不填从 vars 读 |
| backup_s3_bucket_file_path | 否 | 不填从 vars 读 |
| backup_id | 否 | 不填从 vars 读 |
| restore_cluster | 否 | 默认 only-restore-data |
| restore_cluster_version | 否 | 默认 nightly-9d616832a |
| restore_s3_bucket | 否 | 默认 moc-backup-test |
| restore_s3_bucket_file_path | 否 | 不填自动生成 |

### restore-mo-cluster.yaml

| Input | 必填 | 说明 |
|-------|:---:|------|
| env | ✅ | 目标环境 |
| restore_s3_bucket | ✅ | 恢复数据所在 bucket |
| restore_s3_bucket_file_path | ✅ | 恢复数据路径（restore-mo-data 的输出） |
| restore_cluster | ✅ | 集群名 |
| restore_cluster_version | ✅ | MO 版本 |

---

## 与 moc_upgrade_test 的差距

当前两个 workflow 组合后仍缺少的能力：

| 缺失能力 | 影响 | 补齐方式 |
|----------|------|----------|
| `--all`（恢复 sys/ob-sys） | 集群功能不完整 | restore-mo-cluster 加 `--all` 参数 |
| 创建实例 | 无法通过实例访问 MO | 新增 workflow 或加 `--create-instance` |
| 恢复 MO 账号 | 租户账号丢失 | 加 `--restore-account` 参数 |
| 升级集群 | 无法升级到新版本 | 使用 moc_upgrade_test 的 upgrade job |
| 加载测试数据 + 跑测试 | 无自动化测试 | 使用 moc_upgrade_test 的 load-data/test jobs |
| 清理集群 | 需手动清理 | 新增 clear workflow |

---

## 设计思路

把 moc_upgrade_test 的一体化流程拆成独立步骤，优势：

- **可单独重跑**：数据拷贝失败不影响已有集群，建集群失败不需要重新拷数据
- **灵活组合**：可以只拷数据、只建集群、或两步串联
- **环境隔离**：dev→dev 用独立 Secret，不与 prod→dev 冲突
- **参数简化**：restore-mo-cluster 只需 4 个必填参数
