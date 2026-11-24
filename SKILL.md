---
name: DGUIApp开发脚手架
description: 生成、审查或扩展 DGUIApp（PySide6）桌面应用脚手架。涉及 Copier 模板、横纵分层（platform/features）、SQLite 可切 MySQL、MVVM + UI Runtime、严格代码规范、质量门禁、DX 一键启动与热重载时使用；用户要求新建桌面应用工程、检查脚手架合规性或往脚手架加功能切片时也要使用。
---

# DGUIApp 开发脚手架

一套已经定稿的工程约定：Copier 模板 + 横纵分层 + 生产级门禁 + 快速开发。
本技能是执行入口，细节按需读取 `references/`。

## 适用与不适用

适用：

- 从零生成一个 PySide6 桌面项目。
- 在已有生成项目里新增业务切片、页面、迁移、适配器。
- 审查一个 PySide6 项目是否符合本套分层与质量门禁。
- 维护 Copier 模板本身（改参数、加 profile、升级迁移）。

不适用：

- Web / 移动端 / 服务端项目。
- 非 PySide6 的 Python GUI（Tkinter、wxPython）。

## 生成流程

前置：Windows 10/11、`uv` 已安装、目标目录为空或仅含 `.git`。**开工前先探测下载源**：判定慢就立即切镜像，不要带着慢源开始渲染或同步。

```powershell
copier copy --trust <模板仓库> myapp
cd myapp
uv sync
uv run dev          # 建库 -> 迁移 -> 出窗口
uv run pytest -q
```

也可用模板仓库的生成器一步完成（探测 + 渲染 + git + 冒烟）：

```powershell
python scripts/scaffold.py C:\path\myapp
```

生成器（模板仓库的 `scripts/scaffold.py`）做五步，顺序固定：

1. 探测 Python 版本并锁定。
2. 校验参数（包名合法、目录为空）。
3. 调用 `copier.run_copy` 渲染。
4. `git init` 并做首次提交（可用 `--no-git` 跳过）。
5. 写生成记录；`.venv` 存在时跑 `--self-test` 冒烟，失败删除半成品。

## 下载与镜像（工程前 + 开发中）

任何联网下载（解释器、依赖、模板、工具、wheel）都按同一口径判定；**一旦慢，立即切换镜像，不反复重试原源**。

| 判定 | 阈值 |
|---|---|
| 吞吐不足 | 低于 200 KB/s 且持续 10 秒 |
| 无进度 | 15 秒内字节数不增长 |
| 单任务超时 | 超过 60 秒且预计剩余超过 5 分钟 |
| 连接失败 | 连续 2 次超时、重置或 5xx |
| 基线回归 | 全新 `uv sync` 超过 2 分钟 |

- 工程开始前：5 秒超时探测 PyPI / GitHub / 解释器源；首字节超过 2 秒或不可达即视为慢，先为本次会话设镜像。
- 开发中：发现慢立即暂停当前下载，探测候选源，选最快者重试，并记录原源、镜像与耗时。
- 配置优先级：单次命令 > 会话环境 > 项目配置 > 用户全局；写全局必须先告知并给出回滚。
- 有锁文件时只用 `uv sync --locked`；切镜像不得升级版本、不得关闭哈希校验。
- 只用 HTTPS 可信镜像；私有包与凭证只走官方或私有源，绝不经第三方代理。
- 细节、探测命令与各资源切换方式见 `references/mirrors.md`。

## 锁定参数

| 项 | 值 |
|---|---|
| 包名 / 显示名 | `myapp` / `MyApp` |
| Python | 自动探测：uv > 当前解释器 > `py -0p`，回退 3.12 |
| 平台 / 许可证 | 仅 Windows / MIT |
| GUI / 数据 | PySide6 / SQLAlchemy 2.0 + Alembic |
| Qt 降级 | 主选 PySide6，构建时自动降级 PySide2（Qt5，仅 32 位或 Win7/8 环境） |
| 工具链 | uv + ruff + mypy(strict) + pytest + pytest-qt（PySide2 固定 `>=4.2,<4.3`） |
| 分发 / 模式 | 默认不打包；仅用户明确要求时才启用 PyInstaller `--onedir` / MVVM |
| 架构 | 横纵分层 + ports/adapters（模块化单体） |
| 数据原则 | 每种持久化数据都有对应表；`QSettings` 只存引导信息 |

