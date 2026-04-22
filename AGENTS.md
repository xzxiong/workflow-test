# AGENTS.md

## 项目概述

workflow-test 是一个 GitHub Actions Workflow 仓库，用于 MatrixOne Cloud (MOC) 的 CI/CD 自动化，涵盖镜像构建、集群升级、备份恢复、BVT 测试、数据迁移等运维场景。

仓库地址: `xzxiong/workflow-test`

## 目录结构

```
.github/workflows/    # 12 个 GitHub Actions Workflow（核心）
docs/                 # 运维文档（FAQ、handbooks 等）
third/mocloud-tester/ # mocloud-tester git submodule（仅供参考/分析，CI 中通过 checkout 拉取）
third/matrixone-operator/ # matrixone-operator git submodule（仅供参考/分析）
third/uni-agent/       # uni-agent git submodule（仅供参考/分析）
third/gitops/          # gitops git submodule（仅供参考/分析）
migrate-2.0.py        # MO-2.0 迁移脚本（依赖 mocloud-tester）
run.sh                # 备份辅助脚本
test.py               # 测试辅助脚本
load-data.sh          # 数据加载脚本
```

## Workflow 清单

| 文件 | 名称 | 用途 |
|------|------|------|
| `build-images.yaml` | Build and Push Images | 构建并推送 Docker 镜像 |
| `build-migrate.yaml` | Build mo-migrate | 构建 mo-migrate 工具 |
| `bvt_test.yaml` | MO BVT Tests | MO BVT 测试 |
| `load-data.yaml` | Load Data | 数据加载 |
| `main.yml` | Study and Test | 学习与测试 |
| `migrate.yaml` | Migrate MO-2.0 | MO-2.0 数据迁移 |
| `moc_upgrade_test.yaml` | MOC Upgrade Test | MOC 升级测试 |
| `restore-mo-data.yaml` | Restore MO Data | 恢复 MO 数据（S3 数据拷贝） |
| `run_mo_tests.yaml` | Run MO Tests | 运行 MO 测试 |
| `start-standalone-mo.yaml` | aliyun mo cluster incremental backup | 阿里云 MO 集群增量备份 |
| `test.yaml` | To Test | 通用测试 |
| `upgrade-mo-cluster.yaml` | Backup/Restore and Upgrade MO Cluster | 备份/恢复与升级 MO 集群 |

所有 workflow 均通过 `workflow_dispatch` 手动触发。

## 关键外部依赖

- **mocloud-tester** (`matrixorigin/mocloud-tester`): Python 测试框架，workflow 中 checkout 后使用，提供 K8s 操作、集群管理、备份恢复等能力。checkout 使用 `main` 分支。
- **Python 3.11**: mocloud-tester 运行环境。
- **阿里云 OSS**: 备份/恢复的 S3 存储，endpoint 配置在 `mocloud-tester/src/upgrade/upgrade-config.yaml`。
- **K8s 集群**: 通过 `K8S_UNIT_CONFIG` 和 `K8S_CONTROLLER_CONFIG` secrets 连接 Unit/Controller 集群。

## 分支约定

- `main`: 主分支，所有正式变更合入此分支
- `dev`: 开发分支
- `allure-reports` / `upgrade-reports`: 测试报告分支

## 编码约定

### Workflow 编写

- YAML 格式，2 空格缩进
- Secrets 通过 GitHub Repository Secrets 管理，不硬编码
- 使用 `actions/checkout@v4`、`actions/setup-python@v6` 等官方 action
- 长流程需设置合理的 `timeout-minutes`

### 脚本

- Shell 脚本使用 `set -e` 严格模式
- Python 脚本依赖 mocloud-tester 的模块结构（`src/upgrade/`、`src/cloud/`、`src/tests/`）

## 文档

- `docs/restore-mo-data-faq.md`: restore-mo-data workflow 的 FAQ，包含参数说明、排障指南
- `README.md`: 项目说明与 Release Notes

## 版本管理

- 使用 git tag 管理版本
- `base` tag: 克隆回来的基线快照
- 版本号遵循 semver（如 `1.0.0`）
- Release Notes 维护在 `README.md` 中
