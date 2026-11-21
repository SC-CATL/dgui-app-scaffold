# 模板规格与实施路线

## 0. 实现状态（2026-11-17）

配套的 Copier 模板仓库 `py-scaffold` 已可生成并自证：

| 验证项 | 结果 |
|---|---|
| 渲染矩阵 `tests/test_render.py` | 16 项全绿（5 种参数组合） |
| 元 CI `.github/workflows/meta-ci.yml` | 渲染矩阵 + 标准组合全量检查（ruff / mypy / pytest / lint-imports / migrate / self-test / doctor） |
| 真实生成 PySide6 + Python 3.12 | `--self-test` ok；ruff 0；mypy 49 文件 0 错；pytest 66 项全绿，覆盖率 94.59% |
| 真实生成 PySide2 + Python 3.10 | `--self-test` ok（Qt 5.15.2）；ruff 0；mypy 0 错；pytest 66 项全绿，覆盖率 94.46% |
| `uv run lint-imports` | 2 契约 KEPT（配置在 `pyproject.toml` 的 `[tool.importlinter]`，不再有 `importlinter.ini`） |
| `uv run doctor` | 5 项全绿 |
| `uv run dev` / `scripts/dev.py --no-watch` | 离屏冒烟存活，迁移 + 窗口正常 |

尚未实现（参数已声明但暂为保留位）：`profile` 不改变产物、`enable_sync` / `include_auth` / `include_updater` 无分支产物；`include_docs`、`include_docker`、`include_packaging` 已生效。

## 1. 模板仓库结构

模板仓库采用 `_subdirectory: template`，生成期脚本放在仓库根 `scripts/`，不进入产物：

```text
py-scaffold/
├─ copier.yml                 # 参数、_exclude、_tasks
├─ pyproject.toml             # 仓库自身：pytest 渲染测试
├─ README.md / CHANGELOG.md / LICENSE / .gitignore
├─ .github/workflows/meta-ci.yml   # 渲染矩阵 + 标准组合全量检查
├─ scripts/
│  ├─ scaffold.py             # 探测 -> 渲染 -> git -> 冒烟 -> 记录
│  ├─ detect_python.py        # uv > 当前解释器 > py -0p
│  └─ detect_qt.py            # 32 位 / Win7-8 / abi3 wheel -> 绑定
├─ tests/test_render.py       # 渲染矩阵 + 残留占位符 + 语法编译
└─ template/                  # 仅此目录进入生成结果
   ├─ main.py                 # 薄层启动器（根目录）
   ├─ pyproject.toml          # 依赖、依赖组、ruff/mypy/coverage/importlinter
   ├─ alembic/                # env.py + script.py.mako + versions/0001_initial.py
   ├─ scripts/                # post_gen.py / dev.py / new_feature.py（可选 build.py）
   ├─ src/<pkg>/
   │  ├─ __main__.py          # --version / --self-test / --migrate / --mock / --dev
   │  ├─ app/                 # bootstrap / cli / container / window（唯一组合根）
   │  ├─ platform/            # config / core / ports / adapters / observability / devtools / ui
   │  ├─ features/settings/   # 七件套：__init__ / api / models / repository / service / view / viewmodel
   │  └─ resources/tokens.toml
   ├─ tests/                  # unit / ui / architecture / integration
   └─ .github/workflows/ci.yml
```

`copier.yml` 关键字段：

```yaml
_subdirectory: template
_templates_suffix: .jinja
_exclude:
  - "{% if not include_docs %}mkdocs.yml{% endif %}"
  - "{% if not include_packaging %}pyinstaller.spec{% endif %}"
  - "{% if not include_packaging %}scripts/build.py{% endif %}"
  - "{% if not include_docker %}docker-compose.yml{% endif %}"
_tasks:
  - "{{ _copier_python }} scripts/post_gen.py"
```

`post_gen.py` 检测到 `uv` 才执行 `uv sync`，失败只提示不阻断；`scaffold.py` 在 `.venv` 存在时再跑 `main.py --self-test`，失败删除半成品。

