# workflow-test

Study github workflow and test.

---

# Release Notes

## v1.0.0

**发布日期**: 2026-02-13
**Tag**: `1.0.0` (commit `5eacea2`)
**Base**: `base` tag (commit `5709c97`, 2025-12-19) — 克隆回来的基线
**仓库**: xzxiong/workflow-test

---

### 概述

`base` tag 为从原始仓库克隆回来的基线快照（449 commits, 2024-01-26 ~ 2025-12-19）。`1.0.0` 在此基线上新增 2 个 commit，完成 CI 环境升级和运维文档沉淀。

---

### 基线内容 (base tag)

基线包含 12 个 GitHub Actions Workflow，覆盖以下能力：

| Workflow | 用途 |
|----------|------|
| `build-images.yaml` | 构建并推送 Docker 镜像 |
| `build-migrate.yaml` | 构建 mo-migrate 工具 |
| `bvt_test.yaml` | MO BVT 测试 |
| `load-data.yaml` | 数据加载 |
| `main.yml` | 学习与测试 (Study and Test) |
| `migrate.yaml` | MO-2.0 迁移 |
| `moc_upgrade_test.yaml` | MOC 升级测试 |
| `restore-mo-data.yaml` | 恢复 MO 数据 |
| `run_mo_tests.yaml` | 运行 MO 测试 |
| `start-standalone-mo.yaml` | 阿里云 MO 集群增量备份 |
| `test.yaml` | 通用测试 |
| `upgrade-mo-cluster.yaml` | 备份/恢复与升级 MO 集群 |

基线演进历程中的关键里程碑：

- **项目初始化** (2024-01) — 初始化 moc-tester 项目，GitHub Pages 测试
- **镜像构建** (2024-02 ~ 2024-03) — mo-backup 镜像构建，registry 配置，支持从 input 选择构建不同镜像
- **Workflow 升级** (2024-03) — 升级 workflows，支持配置集群镜像仓库，升级后创建实例测试
- **问题修复** (2024-05 ~ 2024-07) — 多轮 issue 修复，新增测试用例
- **TPCC 支持** (2024-11) — 支持选择 TPCC 数据规模
- **插件升级** (2024-12) — 升级 plugin
- **超时调整** (2025-12) — reset timeout（base tag 最终 commit）

---

### v1.0.0 新增变更 (base → 1.0.0)

#### CI 环境升级

**commit**: `4ab2f28` — chore: update python to 3.11

- `restore-mo-data.yaml` 中 mocloud-tester 的 checkout 分支从 `qa-txzhou` 切回 `main`
- `actions/setup-python` 从 v5 升级到 v6
- Python 版本从 3.8 升级到 3.11（Python 3.8 已于 2024-10 EOL）

#### 运维文档

**commit**: `5eacea2` — chore: restore-mo-data handbook

新增 `docs/restore-mo-data-faq.md`（231 行），涵盖 7 个 FAQ：

1. `restore --only-data` 的执行范围（只做 S3 数据拷贝，不建集群）
2. workflow 参数有效性分析（区分 `--only-data` 下有用/无用参数）
3. `k8s_unit_config` / `k8s_controller_config` 的传入与使用流程
4. 连接 Controller 集群的操作范围
5. `RESTORE_CLUSTER_VERSION` 的作用与必要性
6. `backup_endpoint` 的配置位置（仅 `upgrade-config.yaml`，CLI 参数已注释）
7. `InvalidAccessKeyId` 问题排查（Secret 不更新导致用了旧 AK）

---

### 文件变更统计

```
 .github/workflows/restore-mo-data.yaml |   8 +-
 docs/restore-mo-data-faq.md            | 231 +++
 2 files changed, 235 insertions(+), 4 deletions(-)
```