`python_version` 一旦锁定，`requires-python`、`.python-version`、ruff `target-version`、mypy `python_version`、CI 矩阵全部由它派生；启用打包时 PyInstaller 解释器也用它。禁止各处手写版本号。

## 当前实现状态（2026-11-24）

| 项 | 状态 |
|---|---|
| Copier 模板 + 生成器 + 渲染矩阵 | 完成，21 项全绿；`meta-ci.yml` 已加 |
| PySide6 3.12 与 PySide2 3.10 双路径 | 完成，ruff/mypy/pytest/`--self-test` 全绿，覆盖率 94%+ |
| 模板仓库治理资产 | README / CHANGELOG / LICENSE / .gitignore / meta CI 已完成 |
| 六服务 UI Runtime + 设置切片七件套 | 完成 |
| `import-linter` 分层契约 | 完成，配置在生成项目 `pyproject.toml` |
| `uv run dev` / `doctor` / `scripts/dev.py` | 完成，离屏冒烟通过 |
| 解释器兜底 + 编辑器配置 | 完成：`main.py` 自动切 `.venv`，生成 `.vscode` 三件套 |
| `include_docs` / `include_docker` / `include_packaging` | 完成，均默认关闭（打包必须显式开启） |
| `profile` / `enable_sync` / `include_auth` / `include_updater` | 参数保留位，暂无分支产物 |
| `-noupdate` 敏感配置保护 | 完成：`.gitignore` + `scripts/check_noupdate.py` + CI + 渲染断言 |
| widgets 库 / F12 面板 / L1-L2 热重载 / seed / db reset | 规划 |

## 架构红线（违反即返工）

1. `features/*` 可以依赖 `platform/*`；`platform/*` 禁止依赖 `features/*`。
2. `features/A` 访问 `features/B` 只能 `from ..B.api import ...` 或走事件总线。
3. `app/` 是唯一组合根，唯一允许同时引用所有切片的地方。
4. 页面不写颜色与样式：`features/*/view.py` 禁止 `#RRGGBB` 与 `setStyleSheet`。
5. `QMessageBox` 只在 `platform/ui/kit/dialogs.py`；`QThread` / `QRunnable` 只在 `platform/ui/kit/tasks.py`；`app.exec()` 只在 `app/bootstrap.py`。
6. UI 线程不做 IO；跨线程只经 `platform/ui/kit/tasks.py` 的信号回传。
7. 同一逻辑出现两次即下沉到 `platform/`，切片内禁止复制平台能力。
8. 根目录 `main.py` 是薄层启动器：先做解释器兜底（非项目 `.venv` 时自动切换），再转发到 `myapp.__main__:main`；禁止写业务逻辑、禁止 import `features/*`、`platform/*`、`app/*`。
9. Qt 绑定只在 `platform/ui/qt.py` 导入；其他任何文件出现 `PySide6` / `PySide2` 即为违规。
10. 禁止 Qt5 独有或已废弃 API：`QRegExp`、`QDesktopWidget`、`QFontMetrics.width`、`exec_()`。
11. 每种持久化数据都要有对应的表；`QSettings` / JSON 文件 / 注册表只允许存引导信息（数据库位置、`install_id`、安装根目录）。
12. 文件本体不入库：大文件放磁盘，另建元数据表登记路径、sha256、大小、时间。
13. 开源敏感配置不入库：API Key、Token、私有连接串等本机文件统一命名为 `<用途>-noupdate.<扩展名>`（或 `.env-noupdate`）；`.gitignore` 必须含 `*-noupdate` 与 `*-noupdate.*`，并由 `scripts/check_noupdate.py` 在 pre-commit 与 CI 拦截已跟踪文件。禁止把明文密钥写进任何受版本控制的文件。

## 代码硬指标

| 项 | 上限 |
|---|---|
| 函数 | 30 行 |
| 文件 / 类 | 300 行 / 200 行 |
| 参数 / 嵌套 | 5 个 / 3 层 |
| 圈复杂度 | 10 |
| 行宽 / docstring | 100 / Google 风格，interrogate ≥ 90% |

参数超过 3 个必须 keyword-only；禁止位置布尔参数；可预期失败返回 `Result[T, E]`，异常只用于意外。

## 质量门禁