> 因为模板带 `_tasks`，直接 `copier copy` 必须加 `--trust`，否则 Copier 拒绝执行任务。

## 2. 参数

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `project_name` | str | `myapp` | 目录名，同时用于文档标题 |
| `package_name` | str | 由 `project_name` 派生 | 合法 Python 标识符，snake_case |
| `app_display_name` | str | `MyApp` | 窗口标题与安装包名 |
| `description` | str | 空 | 一句话描述，写入 pyproject 与 README |
| `author` | str | 空 | 写入 pyproject 与 LICENSE；空值不生成 authors 字段 |
| `license` | choice | `MIT` | 本期只支持 MIT |
| `python_version` | str | `3.12`（scaffold 探测后覆盖） | 覆盖探测值；PySide2 时钳制 3.10 |
| `qt_binding` | choice | `auto` | `auto` / `pyside6` / `pyside2`，规则见 `qt-binding.md` |
| `profile` | choice | `standard` | 保留位：当前不改变产物，门禁一律全开 |
| `enable_mysql` | bool | `false` | 加 `pymysql` 依赖、MySQL compose、CI 服务；集成测试仍走同一套 SQLAlchemy 代码 |
| `enable_sync` | bool | `false` | 保留位（P3 交付），当前无分支产物 |
| `include_auth` | bool | `true` | 保留位（P3 交付），当前无分支产物 |
| `include_updater` | bool | `true` | 保留位（P3 交付），当前无分支产物 |
| `include_crash_report` | bool | `true` | 控制崩溃捕获模块（当前随平台层生成） |
| `include_docs` | bool | `true` | 生成 `mkdocs.yml` 与 `docs/` |
| `include_docker` | bool | `false` | 生成 `docker-compose.yml`（MySQL） |
| `include_packaging` | bool | `false` | 生成 PyInstaller spec、`scripts/build.py`、打包 CI 与依赖；仅用户明确要求时开启 |

参数约束：

- `package_name` 必须匹配 `^[a-z][a-z0-9_]*$`，且不与标准库模块同名。
- `enable_mysql` 与 `include_docker` 独立：前者影响测试与 CI，后者影响交付物。
- 关闭 `include_docs` 时不会生成 `mkdocs.yml`，CI 不引用文档命令。

## 3. Profiles（保留位）

`profile` 参数已声明但**当前不改变任何产物**，仅作为后续门禁分组与预设的入口。实际生效的开关是 `include_docs` / `include_docker` / `include_packaging` / `enable_mysql`。

规划中的目标形态：

| 能力 | minimal | standard | full |
|---|---|---|---|
| 平台层（core/ports/adapters/ui-kit） | 是 | 是 | 是 |
| 业务切片七件套 | 否 | 是 | 是 |
| 生成器脚本 | 是 | 是 | 是 |
| DX（dev / doctor / 热重启） | 是 | 是 | 是 |
| 文档站 | 否 | 是 | 是 |
| 示例 gallery | 否 | 否 | 是 |
| 全部硬化门禁 | 是 | 是 | 是 |
| 打包（PyInstaller） | 否 | 否 | 否 |

门禁不随 profile 降级；打包一律默认关闭，只有 `include_packaging=true` 才生成。

## 4. Python 版本

- 生成时执行一次探测，值写入 `pyproject.toml` 的 `requires-python` 与 `.python-version`，之后构建与 CI 一律读锁定值。
- 探测顺序：`uv` 已安装的最高版本 -> 运行 Copier 的当前解释器 -> `py -0p`；全部失败回退 `3.12` 并输出警告。
- `--data python_version=3.13` 优先于探测值。
- 绑定降级时（`qt_binding=pyside2`）Python 钳制到 `min(detected, 3.10)`，因为 PySide2 无 3.11+ wheel；实测可用组合为 Python 3.10.22 + PySide2 5.15.2.1。
- PySide2 路径把 `pytest-qt` 固定为 `>=4.2,<4.3`：4.5 起上游移除了 PySide2 支持；PySide6 路径用 `>=4.4`。
- 同源派生：ruff `target-version`、mypy `python_version`、CI 矩阵均由该值生成；启用打包时 PyInstaller 解释器同源。
- `tests/test_render.py` 断言渲染结果中的版本字段与探测值一致。

