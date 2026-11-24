# 架构：横纵分层

横向是技术平台层，纵向是业务能力切片，两者正交。目标是模块化单体：切片内聚、切片间低耦合、跨切片能力全部下沉到平台层复用。

## 1. 横向：平台层 `platform/`

跨所有业务复用，只提供能力，不含具体业务规则。

| 目录 | 职责 |
|---|---|
| `platform/core/` | 错误类型 `AppError`（`errors.py`）、`Result`（`result.py`）、时钟（`clock.py`）、事件总线（`bus.py`） |
| `platform/ports/` | 跨模块契约：当前只有 `uow.py`（UnitOfWork 协议） |
| `platform/adapters/db/` | `base.py`（Base + 时间戳混入）、`engine.py`、`session.py`、`migrate.py` |
| `platform/ui/kit/` | UI Runtime：`manager` / `theme` / `qss` / `dialogs` / `layout` / `pages` / `tasks` / `lifecycle` |
| `platform/ui/qt.py` | Qt 绑定唯一导入入口 |
| `platform/observability/` | `logging.py`（structlog）、`crash.py`（崩溃 dump） |
| `platform/devtools/` | `cli.py`（`dev` / `doctor` 子命令）、`doctor.py`（环境体检） |
| `platform/config.py` | 配置与多环境 |
| `platform/selfcheck.py` | `--self-test` 的检查实现 |

规划中（当前未生成）：`platform/ui/widgets/`（表格 / 表单 / 空态 / 加载 / 错误边界）、`platform/ui/devtools/`（F12 面板）、`platform/adapters/{repositories,security,updater,sync}/`、OpenTelemetry。

## 2. 纵向：业务切片 `features/<name>/`

按业务能力端到端切片，每个切片内部自带完整分层。切片是「七件套」：

```text
features/settings/
├─ __init__.py     # 包标记，不导出业务符号
├─ api.py          # 唯一对外出口（含 ROUTE 常量）
├─ models.py       # ORM 模型（SQLAlchemy 2.0 typed）
├─ repository.py   # 仓储实现（继承平台基类）
├─ service.py      # 用例编排
├─ view.py         # 页面（QWidget，只用样式令牌）
└─ viewmodel.py    # 状态与命令（Signal 从 platform.ui.qt 导入）
```

初始切片只有一个：`settings`（主题 / 语言等偏好，落 `app_settings` 表）。

规划中：`users`、`sync`、`updates`、`onboarding`（对应 `include_auth` / `enable_sync` / `include_updater` 参数，当前为保留位）。

切片内单向依赖：

- `api -> service -> repository -> models`
- `view -> viewmodel -> service`

`api.py` 只导出稳定接口，禁止把内部实现细节泄露出去。

## 3. 依赖规则

| 关系 | 允许 |
|---|---|
| `features/* -> platform/*` | 是 |
| `platform/* -> features/*` | 否 |
| `features/A -> features/B.api` | 是 |
| `features/A -> features/B.service/repository/models` | 否 |
| `features/A -> features/B`（经事件总线） | 是 |
| `app/ -> 任意切片` | 是，且仅限组合根 |

组合根 `app/` 负责注册切片、装配依赖、挂载路由；所有依赖显式注入，不使用模块级全局单例。

## 4. 复用资产

| 资产 | 位置 | 说明 |
|---|---|---|
| 通用基类 | `platform/core`、`platform/adapters/db/base.py` | `Result`、`AppError`、事件总线、时钟、`Base` + 时间戳混入 |
| 数据访问 | `platform/adapters/db` | engine / session / UnitOfWork / 迁移执行 |
| UI Runtime | `platform/ui/kit` | 六服务 + QSS 生成，全部切片共用 |
| 切片生成器 | `scripts/new_feature.py` | 生成七件套 + `tests/unit/test_<name>_service.py` |
| 环境体检 | `platform/devtools/doctor.py` | Python / Qt / DB / 迁移 / 令牌五项检查 |

复用原则：同一逻辑出现两次即下沉到平台层；切片内不允许复制平台已有能力。

## 5. 生成后的目录