| 指标 | 门槛 |
|---|---|
| 核心 / 整体覆盖率 | 85% / 75% |
| 冷启动 / idle 内存 | 2.5s（P95） / 250MB |
| 崩溃率 / 数据丢失 | 0.5% / 0 |
| 重复代码率 | 3% |
| 平台复用率 / 新功能复用率 | 60% / 80% |
| 敏感配置被 git 跟踪 | 0 个（`scripts/check_noupdate.py` 必须通过） |
| 劣化回归 | 超过基线 5% 失败 |

不达标不发布；发布走 Go/No-Go 清单 + 回滚方案 + 值班人。打包相关门槛（安装包冒烟、签名、体积）只在启用打包时适用。

## 参考文件

| 文件 | 何时读 |
|---|---|
| `references/architecture.md` | 建目录、定依赖、写数据层或切片骨架 |
| `references/ui-runtime.md` | 写页面、弹窗、主题、任务提交或生命周期 |
| `references/code-standards.md` | 写代码前对齐函数与写法规范 |
| `references/quality-gates.md` | 配 CI、配门禁、判断能否发布 |
| `references/database.md` | 建表、改表、写迁移、ORM 与事务 |
| `references/lifecycle.md` | 生成记录、审查报告、归档、文档站 |
| `references/dx.md` | 做一键启动、热重载、生成器、开发面板 |
| `references/template-spec.md` | 改 `copier.yml`、加 profile、按阶段实施 |
| `references/qt-binding.md` | 处理 PySide6 / PySide2 兼容、降级、打包差异 |
| `references/mirrors.md` | 下载慢、依赖装不上、模板拉不下来、解释器或工具下载卡住 |

## 任务路由

| 用户要什么 | 做什么 |
|---|---|
| 新建项目 | 按「生成流程」执行；参数与阶段状态见 `template-spec.md` |
| 找入口 / 改启动链 | 根 `main.py` 只做解释器兜底与转发，逻辑落在 `__main__.py`，`app.exec()` 只在 `bootstrap.py` |
| 加业务功能 | `uv run python scripts/new_feature.py <name>`，读 `architecture.md` 的切片模板 |
| 加页面 | 在切片内加 `view.py` + `viewmodel.py`，页面继承 `QWidget`，样式只用令牌；`new page` 生成器尚在规划 |
| 加表 / 改表 | 手写 Alembic 迁移（`new migration` 生成器尚在规划），先读 `database.md`；必须可回滚 |
| 改主题 | 只改 `tokens.toml`，不碰业务代码 |
| 配 CI | 按 `quality-gates.md` 的四段流水线 |
| 发版 | `lifecycle.md` 的审查 + Go/No-Go + 归档 |
| 模板升级 | 读 `template-spec.md` 的参数表与渲染矩阵；`_migrations` 升级脚本规划中 |
| 兼容老环境 | 读 `qt-binding.md`，绑定只在 `platform/ui/qt.py`，构建时自动降级 |
| 要打包分发 | 先确认用户明确要求；才生成 `pyinstaller.spec`、`scripts/build.py` 与打包 CI |
| 下载慢 / 依赖装不上 / 模板拉不下来 | 按 `mirrors.md` 判定慢速并立即切镜像；记录切换前后的实测 |
| 换机器 / 内网部署 | 用 `mirrors.md` 的离线缓存与私有源方案，不依赖个人会话配置 |
| 加 API Key / 敏感配置 | 本机文件命名成 `*-noupdate.*`（如 `apikey-noupdate.toml`），不写进受控文件；跑 `uv run python scripts/check_noupdate.py` 确认未入库 |

## 生成后验收

- [ ] `uv run dev` 出窗口，无裸栈报错。
- [ ] `uv run app --self-test` 退出码 0，`uv run app --help` 有用法输出。
- [ ] `uv run pytest -q` 全绿，覆盖率达标。
- [ ] `uv run ruff check .` 与 `uv run mypy src` 零错误。
- [ ] `uv run lint-imports` 分层契约通过。
- [ ] `alembic upgrade head` 与 `downgrade base` 均可执行。
- [ ] 每种持久化数据都能对应到一张表，主题/语言/最近打开等均已落库。
- [ ] `uv run doctor` 自检全绿，输出当前 Qt 绑定与版本。
- [ ] PySide6 与 PySide2 两个绑定各出一次可启动产物（仅启用打包时要求）。
- [ ] 明暗主题切换无残留样式。
- [ ] 关闭窗口后进程与单实例锁正常释放。
- [ ] 用系统 Python 执行 `python main.py --self-test` 仍能自动切到 `.venv` 并通过。
- [ ] 生成项目自带 `.vscode/settings.json` 与 `launch.json`，默认解释器指向 `.venv`。
- [ ] `.gitignore` 含 `*-noupdate` 与 `*-noupdate.*`，`uv run python scripts/check_noupdate.py` 通过；仓库内无被跟踪的敏感配置。
- [ ] 下载全程无长时间卡住；切过镜像时，生成记录已写明原源、镜像与耗时。

