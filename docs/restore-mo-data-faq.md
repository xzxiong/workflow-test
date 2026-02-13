# Restore MO Data 常见问题说明

本文档总结与 `restore --only-data` 及 `.github/workflows/restore-mo-data.yaml` 相关的配置、参数与行为。

---

## 1. restore --only-data 做了什么

**作用**：只执行「把备份数据从备份 S3 拷贝到恢复 S3」这一步，不创建/恢复集群、不创建实例、不恢复账号。

### 会执行的步骤

- 使用配置中的集群版本、备份/恢复的 bucket 与路径。
- 若指定了恢复集群版本且配置了备份/恢复 S3：
  - 创建恢复任务所在 namespace（若不存在）。
  - 通过 `should_restore_data()` 检查 S3 认证等条件。
  - 调用 **restore_mo_data()**：
    - 在 K8s 中启动一个 **restore Pod**（使用 `BACKUP_POD_TEMPLATE`，action=restore）。
    - 该 Pod 将数据从 `backup_bucket/backup_filePath` 拷贝到 `restore_bucket/restore_filePath`。
    - 等待该 Pod 成功结束（失败时有重试逻辑，会换路径如 `-retry` 再试）。

### 会跳过的步骤（因 `--only-data` 直接 return）

- 不调用 `restore_cluster()`：不创建 root cluster 资源、不基于恢复出的 S3 数据拉起 MO 集群。
- 不恢复 sys、ob-sys、stats、ob 等额外集群。
- 不等待集群 Pod、不做 Global Check、不重启 local-system-service/metric-worker、不更新 CU/Storage。
- 不创建实例、不跑 MO 测试、不恢复 MO 账号。

### 小结

| 项目                         | 会做 | 不会做 |
|------------------------------|------|--------|
| 备份 → 恢复 S3 的数据拷贝   | ✅   |        |
| 创建/恢复 root 集群并起 MO   |      | ✅     |
| 恢复 sys/ob-sys、建实例等   |      | ✅     |

---

## 2. restore-mo-data.yaml 里哪些参数有用

针对 **restore --only-data** 的流程，参数用途如下。

### 对 restore --only-data 有用的参数

| 类型     | 参数/环境变量 | 说明 |
|----------|----------------|------|
| **Inputs** | backup_s3_bucket | 备份所在 S3 bucket |
| | backup_s3_bucket_file_path | 备份在 bucket 内的路径 |
| | backup_id | 要恢复的备份 ID，未设置会报错退出 |
| | restore_cluster | 集群名，参与配置与恢复路径 |
| | restore_cluster_version | 恢复使用的集群版本，决定是否执行恢复数据分支 |
| | restore_s3_bucket | 数据恢复到的 S3 bucket |
| | restore_s3_bucket_file_path | 恢复路径；不设则默认 `{cluster_name}-{timestamp}` |
| **Step env** | BACKUP_ALIYUN_AK / BACKUP_ALIYUN_SK | 读备份桶的 AK/SK |
| | RESTORE_ALIYUN_AK / RESTORE_ALIYUN_SK | 写恢复桶的 AK/SK |
| | 上述 BACKUP_* / RESTORE_* 等 | 与 config 中备份/恢复配置对应 |
| **Job env** | env | 写入 config 的 cluster.env |
| | k8s_unit_config / k8s_controller_config | 连接 Unit/Controller 的 kubeconfig，创建 namespace 与 restore Pod 必需 |

### 对 restore --only-data 无用的参数

以下在「只恢复数据」路径中不会被使用（不跑测试、不建实例、不连管理平台）：

- normal_email_1、normal_password_1
- database_platform_password
- management_platform_url / management_platform_user / management_platform_password

已注释的 RESTORE_ROOT_USERNAME/PASSWORD、MO_IMAGE_REPOSITORY、MOC_PLUGIN_IMAGE 在 --only-data 下也不会用到。

### 注意

`restore_s3_bucket_file_path` 在 workflow 中未给 default；若 inputs 与 vars 都未设置，会传空，代码会使用 `cluster_name + timestamp` 作为恢复路径。

---

## 3. k8s_unit_config 和 k8s_controller_config 在哪里使用

### 传入方式

在 `restore-mo-data.yaml` 的 job `env` 中：

