# 生命周期：生成 / 审查 / 归档

三层与生命周期的对应：生成属于沉淀层产出，审查属于硬化层，归档属于回流层。

> 实现状态：生成链路（探测 / 渲染 / git / 记录 / 冒烟）已落地；审查目前只跑自动门禁（ruff / mypy / pytest / lint-imports / pip-audit），`reports/review` 与人工清单未生成；归档、回流、`docs/dev`、`_migrations` 仍为规划。下文保留目标形态，作为后续实现依据。

## 1. 生成

入口 `scripts/scaffold.py`，五步顺序固定：

1. 探测 Python 版本（uv > 当前解释器 > `py -0p`，回退 3.12），同时决定 Qt 绑定。
2. 参数校验：包名合法、目标目录为空或仅含 `.git`。
3. 调 `copier.run_copy` 渲染。
4. `git init` + 首次提交（`--no-git` 跳过）。
5. 写生成记录；`.venv` 存在时跑 `main.py --self-test` 冒烟。

产物：

- 项目目录
- `.copier-answers.yml`
- `reports/generation/<timestamp>.json`：answers、模板名、探测到的 Python、目标路径、UTC 时间

失败即回滚：冒烟失败时删除目标目录并打印 stdout/stderr。

## 2. 审查

### 2.1 自动门禁

生成后立即执行，也可在 CI 重跑：

- lint / 类型 / 单测 / 集成 / 架构红线 / 覆盖率 / 安全扫描
- 输出 `reports/review/<timestamp>.md`，汇总通过项与阻断项

### 2.2 人工审查

`docs/dev/review-checklist.md` 覆盖五项：

1. 架构红线
2. UI 规范
3. 数据迁移
4. 安全
5. 文档

要求：

- PR 模板强制勾选。
- 新项目首个 PR 必须完成一次初始化审查并留下签署记录。
- 模板变更审查：影响面分析（哪些生成物受影响）-> `copier update --pretend` 试跑 -> 模板 PR 审查。

## 3. 归档

仓库内 `archive/`，配 `archive/INDEX.md` 索引：

| 对象 | 路径 | 内容 |
|---|---|---|
| 生成记录 | `archive/generations/YYYY-MM-DD-<project>.json` | 参数、模板版本、探测版本、操作人、commit |
| 构建制品 | `archive/builds/<project>/<version>/`（仅启用打包时） | 安装包、SBOM、sha256、构建日志 |
| 模板版本 | `archive/templates/vX.Y.Z/` | tag、changelog、冻结的 answers schema |
| 项目退役 | `archive/retired/<project>/` | 冻结清单、数据处置、密钥吊销记录 |

规则：

- 归档文件只增不改，写入即计算 sha256 并登记到索引。
- 退役流程：只读冻结 -> 导出必要数据 -> 吊销密钥与更新通道 -> 标记 retired。

## 4. 回流层

- `scripts/feedback/report.py`：汇总 junit、覆盖率、ruff/mypy 告警数，产出 `reports/health.md`。
- 漂移检测：生成项目执行 `copier update --pretend`，报告偏离模板的文件与原因。
- 度量驱动改进：
  - 弹窗触发频次高 -> 改为内联提示
  - 任务耗时分布 -> 调整池大小
  - 平台复用率下降 -> 排查切片内重复实现
  - 重复代码率上升 -> 抽取到 `platform/` 或 `snippets/`
  - `on_dispose` 未调用告警 -> 生命周期泄漏
  - 反复手工改动 -> 升级为 snippet / recipe
  - 反复踩坑 -> 升级为 lint 规则或架构测试
- 文档资产：ADR、recipes、retro 模板、「模板改进」issue 模板。
- 变更日志：`git-cliff` 按 conventional commits 生成。

## 5. 开发文档

mkdocs-material 站点，`mike` 多版本，源文件在仓库。

```text
docs/
├─ index.md
├─ dev/
│  ├─ template.md          # 模板开发指南：目录、参数、条件文件、_migrations
│  ├─ code-style.md        # 代码规范正文与门禁映射
│  ├─ generate.md
│  ├─ review.md
│  ├─ archive.md
│  ├─ release.md
│  ├─ tech-debt.md
│  └─ testing.md
├─ architecture/
│  ├─ overview.md
│  ├─ ui-runtime.md
│  └─ adr/
├─ recipes/
├─ user/
│  ├─ install.md
│  ├─ configure.md
│  └─ upgrade.md
└─ api/                    # mkdocstrings 自动生成
```

- 文档即门禁：`mkdocs build --strict` 进 CI，死链或缺失引用直接失败。
- 每个公开模块必须有 docstring，`mkdocstrings` 自动出 API 参考。
- 文档版本与模板 tag 同步，`mike deploy` 发布。

## 6. 模板升级路径

- 模板变更按语义化版本发布，`_migrations/` 存放升级脚本。
- 生成项目通过 `copier update` 拉取，冲突只允许出现在模板保护区之外。
- 升级前先 `--pretend` 出漂移报告，人工确认后再执行。
- 破坏性变更必须提供迁移指南，并在 `CHANGELOG.md` 标注。