禁止在各配置文件里手写版本号。

## 5. 渲染矩阵

`tests/test_render.py` 当前覆盖五组参数（共 16 个断言用例）：

| 组合 | 关键开关 | 断言 |
|---|---|---|
| minimal | `profile=minimal` | 关键入口、切片、迁移、令牌存在 |
| standard | `profile=standard` | 同上 |
| full | `profile=full` | 同上 |
| Qt5 降级 | `qt_binding=pyside2` + `python_version=3.10` | 依赖只有 `PySide2>=`，`requires-python=>=3.10` |
| 关闭可选物 | `include_docs=false` + `include_packaging=false` | 无 `mkdocs.yml`、无 spec、无 `build.py` |

每个渲染结果还统一断言：

- 文件树完整：`main.py`、`app/bootstrap.py`、`features/settings/api.py`、`alembic/versions/0001_initial.py`、`resources/tokens.toml`。
- 无残留 Jinja（扫描时跳过 `.venv` / `.git` / `reports` / 缓存目录）。
- 全部 `.py` 通过 `py_compile`。

额外用例：`include_packaging=true` 生成 spec / build.py / 打包依赖；`include_docker=true` 生成 compose。

生成器 `scripts/scaffold.py` 的默认链路：

```text
探测 Python / Qt -> copier.run_copy -> git init + 首次提交 -> 写 reports/generation/*.json
-> 若 .venv 存在：ENVIRONMENT=test QT_QPA_PLATFORM=offscreen python main.py --self-test
```

完整质量链路需手动执行（或由 CI 承担）：

```text
ruff check . -> mypy src -> pytest -q --cov -> uv run lint-imports -> uv run doctor
```

每个组合还统一断言 `main.py` 存在；根目录启动器的行为靠生成后的真实冒烟与 `tests/architecture/test_layers.py` 约束：

- 只 import 标准库与 `<pkg>.__main__`（AST 扫描）。
- 未 `uv sync` 时把 `src/` 插入 `sys.path` 兜底。
- 打包入口固定 `src/<pkg>/__main__.py`，不使用 `main.py`。

## 6. 实施路线

### P0 模板骨架（已完成主体）

- [x] `copier.yml`（参数 + `_subdirectory` + `_exclude` + `_tasks`）
- [x] 渲染矩阵测试（16 项）
- [x] meta CI（`meta-ci.yml`：渲染矩阵 + 标准组合全量检查）
- [x] README / CONTRIBUTING / CHANGELOG
- [x] Python 版本自动探测与一致性测试
- [x] Qt 绑定自动探测与锁定（`qt_binding`，PySide6 主选 / PySide2 降级）
- [x] 根目录 `main.py` 启动器（薄层转发 + `src/` 兜底 + 友好退出）
- [x] `scripts/scaffold.py` 生成入口与生成记录（`reports/generation/`）
- [x] 文档站骨架（mkdocs + mike 依赖）
- [x] `uv run dev` / `doctor` 零配置启动
- [ ] profiles 真正影响产物（当前仅保留字段）

验收：五种参数组合渲染成功；PySide6 / PySide2 两条链路真实生成通过。

### P1 运行时内核（已完成主体）

- [x] `config.py` 多环境配置
- [x] `observability`：structlog 结构化日志、崩溃捕获（OpenTelemetry 未接）
- [x] `core`（errors / result / bus / clock）与 `ports`（uow）
- [x] `platform/adapters/db`：engine、session、Base + 时间戳混入、SQLite 外键/WAL
- [x] Alembic 初始化 + 首个迁移（`0001_initial`）
- [x] `app/` 组合根（container / bootstrap / window / cli）、优雅退出
- [x] `__main__.py`：`--version` / `--self-test` / `--migrate` / `--mock` / `--dev`
- [x] 最小可启动主窗口
- [ ] 单实例锁
- [ ] fake 适配器与 `seed` / `db reset`