```yaml
k8s_unit_config: ${{ secrets.K8S_UNIT_CONFIG }}
k8s_controller_config: ${{ secrets.K8S_CONTROLLER_CONFIG }}
```

环境变量名为 **k8s_unit_config**、**k8s_controller_config**（小写、下划线）。

### 使用位置与流程

1. **入口**：`main.py` 中  
   `init_k8s_client('controller')`、`init_k8s_client('unit')`  
   分别使用 k8s_controller_config、k8s_unit_config。

2. **读取**：`src/tests/conftest.py` 的 `init_k8s_client(name)` 中：
   - `env.get_config(f'k8s_{name}_config')`  
     即 `k8s_controller_config` 或 `k8s_unit_config`。

3. **值的来源**：`Environment.get_config(key)` 先读当前环境配置文件（如 config.qa.json）；若配置中无或为空，则用 **环境变量** `os.environ.get(key)`。CI 中多为 kubeconfig 内容。

4. **使用方式**：若 `k8s_config` 不是已有文件路径，则会把内容写到 `src/cloud/kube-configs/config.controller` 或 `config.unit`，再以该路径创建 `K8sClient(name, config_path)`。

### 小结

- **使用位置**：`src/tests/conftest.py` 的 `init_k8s_client(name)`，通过 `Environment.get_config('k8s_controller_config'/'k8s_unit_config')` 读取。
- **作用**：提供连接 Controller 集群与 Unit 集群的 kubeconfig；restore 时用这两个 client 创建 namespace、创建并等待 restore Pod 等。

---

## 4. 连接 Controller 集群会执行什么操作

### restore --only-data 且未加 --force 时

- **list_cluster**（至少 1～2 次）：判断要恢复的集群名是否存在、是否为 root、是否 Active。
- **不**对 Cluster 做 create、delete、patch。

### restore --force 且集群已存在时

会执行 **clear_cluster**，对 Controller 做：

- list_cluster、**delete_cluster**（先删实例集群再删 root）、**patch_cluster**（清 finalizer）、**read_cluster**（期望 404）。

### 其它子命令（backup / upgrade / clear / 完整 restore 等）

会对 Controller 做 list_cluster、read_cluster、create_cluster、patch_cluster、delete_cluster 等，均为对 MatrixOne Cloud Cluster CRD 的增删改查。

---

## 5. RESTORE_CLUSTER_VERSION 的作用

### 来源

- 环境变量：`RESTORE_CLUSTER_VERSION`
- 配置写入：`cfgs['moBackup']['restore']['cluster']['version']`
- 命令行 `restore --cluster-version xxx` 也会覆盖该配置。

### 作用

1. **决定是否执行「恢复数据」分支（关键）**  
   - **有**设置：才会进入 create_ns、should_restore_data、restore_mo_data；对 restore --only-data 即只做到这里后 return。  
   - **未**设置：不会执行 restore_mo_data，只会打 warning 并断言集群已存在（read_cluster 200）。

2. **作为「恢复出来的集群」的版本（完整 restore 时）**  
   - 代码中：`self.configs['cluster']['version']['from'] = self.cluster_version`。  
   - 完整恢复时会用该版本渲染 Cluster spec（如 root-cluster-template），即恢复后的集群的 MO 版本。

3. **不决定恢复 Pod 的镜像**  
   - 恢复 Pod 使用 `moBackup.image.repository` + `moBackup.image.tag`（来自 upgrade-config.yaml），模板中未用 cluster version 拼镜像。

### 小结

| 场景              | RESTORE_CLUSTER_VERSION 的作用 |
|-------------------|--------------------------------|
| restore --only-data | 必须设置，用于**打开恢复数据分支**；不参与恢复 Pod 镜像。 |
| 完整 restore      | 同上，并作为**恢复出的集群的 MO 版本**。 |

---

## 6. backup_endpoint 在哪里设置

此处「backup endpoint」指备份使用的 S3/OSS endpoint（如 `moBackup.backup.endpoint`）。

### 当前唯一生效来源：配置文件

**文件**：`mocloud-tester/src/upgrade/upgrade-config.yaml`

