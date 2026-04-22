# Restore MO Data #5 运行记录

**Run URL**: https://github.com/xzxiong/workflow-test/actions/runs/21993275300  
**触发方式**: workflow_dispatch（手动触发）  
**触发人**: xzxiong  
**触发时间**: 2026-02-13 15:53 UTC (23:53 CST)  
**分支**: `upgrade-py` (commit `5eacea2`)  
**状态**: ✅ Success  
**总耗时**: 1h 29m 7s

---

## 运行环境

| 项目 | 值 |
|------|-----|
| Runner | ubuntu-latest (Ubuntu 24.04.3 LTS) |
| Runner Image | ubuntu-24.04 (20260209.23.1) |
| Runner Version | 2.331.0 |
| Azure Region | westcentralus |
| Python | CPython 3.11.14 |
| Git | 2.52.0 |
| mocloud-tester commit | `140c841b6d36a4787197c566247ce367126b94d7` (main) |

---

## 输入参数

| 参数 | 值 | 说明 |
|------|-----|------|
| env | `dev` | 目标环境 |
| provider | `aliyun` | 云厂商 |
| region | `cn-hangzhou` | 区域 |
| BACKUP_S3_BUCKET | `mo-backup-20240201` | 备份源 bucket |
| BACKUP_S3_BUCKET_FILE_PATH | `backup260128-2100` | 备份源路径 |
| BACKUP_ID | `019c04dd-7253-7c21-befa-d971c8aa86d7` | 备份 ID |
| RESTORE_CLUSTER | `only-restore-data` | 恢复集群名 |
| RESTORE_CLUSTER_VERSION | `nightly-9d616832a` | 恢复集群版本 |
| RESTORE_S3_BUCKET | `moc-backup-test` | 恢复目标 bucket |
| RESTORE_S3_BUCKET_FILE_PATH | `restore-dev-20260213235245` | 恢复目标路径 |

---

## 执行步骤

| # | Step | 状态 | 说明 |
|---|------|------|------|
| 1 | Set up job | ✅ | 准备 runner 环境 |
| 2 | Checkout | ✅ | checkout `matrixorigin/mocloud-tester` main 分支 |
| 3 | Set Up Python3.11 | ✅ | 安装 CPython 3.11.14 |
| 4 | Install Python Dependencies | ✅ | pip install -r requirements.txt（~20s） |
| 5 | **Restore MOC Data** | ✅ | 核心步骤，耗时 **1h 28m 32s** |
| 6 | Restore MOC Data Again | ⏭️ 跳过 | 首次成功，未触发重试 |
| 7 | Upload Logs | ✅ | 上传 upgrade-test-logs (2.17 KB, 已过期) |
| 8 | Post cleanup | ✅ | 清理 git credentials |

---

## 核心步骤分析：Restore MOC Data

### 执行命令

```bash
python src/upgrade/main.py restore --only-data
```

### 关键日志

```
15:54:12 - INFO - Start to restore data with command of mo-backup as:
  /mo_br restore
    --base_id 019c04dd-7253-7c21-befa-d971c8aa86d7
    --backup_dir s3
    --backup_endpoint http://oss-cn-hangzhou-internal.aliyuncs.com
    --backup_bucket mo-backup-20240201
    --backup_filepath backup260128-2100
    --backup_region cn-hangzhou
    --backup_access_key_id $(BACKUP_ACCESS_KEY_ID)
    --backup_secret_access_key $(BACKUP_SECRET_ACCESS_KEY)
    --restore_dir s3
    --restore_endpoint http://oss-cn-hangzhou-internal.aliyuncs.com
    --restore_access_key_id $(RESTORE_ACCESS_KEY_ID)
    --restore_secret_access_key $(RESTORE_SECRET_ACCESS_KEY)
    --restore_bucket moc-backup-test
    --restore_filepath restore-dev-20260213235245
    --restore_region cn-hangzhou
    --log_level debug
    --parallelism 150

17:22:31 - INFO - MO data are restored into moc-backup-test/restore-dev-20260213235245
                   from mo-backup-20240201/backup260128-2100

17:22:32 - INFO - Set github outputs:
                   restore-bucket=moc-backup-test,
                   restore-bucket-paths=restore-dev-20260213235245.
```

### 时间线

| 时间 (UTC) | 事件 |
|------------|------|
| 15:53:33 | Job 启动，准备 runner |
| 15:53:35 | Checkout mocloud-tester |
| 15:53:36 | 安装 Python 3.11.14 |
| 15:53:39 ~ 15:53:59 | pip install 依赖（~20s） |
| 15:54:12 | 开始 S3 数据拷贝（mo_br restore） |
| 17:22:31 | 数据拷贝完成 |
| 17:22:32 | 输出 github outputs |
| 17:22:33 | 上传日志 artifact |

### 性能数据

- **数据拷贝耗时**: 1h 28m 19s（15:54:12 → 17:22:31）
- **并行度**: 150
- **源**: `mo-backup-20240201/backup260128-2100`（阿里云 OSS 内网）
- **目标**: `moc-backup-test/restore-dev-20260213235245`（阿里云 OSS 内网）
- **Endpoint**: `http://oss-cn-hangzhou-internal.aliyuncs.com`（内网传输）

---

## Workflow 设计要点

### 重试机制

Workflow 内置了一次自动重试：
- 第一次 `Restore MOC Data` 设置了 `continue-on-error: true`
- 若失败，`Restore MOC Data Again` 通过 `if: steps.restore.outcome == 'failure'` 触发
- 本次首次即成功，重试步骤被跳过

### 超时设置

- `timeout-minutes: 720`（12 小时），对于 ~1.5h 的数据拷贝任务留有充足余量

### Artifact

- 上传了 `upgrade-test-logs`（2.17 KB），retention 7 天，已过期

---

## 总结

本次运行为 `restore --only-data` 模式，仅执行 S3 到 S3 的数据拷贝，不创建/恢复集群。使用 v1.0.0 版本的 workflow（Python 3.11 + mocloud-tester main 分支），一次成功，耗时约 1.5 小时，parallelism=150，走阿里云 OSS 内网传输。