```text
myapp/
├─ pyproject.toml               # 依赖、依赖组、ruff/mypy/coverage、[tool.importlinter]
├─ uv.lock
├─ .python-version
├─ main.py                      # 根目录启动器（薄层转发）
├─ alembic.ini
├─ alembic/
│  ├─ env.py
│  ├─ script.py.mako
│  └─ versions/0001_initial.py
├─ pyinstaller.spec             # 仅 include_packaging=true
├─ mkdocs.yml                   # 仅 include_docs=true
├─ docker-compose.yml           # 仅 include_docker=true（MySQL 8 测试环境）
├─ .pre-commit-config.yaml
├─ .editorconfig
├─ .gitattributes
├─ .env.example
├─ .env-noupdate                 # 可选：本机敏感配置，已被 .gitignore 忽略
├─ .gitignore                    # 含 *-noupdate 与 *-noupdate.*
├─ .github/workflows/ci.yml
├─ docs/index.md                # 仅 include_docs=true
├─ reports/generation/*.json    # 生成记录（生成期写入）
├─ scripts/
│  ├─ dev.py                    # 轮询 src/ 热重启；--no-watch 关闭
│  ├─ new_feature.py            # 切片生成器
│  ├─ check_noupdate.py         # 拦截被 git 跟踪的 -noupdate 敏感配置
│  ├─ post_gen.py               # _tasks 调用：检测 uv 后 uv sync
│  └─ build.py                  # 仅 include_packaging=true
├─ src/myapp/
│  ├─ __init__.py               # __version__
│  ├─ __main__.py               # --version/--self-test/--migrate/--mock/--dev
│  ├─ app/                      # 唯一组合根
│  │  ├─ bootstrap.py           # 建 QApplication、装 excepthook、唯一 app.exec()
│  │  ├─ cli.py                 # dev / doctor 子命令
│  │  ├─ container.py           # 装配平台服务与切片
│  │  └─ window.py              # 主窗口 + 路由注册
│  ├─ platform/
│  │  ├─ config.py
│  │  ├─ selfcheck.py
│  │  ├─ core/{errors,result,bus,clock}.py
│  │  ├─ ports/uow.py
│  │  ├─ adapters/db/{base,engine,session,migrate}.py
│  │  ├─ devtools/{cli,doctor}.py
│  │  ├─ observability/{logging,crash}.py
│  │  └─ ui/
│  │     ├─ qt.py               # 绑定唯一入口
│  │     └─ kit/{manager,theme,qss,dialogs,layout,pages,tasks,lifecycle}.py
│  ├─ features/settings/        # 七件套
│  │  ├─ __init__.py
│  │  ├─ api.py
│  │  ├─ models.py
│  │  ├─ repository.py
│  │  ├─ service.py
│  │  ├─ view.py
│  │  └─ viewmodel.py
│  └─ resources/
│     ├─ __init__.py
│     └─ tokens.toml
└─ tests/
   ├─ conftest.py
   ├─ architecture/test_layers.py
   ├─ integration/test_migrate.py
   ├─ ui/{test_window,test_settings_page}.py
   └─ unit/*.py                    # 含 test_noupdate_guard.py
```

测试统一放在项目根 `tests/`，不在切片内建 `tests/`；切片生成器会追加 `tests/unit/test_<name>_service.py`。

### 5.1 入口与启动链

项目有且只有三个入口，职责严格分离：

| 入口 | 位置 | 职责 |
|---|---|---|
| 根目录启动器 | `main.py` | 薄层转发：解释器兜底（非项目 `.venv` 时切换）+ 路径兜底 + 调用 `myapp.__main__:main`，零业务逻辑 |
| 包入口 | `src/<pkg>/__main__.py` | 解析 CLI（`--self-test` / `--migrate` / `--mock` / `--dev`），调用 `app/` |
| 组合根 | `src/<pkg>/app/bootstrap.py` | 装配依赖、创建 QApplication、`app.exec()` |

启动链：`main.py` 或 `python -m <pkg>` -> `__main__.main(argv)` -> `bootstrap.run()`。

`main.py` 硬性约束：

1. 只允许 import 标准库与 `<pkg>.__main__`，禁止 import `features/*`、`platform/*`、`app/*`。
2. 解释器兜底：当前 `sys.executable` 不是项目 `.venv` 时用 `os.execv` 切换，失败则继续，不阻断启动。
3. 未做 `uv sync` 时把 `src/` 插入 `sys.path` 兜底，让 `python main.py` 可直接跑。
4. 捕获导入失败与启动异常，写结构化日志后返回非零退出码，不抛裸栈。
5. 参数原样透传，不自行解析。
6. 不参与打包：PyInstaller 入口固定为 `src/<pkg>/__main__.py`，`main.py` 仅本地与 CI 冒烟使用。
7. 文件保持在 60 行以内，且必须有 `--help` 可用。
8. `__main__` 的导入放在函数内，这是路径兜底的必要延迟导入；若开启 PLC0415，加 `# noqa: PLC0415` 并注明原因。
9. `os.execv` 必须带 `# noqa: S606` 并注明原因；`<PKG>_NO_REEXEC` 环境变量用于显式跳过切换。

```python
"""根目录启动器：优先切换到项目 .venv，再转发到 <pkg>.__main__:main。"""

from __future__ import annotations

import os
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent
NO_REEXEC_ENV = "<PKG>_NO_REEXEC"


def _venv_python() -> Path | None:
    """返回项目虚拟环境解释器，找不到时为 None。"""
    for relative in (".venv/Scripts/python.exe", ".venv/bin/python"):
        candidate = ROOT / relative
        if candidate.is_file():
            return candidate
    return None


def _ensure_venv() -> None:
    """系统 Python 启动时自动切换到项目 .venv，避免依赖缺失。"""
    if os.environ.get(NO_REEXEC_ENV):
        return
    target = _venv_python()
    if target is None:
        return
    try:
        if Path(sys.executable).samefile(target):
            return
    except OSError:
        pass
    args = [str(target), str(ROOT / "main.py"), *sys.argv[1:]]
    try:
        os.execv(str(target), args)  # noqa: S606 - 必须替换为项目解释器
    except OSError:
        return


def _ensure_src_on_path() -> None:
    """未安装包时把 src/ 加入导入路径。"""
    src = ROOT / "src"
    if src.is_dir() and str(src) not in sys.path:
        sys.path.insert(0, str(src))


def main() -> int:
    """启动应用并返回退出码。"""
    _ensure_venv()
    _ensure_src_on_path()
    try:
        from myapp.__main__ import main as app_main
    except ImportError as exc:
        print(f"无法导入 myapp，请先执行 uv sync：{exc}", file=sys.stderr)
        return 2
    return app_main()


if __name__ == "__main__":
    raise SystemExit(main())
```