验收：`uv run dev` 出窗口；`alembic upgrade head` / `downgrade base` 通过；`--self-test` 退出码 0。

### P2 UI Runtime（六服务与设置页已完成）

- [x] `platform/ui/kit` 六服务 + `UiManager`
- [x] 设置示例页（七件套 + MVVM）
- [x] 主题切换（light / dark / high_contrast）、自绘弹窗、命名线程池（io / serial / cpu）
- [x] pytest-qt 用例（PySide2 固定 4.2.x）
- [ ] `platform/ui/widgets` 组件库（表格 / 表单 / 空态 / 加载 / 错误边界）
- [ ] F12 开发面板
- [ ] 分级热重载（L1 / L2；当前只有进程级轮询重启）

验收：明暗切换无残留样式；任务失败不冒泡；生命周期逆序清理。

### P3 业务切片（仅生成器完成）

- [x] `scripts/new_feature.py` 切片生成器（七件套 + 服务测试）
- [ ] `new page` / `new migration` / `new adapter` 生成器 + 自动注册
- [ ] 七件套按开关条件生成
- [ ] MySQL 集成测试跑通（当前只加依赖、compose 与 CI 服务）

验收：每个开关组合可启动，双库测试通过。

### P4 硬化（核心门禁已完成）

- [x] ruff / mypy / importlinter 配置（契约写在 `pyproject.toml`）
- [x] AST 架构测试（绑定唯一入口、platform 反向依赖）
- [x] 覆盖率门槛（`fail_under = 75`，实测 94%）
- [x] pre-commit
- [x] CI 四段（lint-type / test / security / 可选 build）
- [x] PyInstaller spec + 冒烟（仅 `include_packaging=true`）
- [x] 安全扫描接入（pip-audit）
- [x] Qt 绑定 AST 扫描（仅 `platform/ui/qt.py` 可导入绑定）
- [ ] 审查报告生成（`reports/review`）
- [ ] import-linter 切片边界契约（当前靠 AST 测试）
- [ ] jscpd 重复代码率门禁
- [ ] 复用资产覆盖率统计
- [ ] interrogate docstring 90% 接入 CI
- [ ] vulture 死代码扫描接入 CI
- [ ] commitlint 钩子
- [ ] `baseline.json` + 回归门禁
- [ ] watch 测试与反馈指标入基线
- [ ] 双绑定打包冒烟

验收：任一红线违规在本地或 CI 被拦截；启用打包时安装包可启动。

### P5 回流与发布

- `scripts/feedback/report.py`
- 漂移检测
- ADR / recipes / retro
- `git-cliff` 变更日志
- `_migrations` 升级路径
- `archive/` 与 `INDEX.md`
- 项目退役流程文档
- mkdocs 发布（mike）
- 复用率度量进 health 报告
- `packages/` 内部包拆分与发布
- 技术债登记与弃用策略
- 兼容矩阵
- 示例 gallery 与月度健康报告
- `v1.0.0` 发布

验收：产出真实 `health.md`；老项目 `copier update` 无冲突。

## 7. 风险与对策

| 风险 | 对策 |
|---|---|
| PySide6 打包体积大、启动慢 | 启用打包时用 `--onedir`、延迟导入、资源压缩、启动耗时纳入度量 |
| SQLite 多线程写冲突 | WAL + `busy_timeout` + 写操作单写者（serial 池） |
| 自绘弹窗无障碍不足 | 统一焦点顺序、键盘导航、屏幕阅读器标签，纳入 UI 测试 |
| 双库方言漂移 | 同一套集成测试跑 SQLite 与 MySQL，CI 强制双跑 |
| `copier update` 冲突 | 业务代码与模板代码分区、`_migrations` 显式迁移、漂移报告 |
| 元 CI 拖慢反馈 | 缓存 uv 与 Qt，矩阵只跑关键组合 |

## 8. 开放项

- 中英双语 i18n（Qt Linguist）：默认开启，可在生成时关闭。
- 自动更新：先做骨架 + 本地 mock 源，不绑定真实 CDN。
- CodeQL：可选接入，不阻塞首版。
- 代码签名：首版不做，发布流程预留占位。
