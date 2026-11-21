# 质量门禁与硬化红线

任一硬门槛不达标不发布。门禁要能在本地与 CI 同时复现，不允许只存在于文档。

## 1. 硬化红线

### 1.1 已落地

1. `import-linter` 分层契约：依赖单向，禁止循环。配置写在生成项目 `pyproject.toml` 的 `[tool.importlinter]`（两层契约：`app -> features -> platform`、`platform` 禁止依赖 `features`），不依赖独立的 `importlinter.ini`。
2. AST 架构测试（`tests/architecture/test_layers.py`）：
   - 除 `platform/ui/qt.py` 外，任何文件禁止直接导入 `PySide6` / `PySide2`
   - `platform/*` 禁止导入 `features/*` 与 `app/*`
   - `features/*` 禁止导入 `app/*`
3. 覆盖率门槛：模板默认 `fail_under = 75`，核心模块实测 90%+；omit 入口胶水（`__main__`、`app/bootstrap|cli|container|window`）、`devtools`、`selfcheck`、`ui/qt.py`、`ports/*`。
4. `pre-commit`：ruff check（`--fix`）、ruff format、trailing-whitespace、end-of-file-fixer、check-yaml、check-toml、check-added-large-files（512KB）、mypy（`src`）。
5. 安全扫描：CI `security` 段执行 `uv run pip-audit`。
6. CI 三段（`lint-type` / `test` / `security`），`enable_mysql` 时 test 段带 MySQL 8 service，`include_packaging` 时额外带 `build` 段。
7. 打包冒烟：启用 `include_packaging` 时，`scripts/build.py` 构建后执行产物 `--self-test`，退出码 0 才算通过。
8. 模型与迁移一致：所有 ORM 模型必须在迁移中有对应表（`tests/integration/test_migrate.py` 覆盖 upgrade / downgrade）。

### 1.2 规划中

9. 更多 AST 约定扫描：
   - `features/*/view.py` 禁止 `#RRGGBB`、`setStyleSheet`
   - `QMessageBox` 仅限 `platform/ui/kit/dialogs.py`
   - `QThread` / `QRunnable` 仅限 `platform/ui/kit/tasks.py`
   - `app.exec()` 仅限 `app/bootstrap.py`（当前靠 code review 与唯一入口约定）
   - 根目录 `main.py` 仅允许转发到 `<pkg>.__main__`，禁止 import `features/*`、`platform/*`、`app/*`
   - 禁用 `QRegExp`、`QDesktopWidget`、`QFontMetrics.width`、`exec_()`
10. 元 CI：模板仓库自身渲染 5 组参数（minimal / standard / full / Qt5 降级 / 关闭可选物）-> 生成项目 -> 跑完整检查。
11. bandit / gitleaks。
12. 切片边界契约：`features/A` 只能 import `features/B.api`（当前只有 `platform` 契约 + AST 测试）。
13. 重复代码率门禁：jscpd 阈值 3%，超限失败。
14. 复用资产覆盖率：`platform/core`、`platform/adapters`、`platform/ui/kit` 单独统计，门槛 80%。
15. 复用率度量：审查报告统计平台复用率与自制代码占比。
16. 数据落库契约：AST 扫描禁止用 `QSettings` / JSON 文件存业务数据。
17. `alembic check` 接入 CI（当前只有 upgrade/downgrade 集成测试）。

## 2. 代码规范与门禁映射

| 条款 | 门禁 | 状态 |
|---|---|---|
| 命名与结构 | ruff N、import-linter | 已接入 |
| 类型与文档 | mypy strict、ruff ANN/D | 已接入；interrogate 未接入 CI |
| 错误与日志 | ruff B/S、AST 约定扫描 | 部分（AST 仅覆盖绑定与分层） |
| 并发与 UI | AST 约定扫描、pytest-qt | pytest-qt 已接入 |
| 数据与迁移 | Alembic upgrade/downgrade 集成测试 | 已接入；`alembic check` 未接入 |
| 测试 | pytest、覆盖率门槛 | 已接入 |
| Git | commitlint | 规划 |

## 3. 量化门槛