**为什么放在根目录**：新人拿到仓库第一件事是找入口；`main.py` 让「双击/直接跑」成立，同时不污染包结构，打包与测试路径不受影响。

## 6. UI Runtime 契约

见 `ui-runtime.md`（六服务、主题、弹窗、布局、生命周期、线程池）。

## 7. 数据层

- 一切持久化数据成表；`QSettings` 只允许存数据库位置、`install_id`、安装根目录。
- 模型放 `features/<name>/models.py`，`Base` 与 `TimestampMixin` 放 `platform/adapters/db/base.py`。
- 当前公共列只有 `created_at` / `updated_at`（UTC，应用写入，不用数据库默认值）；主键按表语义选择（如 `app_settings` 用 `key` 作主键），需要自增时用 `id`。
- 同步实体（`enable_sync` 实现后）另加 `uid CHAR(26)` ULID、`version`、`deleted_at`；金额用 `BIGINT` 存最小单位；枚举用 `VARCHAR + CHECK`。
- 统一 SQLAlchemy 2.0 类型与方言差异：JSON、`DateTime`、Boolean、自增策略。
- 本地默认 SQLite（开启外键与 WAL）；改 `DATABASE_URL` 切 MySQL，业务代码不变。
- 只维护一份 Alembic 版本链，双库共用。
- CI 在 `enable_mysql=true` 时用 compose 起 MySQL 8，跑同一套集成测试。
- 迁移必须可回滚；每次迁移附数据兼容说明。

完整规范见 `database.md`。

## 7.1 敏感配置与 `-noupdate`

从开源项目里隔离本机凭证，不靠“记得别提交”，靠命名约定与自动拦截：

- **命名**：API Key、Token、私有连接串等本机文件统一命名为 `<用途>-noupdate.<扩展名>`，例如 `apikey-noupdate.toml`、`secrets-noupdate.json`、`.env-noupdate`。`-noupdate` 必须紧邻扩展名或位于结尾。
- **忽略**：根 `.gitignore` 固定包含 `*-noupdate` 与 `*-noupdate.*`，覆盖任意目录层级。
- **拦截**：`scripts/check_noupdate.py` 读取 `git ls-files`，发现被跟踪的敏感文件即退出码 1；pre-commit 的 `check-noupdate` 钩子与 CI 的 `lint-type` 段执行同一脚本。
- **示例文件**：可提交只含占位符的 `.env.example` / `config.example.toml`；真实值只进 `-noupdate` 文件。
- **误提交**：先 `git rm --cached <file>`，再轮换已暴露的密钥；仅删除历史不等于止损。
- 明文密钥仍遵守 `database.md`：业务侧只存 `key_id` 与指纹，运行时密钥走 DPAPI 或系统密钥库。

## 8. 业务切片骨架

| 切片 / 平台 | 状态 | 实现要点 |
|---|---|---|
| `features/settings` | 已实现 | 主题/语言等偏好入 `app_settings` 表，MVVM 设置页，切换主题即时生效 |
| `platform/ui/kit` + `platform/ui/kit/tasks.py` | 已实现 | 六服务 + QThreadPool 命名池（io / serial / cpu），任务信号回传 UI |
| `platform/devtools/doctor.py` | 已实现 | Python / Qt / DB / 迁移 / 令牌五项体检 |
| `features/onboarding` | 规划 | 首启向导、校验后落 `.env`、日志查看入口 |
| `features/users` | 规划 | 本地用户表、argon2 哈希、角色、`@require_perm("x")` 服务层守卫 |
| `features/updates` | 规划 | 签名清单 JSON -> 下载 -> sha256 校验 -> 重启替换（Windows rename-on-reboot），stable/beta 通道，先接本地 mock 源 |
| `features/sync` | 规划 | `sync_outbox` 变更日志，push/pull，`version + updated_at` 冲突策略，SQLite <-> MySQL |
| `platform/observability` | 已实现 | `sys.excepthook` + Qt handler，dump 到 `%LOCALAPPDATA%`；用户同意后上报尚未接 |
| `platform/ui/widgets` | 规划 | 表格 / 表单 / 空态 / 加载 / 错误边界 |
| F12 开发面板 | 规划 | SQL 与任务队列查看 |

切片之间只经 `api.py` 或事件总线通信；需要跨切片共享的逻辑一律上移到 `platform/`。
