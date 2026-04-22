# Review: restore-mo-cluster.yaml 能力差异分析

**日期**: 2026-04-22  
**对比对象**:
- `restore-mo-cluster.yaml` (本仓库)
- `third/mocloud-tester/.github/workflows/moc_upgrade_test.yaml` (mocloud-tester 原版)

---

## 1. 命令差异

| 维度 | restore-mo-cluster.yaml | moc_upgrade_test.yaml |
|------|:---:|:---:|
| 基础命令 | `restore --force` | `restore --all` + 动态拼接参数 |
| `--all` | ❌ **缺失** | ✅ |
| `--force` | ✅ 硬编码 | ✅ 通过 `vars.RESTORE_FORCE` 控制 |
| `--create-instance` | ❌ | ❌ |
| `--disable-mo-tests` | ❌ | ✅ 通过 `vars.RUN_MO_TESTS` 控制 |
| `--disable-auto-upgrade` | ❌ | ✅ 通过 `vars.AUTO_UPGRADE` 控制 |
| `--restore-account` | ❌ | ✅ 通过 `vars.RESTORE_MO_ACCOUNT` 控制 |

---

## 2. `--all` 缺失的影响（P0）

`--all` 控制 `restore.py` 中 `start(restore_all=True)` 的行为。

### 带 `--all`（mocloud-tester 版）

恢复 root cluster 后，额外创建 4 个集群 CRD：

1. `sys` — 系统集群（共享 CN）
2. `ob-sys` — OB 系统集群（CN Pod）
3. `stats` — 统计集群
4. `ob` — OB 集群

然后删除 `mo-ob` namespace 下 `app=ob-service` 的 Pod 触发重建。

### 不带 `--all`（当前）

不创建上述 4 个集群，但会：
- 将 `ob-sys-cn` replicas 设为 0，`s2-pool-cn` replicas 设为 3
- 仍然执行：等待 Pod Running → Global Check → 重启 local-system-service → 更新 CU/Storage

**结论**：不带 `--all` 的集群可启动但功能不完整，缺少 ob-sys 等组件。

---

## 3. 环境变量差异

| 环境变量 | restore-mo-cluster | moc_upgrade_test | 用途 |
|----------|:---:|:---:|------|
| `BACKUP_ALIYUN_AK/SK` | ✅ | ✅ | 读备份桶凭证 |
| `RESTORE_ALIYUN_AK/SK` | ✅ | ✅ | 写恢复桶凭证 |
| `BACKUP_S3_BUCKET` | ✅ | ✅ | 备份桶名 |
| `BACKUP_S3_BUCKET_FILE_PATH` | ✅ | ✅ | 备份路径 |
| `BACKUP_ID` | ✅ | ✅ | 备份 ID |
| `RESTORE_CLUSTER` | ✅ | ✅ | 集群名 |
| `RESTORE_CLUSTER_VERSION` | ✅ | ✅ | 集群版本 |
| `RESTORE_S3_BUCKET` | ✅ | ✅ | 恢复桶名 |
| `RESTORE_S3_BUCKET_FILE_PATH` | ✅ | ✅ | 恢复路径 |
| `MO_IMAGE_REPOSITORY` | ✅ | ✅ | MO 镜像仓库 |
| `RESTORE_MOC_PLUGIN_TAG` | ✅ | ✅ | plugin 版本 |
| **`INSTANCE_PASSWORD`** | ❌ | ✅ | 创建实例密码，`--create-instance` / `--restore-account` 时需要 |
| **`UPGRADE_STRATEGY`** | ❌ | ✅ | 升级策略 JSON，写入集群 CRD spec |

---

## 4. Runner 差异

| 维度 | restore-mo-cluster | moc_upgrade_test |
|------|------|------|
| Runner | `ubuntu-latest` (GitHub hosted) | `amd64-mo-guangzhou-medium8` (自托管，广州) |
| Python | `actions/setup-python@v6` 3.11 | venv + 阿里云 pip 镜像 |
| 网络 | 公网，通过 kubeconfig 连 K8s | 内网，直连 K8s 和 OSS |

**风险**：`restore_cluster()` 需要持续 watch K8s API（创建 CRD → 等 namespace → 等集群 Active → 等 Pod Running），耗时 30min+。GitHub hosted runner 走公网，网络延迟和稳定性不如自托管 runner。需确认 kubeconfig 支持公网访问。

---

## 5. Outputs 差异

| Output | restore-mo-cluster | moc_upgrade_test | 用途 |
|--------|:---:|:---:|------|
| `mo-host` | ❌ | ✅ | 实例连接地址，供下游 load-data/tpcc 使用 |
| `mo-port` | ❌ | ✅ | 实例端口 |
| `mo-usr` | ❌ | ✅ | 实例用户名 |
| `mo-pass` | ❌ | ✅ | 实例密码 |
| `restore-bucket-paths` | ❌ | ✅ | 恢复路径，供 clear job 清理 S3 数据 |

当前无下游 job 串联，暂不影响。未来扩展时需补上。

---

## 6. 代码流程对比

`restore --force`（不带 `--all`、不带 `--only-data`）的执行路径：

```
start_to_restore()
  ├─ 检查集群是否存在
  │   └─ --force: 删除已有集群 + 清理 S3 数据
  └─ MORestore.start(restore_all=False)
      ├─ create_ns()                          ✅ 两者都有
      ├─ should_restore_data()                ✅ 数据已存在则跳过
      ├─ restore_mo_data()                    ⏭️ 跳过（数据已拷贝）
      ├─ restore_cluster()                    ✅ 创建 root cluster CRD
      ├─ _restore_cluster(sys/ob-sys/...)     ❌ 需要 --all
      ├─ wait pods running                    ✅
      ├─ make_global_check()                  ✅
      ├─ restart local-system-service         ✅
      ├─ update_cu_and_storage_time()         ✅
      ├─ create instances                     ❌ 需要 --create-instance
      └─ restore_mo_account()                 ❌ 需要 --restore-account
```

---

## 7. 改进建议

| 优先级 | 项目 | 建议 |
|:---:|------|------|
| **P0** | 补 `--all` | 命令改为 `restore --force --all`，否则集群功能不完整 |
| P1 | `INSTANCE_PASSWORD` | 新增 secret，当需要 `--create-instance` 或 `--restore-account` 时必须 |
| P1 | `UPGRADE_STRATEGY` | 新增 variable，影响集群 CRD 的升级策略字段 |
| P1 | `--force` 可配置 | 改为 input boolean 参数，避免误删已有集群 |
| P2 | outputs | 补充 `mo-host/port/usr/pass` 和 `restore-bucket-paths` 输出 |
| P2 | Runner | 如 GitHub hosted runner 网络不稳定，考虑切自托管 runner |
| P2 | `--disable-mo-tests` | 当前未传，默认 `run_mo_tests=True`，若无测试环境会报错 |