## 常见错误

| 现象 | 原因 | 处理 |
|---|---|---|
| `platform` 导入 `features` | 依赖方向反了 | 把共用能力上移到 `platform/` |
| 页面样式不生效 | 直接写了 `setStyleSheet` | 改用 `tokens.toml` 令牌 |
| UI 卡死 | 在 UI 线程做 IO | 改走 `ui.tasks.submit(..., pool="io")` |
| SQLite `database is locked` | 并发写 | WAL + `busy_timeout`，写操作走 `serial` 池 |
| 打包后启动失败（启用打包时） | 资源未收集 | 检查 `pyinstaller.spec` 的 datas 与隐式导入 |
| `copier update` 冲突 | 改了模板保护区 | 业务代码放特征区，模板区变更走 `_migrations` |
| `python main.py` 报 `ModuleNotFoundError: structlog/PySide6` | 用了系统 Python，或启动器未做解释器兜底 | 根 `main.py` 必须先比较 `sys.executable` 与项目 `.venv`，不一致就 `os.execv` 切换；同时生成 `.vscode` 配置 |
| `python main.py` 报 `ModuleNotFoundError`（包本身） | 未 `uv sync` 或 `src/` 兜底失效 | 先 `uv sync`，再确认 `main.py` 插入 `src/` 与包名一致 |
| 启动器里堆业务逻辑 | 职责越界 | 全部移回 `__main__.py` / `app/` |
| 降级后 `ImportError` | 直接写了 `import PySide6` | 改从 `platform/ui/qt.py` 导入 |
| Qt5 下高分屏模糊 | 未设高 DPI 属性 | 调 `qt.enable_high_dpi()`，且在 QApplication 之前 |
| 同时装了 PySide6 与 PySide2 | 依赖没锁死 | 只保留一个绑定依赖，重锁 `uv.lock` |
| 设置项重启后丢失 | 存了 `QSettings` 或内存 | 入 `app_settings` 表 |
| 模型加了字段但库没变 | 没生成迁移 | 手写 Alembic 迁移（`new migration` 生成器规划中），CI 跑 `alembic check` |
| 启动慢 | 首屏同步导入重模块 | 延迟导入，启动耗时进基线 |
| `uv sync` 长时间无进度 | 源慢或被限速 | 按 `mirrors.md` 判定并切镜像，不反复重试 |
| 镜像缺少锁定版本 | 镜像同步滞后 | 该次命令回官方源，保持版本不变 |
| 配了镜像仍然慢 | 镜像本身慢，或代理与镜像叠加 | 重新探测，只保留一层加速 |
| 切换镜像后版本被升级 | 误跑了 `uv lock --upgrade` | 回到 `uv.lock`，用 `--locked` 重装 |
| TLS 证书错误 | 企业证书未被信任 | `UV_SYSTEM_CERTS=1` 或配置企业根证书，不要关闭校验 |
| API Key 出现在提交/流水线 | 敏感配置未按 `-noupdate` 命名或未忽略 | 文件改名为 `*-noupdate.*`，`git rm --cached` 后轮换密钥，并跑 `scripts/check_noupdate.py` |

## 输出要求

- 改代码前先读对应 reference，不要凭记忆套规范。
- 新增文件必须同时补测试与注册（容器、路由）。
- 生成器改动后必须跑一次渲染矩阵，五种组合全部通过。
- 切过下载源必须记录原源、镜像与耗时；不得静默修改用户全局配置。
- 生成项目必须包含可移植的 `.vscode/settings.json`、`launch.json`，解释器用 `${workspaceFolder}/.venv/Scripts/python.exe`，不写绝对路径。
- 涉及 API Key / Token / 私有连接串的配置必须使用 `-noupdate` 命名后缀，并确认 `check_noupdate` 与 CI 通过。
- 任何绕过门禁的改动都要在回复里显式说明原因与补救计划。