- 备份 endpoint：`moBackup.backup.endpoint`（约第 38 行）  
  当前值：`http://oss-cn-hangzhou-internal.aliyuncs.com`
- 恢复 endpoint：`moBackup.restore.endpoint`（约第 54 行），值同上。

**没有**通过环境变量或命令行覆盖；`get_envs()`/`get_configs()` 中无对 backup/restore endpoint 的覆盖逻辑。

### 命令行参数（已注释）

`main.py` 中曾有：

- `--endpoint`（backup）、`--backup-endpoint`、`--restore-endpoint`  
  对应 `backup_endpoint` / `restore_endpoint`，均已注释，**不会生效**。

### 使用位置

- restore 时检查恢复路径是否为空：使用 `moBackup.restore.endpoint`。
- backup 时检查桶路径是否为空：使用 `moBackup.backup.endpoint`。
- 备份/恢复 Pod 的 args：模板 `backup-pod-template.yaml.j2` 使用 `moBackup.backup.endpoint` 与 `moBackup.restore.endpoint`（模板内带默认值）。

### 小结

| 名称           | 在哪里设置 | 说明 |
|----------------|------------|------|
| 备份 endpoint  | 仅 `upgrade-config.yaml` 的 `moBackup.backup.endpoint` | 无 env/CLI 覆盖 |
| 恢复 endpoint  | 同一文件的 `moBackup.restore.endpoint` | 同上 |
| backup_endpoint（CLI 变量名） | 无 | 对应参数已注释 |

要修改备份/恢复的 endpoint，需改 `upgrade-config.yaml`；若需通过环境变量或命令行设置，需在代码中增加对应读取与覆盖逻辑并（如需要）恢复相关 CLI 参数。

---

## 7. 使用的 OSS Access Key 不是 workflow 里传的 AK（InvalidAccessKeyId）

### 现象

报错 `InvalidAccessKeyId`，且错误信息里的 `OSSAccessKeyId` 既不是 `BACKUP_ALIYUN_AK` 也不是 `RESTORE_ALIYUN_AK`。

### 原因

流程里会先把环境变量写入配置，再在 **K8s 命名空间**里创建/更新 Secret `aliyun-moc-backup-test`，之后 **一律从该 Secret 读 AK/SK** 去调 OSS（如 `s3_bucket_path_is_empty`）。

`create_or_patch_secret()` 默认 **update_if_exists=False**：若该 Secret **已存在**（例如之前跑过 restore/backup 或别人创建过），则 **不会用本次的 BACKUP_ALIYUN_AK/RESTORE_ALIYUN_AK 覆盖**，直接跳过。后面读到的就是集群里**旧 Secret 的 AK**，所以会出现“用的 AK 不是我传的”的情况。

### 修复（代码已改）

在 `restore.py` 和 `backup.py` 里调用 `create_or_patch_secret(..., update_if_exists=True)`，这样每次都会用当前 workflow 传入的 AK/SK 更新 Secret，保证用的是你配置的 `BACKUP_ALIYUN_AK` / `RESTORE_ALIYUN_AK`。

### 临时处理（不改代码时）

- 在 Unit 集群的对应 namespace（restore 用的 `RESTORE_JOB_RUNNING_NAMESPACE`）下删除 Secret：  
  `kubectl delete secret aliyun-moc-backup-test -n <namespace>`  
  下次运行会重新创建 Secret，并使用本次传入的 AK。
- 或确保 workflow 的 `BACKUP_ALIYUN_AK` / `RESTORE_ALIYUN_AK` 与集群里该 Secret 当前内容一致。

---

## 参考文件

- `.github/workflows/restore-mo-data.yaml` — restore 数据 workflow
- `mocloud-tester/src/upgrade/main.py` — 入口、get_envs、get_configs、start_to_restore
- `mocloud-tester/src/upgrade/restore.py` — MORestore、restore_mo_data、restore_cluster
- `mocloud-tester/src/upgrade/upgrade-config.yaml` — 备份/恢复 endpoint、版本等配置
- `mocloud-tester/src/tests/conftest.py` — init_k8s_client
- `mocloud-tester/src/fixture/environment.py` — Environment.get_config
- `mocloud-tester/src/data/moc/templates/backup-pod-template.yaml.j2` — 备份/恢复 Pod 模板