| 质量属性 | 指标 | 门槛 |
|---|---|---|
| 可靠性 | 会话崩溃率 | ≤ 0.5% |
| 可靠性 | 数据丢失事件 | 0 |
| 安全性 | high / critical 漏洞 | 0 |
| 性能 | 冷启动 | ≤ 2.5s（P95） |
| 性能 | idle 内存 | ≤ 250MB |
| 可维护性 | 圈复杂度 | ≤ 10 |
| 可维护性 | 单文件行数 | ≤ 300 |
| 可维护性 | 重复代码率 | ≤ 3% |
| 可测试性 | 核心覆盖率 | ≥ 85% |
| 可测试性 | 整体覆盖率 | ≥ 75% |
| 可复用性 | 平台复用率 | ≥ 60% |
| 可复用性 | 新功能复用率 | ≥ 80% |
| 可观测性 | 结构化日志 + trace + crash dump | 关键路径 100% |
| 兼容性 | Windows 10 / 11 + 高 DPI | CI 矩阵通过 |

## 4. CI 分段

```text
lint-type  ->  test  ->  security  ->  build（仅 include_packaging=true）
```

实际内容：

| 段 | 步骤 |
|---|---|
| `lint-type` | `uv sync --frozen` -> `ruff check .` -> `mypy src` -> `lint-imports` |
| `test` | `pytest -q --cov --cov-report=xml`；`enable_mysql` 时附带 MySQL 8 service 与 `DATABASE_URL` |
| `security` | `pip-audit` |
| `build`（可选） | `python scripts/build.py` -> 上传 `dist/<App>` artifact |

- 未启用打包时不生成 `build` 段，CI 为三段，不留空任务。
- 段间不共享可变工作区，产物用 artifact 传递。

## 5. 基线与回归（规划）

- `baseline.json` 记录：冷启动、idle 内存、覆盖率、复杂度、重复率、复用率。
- 任一指标劣化超过 5% 直接失败，不允许人工放行。
- 基线随版本更新，更新必须附原因与对比数据。

## 6. 复用层级与度量

| 层级 | 形式 | 范围 | 落地 | 状态 |
|---|---|---|---|---|
| L1 | snippets | 单文件片段 | `snippets/` | 规划 |
| L2 | platform 模块 | 项目内跨切片 | `platform/` | 已实现 |
| L3 | feature 切片 | 业务能力 | `features/` + `scripts/new_feature.py` | 已实现 |
| L4 | 内部 wheel 包 | 跨项目 | `packages/` | 规划 |
| L5 | 模板 profiles | 项目规模 | `profile` 参数 | 保留位 |
| L6 | 示例 gallery | 复用范本 | `examples/` | 规划 |

当前新项目一律生成 standard 形态（平台层 + 设置切片 + 全部门禁），`profile` 尚未改变产物。

复用率统计口径：

- 平台复用率 = 切片中调用平台 API 的行数 / 切片总有效行数。
- 新功能复用率 = 新切片中复用资产行数 / 新切片总有效行数。
- 统计进 `reports/health.md`，按月对比。

## 7. 发布门槛（Go / No-Go）

- [ ] 所有硬化红线通过
- [ ] 量化门槛全部达标
- [ ] 崩溃上报与自动更新在真机验证
- [ ] 备份与回滚方案可用
- [ ] 遥测与隐私合规（用户同意）
- [ ] 安装包签名与 sha256（仅启用打包时）
- [ ] 发布说明、已知问题、值班人
- [ ] 回滚演练记录

任一项未勾选，发布即为 No-Go，不允许「先发后补」。

## 8. 持续优化机制

- 基线：`baseline.json` 记录启动、内存、覆盖率、复杂度、复用率。
- 回归门禁：劣化超过 5% 失败。
- 定期评审：月度健康报告，季度模板评审。
- 技术债：`docs/dev/tech-debt.md` 登记，按影响 / 成本排序偿还。
- 弃用策略：至少保留一个次版本，提供迁移指南。
- 兼容矩阵：Python / PySide6 / SQLAlchemy / 打包工具，CI 覆盖当前与上一版本。

## 9. 质量责任

- 生成即达标：模板默认配置就是生产配置，不靠事后修补。
- 规范即门禁：每条规范都有对应检查。
- 度量即决策：优化优先级由 health 报告与技术债清单驱动。
