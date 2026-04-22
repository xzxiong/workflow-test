# Restore MO Data #7 运行记录（dev→dev 首次成功）

**Run URL**: https://github.com/xzxiong/workflow-test/actions/runs/24761070833  
**触发方式**: workflow_dispatch  
**触发人**: xzxiong  
**触发时间**: 2026-04-22 05:00 UTC (13:00 CST)  
**分支**: `dev-2-dev` (commit `169db0e`)  
**状态**: ✅ Success  
**总耗时**: 27m 57s

---

## 场景

**dev→dev**：从 dev 环境备份恢复到 dev 环境。

此前 Run #6 因 K8s Secret AK 不匹配导致 403 失败（详见 run-5 handbook FAQ #7）。本次通过 `ALIYUN_SECRET_NAME=aliyun-moc-backup-dev` 使用独立 Secret 解决。

---

## 输入参数

| 参数 | 值 |
|------|-----|
| backup_env | `dev` |
| env | `dev` |
| ALIYUN_SECRET_NAME | `aliyun-moc-backup-dev` |
| BACKUP_S3_BUCKET | `moc-backup-dev` |
| BACKUP_S3_BUCKET_FILE_PATH | `backup-260414-104849` |
| BACKUP_ID | `019d8b9b-b490-7168-b58d-e71373089697` |
| RESTORE_CLUSTER | `only-restore-data` |
| RESTORE_CLUSTER_VERSION | `nightly-9d616832a` |
| RESTORE_S3_BUCKET | `moc-backup-test` |
| RESTORE_S3_BUCKET_FILE_PATH | `restore-dev2dev-20260422050052`（自动生成） |

---

## 执行步骤

| # | Step | 状态 | 说明 |
|---|------|------|------|
| 1 | Set up job | ✅ | |
| 2 | Checkout | ✅ | mocloud-tester `xzxiong` 分支 |
| 3 | Set Up Python3.11 | ✅ | |
| 4 | Install Python Dependencies | ✅ | |
| 5 | Set Restore File Path | ✅ | 生成 `restore-dev2dev-20260422050052` |
| 6 | Restore MOC Data (from prod) | ⏭️ 跳过 | backup_env=dev |
| 7 | **Restore MOC Data (from dev)** | ✅ | 耗时 **27m 7s** |
| 8 | Restore MOC Data Again (from prod) | ⏭️ 跳过 | |
| 9 | Upload Logs | ✅ | |

---

## 关键日志

```
05:01:11 - Start to restore data:
  /mo_br restore
    --base_id 019d8b9b-b490-7168-b58d-e71373089697
    --backup_bucket moc-backup-dev
    --backup_filepath backup-260414-104849
    --restore_bucket moc-backup-test
    --restore_filepath restore-dev2dev-20260422050052
    --parallelism 150

05:28:18 - MO data are restored into
  moc-backup-test/restore-dev2dev-20260422050052
  from moc-backup-dev/backup-260414-104849

05:28:19 - Set github outputs:
  restore-bucket=moc-backup-test,
  restore-bucket-paths=restore-dev2dev-20260422050052
```

---

## 与 Run #5 (prod→dev) 对比

| 维度 | Run #5 (prod→dev) | Run #7 (dev→dev) |
|------|------|------|
| 备份源 | `mo-backup-20240201` (prod) | `moc-backup-dev` (dev) |
| 数据拷贝耗时 | 1h 28m | **27m** |
| Secret | `aliyun-moc-backup-test` | `aliyun-moc-backup-dev` |
| 恢复路径 | `restore-dev-20260213235245` | `restore-dev2dev-20260422050052` |
| mocloud-tester 分支 | `main` | `xzxiong` |

dev→dev 耗时显著短于 prod→dev（27m vs 1.5h），因为 dev 备份数据量较小。

---

## 验证的修复

- ✅ `ALIYUN_SECRET_NAME=aliyun-moc-backup-dev` 独立 Secret 方案有效，避免与 prod Secret 冲突
- ✅ 自动生成路径 `restore-dev2dev-*` 格式正确
- ✅ mocloud-tester `xzxiong` 分支的 Secret name 可配置功能正常
