# 快速开发（DX）

目标：新项目 5 分钟内出窗口，日常改动反馈 < 5 秒。

## 0. 已实现 / 规划

| 能力 | 状态 | 命令或文件 |
|---|---|---|
| 一键启动（迁移 + 窗口） | 已实现 | `uv run dev` |
| 热重启（轮询 `src/**/*.py`） | 已实现 | `uv run python scripts/dev.py` |
| 关闭热重启 | 已实现 | `uv run python scripts/dev.py --no-watch` |
| 环境体检 | 已实现 | `uv run doctor` |
| 自检并退出 | 已实现 | `uv run app --self-test` |
| 切片生成器（七件套 + 服务测试） | 已实现 | `uv run python scripts/new_feature.py <name>` |
| 打包（仅显式开启） | 已实现 | `uv run python scripts/build.py` |
| L1/L2 热重载、F12 面板、seed、db reset | 规划 | — |
| `new page` / `new migration` / `new adapter` | 规划 | — |

## 1. 一键启动

```powershell
uv sync
uv run dev                     # 建库 -> 迁移 -> 启动窗口
uv run python scripts/dev.py   # 同上，并在源码变化时重启
uv run doctor                  # 环境体检：Python / Qt / DB / 迁移 / 令牌
```

- 首次启动自动创建数据库目录，零配置可跑。
- 失败时输出可执行的修复建议，不抛裸栈。
- `pyproject.toml` 注册 `[project.scripts]`，`app` / `dev` / `doctor` 随项目分发。
- 当前 `dev` 不接受 `--mock` / `--reset`；需要 mock 时用 `uv run app --mock`。

## 2. 分级热重载

| 级别 | 范围 | 机制 |
|---|---|---|
| L1 | 主题 / QSS / 图标 | 文件监听 -> 重新应用 QSS，不重启 |
| L2 | 页面模块 / ViewModel | 卸载页面 -> reload 模块 -> 重新挂载 |
| L3 | 平台层 / 数据结构 | watchfiles 重启进程 |

- 监听 `src/`、`tokens.toml`、`resources/`。
- 热重载失败自动降级到 L3，并保留错误日志。
- L2 只在调试模式启用，发布包不包含热重载代码。

## 3. 代码生成器

已实现：

```powershell
uv run python scripts/new_feature.py invoices
```

生成 `features/invoices/` 七件套（`__init__` / `api` / `models` / `repository` / `service` / `view` / `viewmodel`）与 `tests/unit/test_invoices_service.py`，然后提示在 `app/window.py` 注册路由。

规划：

```powershell
uv run new page settings --feature settings
uv run new migration "add user roles"
uv run new adapter reporter
```

生成器必须遵守的约束：

1. 生成的文件立即满足函数与文件行数上限。
2. 生成的 `view.py` 不含任何颜色与 `setStyleSheet`。
3. 生成的切片默认只 import `platform/`，不 import 其他切片。
4. 生成的迁移带 `downgrade()`，且不修改已有迁移。

## 4. 种子与假适配器（规划）

- `uv run seed`：固定随机种子，生成可复现数据。
- `uv run db reset`：删库 -> 迁移 -> 种子。
- fake 实现：Clock、Updater、Reporter、SyncTransport、FileSystem。
- `--mock` 启动时全部替换为内存实现，测试与开发一致；当前已支持 `--mock` 使用内存数据库。

## 5. 开发面板（F12）

- 日志：实时过滤、级别切换、复制。
- SQL：语句、耗时、行数、慢查询标红。
- 任务队列：运行中 / 排队 / 历史，可取消。
- 功能开关：灰度与实验特性。
- 主题：即时切换与令牌查看。

面板只在开发构建注册，生产构建通过编译期开关剔除。

## 6. watch 测试（规划）

```powershell
uv run pytest -q                 # 当前可用：全量测试
uv run test:watch                # 规划：文件变化自动跑相关测试
```

## 7. 反馈目标（硬指标）

| 场景 | 目标 |
|---|---|
| 单元测试（受影响） | < 5s |
| 完整测试 | < 90s |
| `uv run dev` 到窗口 | < 3s（热启动） |
| 全新 `uv sync` | < 2min |
| 生成 feature 并跑通测试 | < 30s |

指标纳入质量基线，回归超过 5% 失败。全新 `uv sync` 超过 2 分钟时，先按 `mirrors.md` 判定是否为下载源问题，再决定是否优化依赖。

## 8. 下载加速与镜像

开工前与开发中共用 `mirrors.md` 的判定口径：

- 单文件低于 200 KB/s 持续 10 秒、15 秒无进度，或全新 `uv sync` 超过 2 分钟：判定为慢，立即切换。
- 会话级设 `UV_DEFAULT_INDEX`；需要团队一致时写进 `pyproject.toml` 的 `[[tool.uv.index]]`（`default = true`）。
- 解释器、uv 本体、Copier 模板分别用 `UV_PYTHON_INSTALL_MIRROR`、`UV_ASTRAL_MIRROR_URL` 与"预克隆到本地再渲染"。
- 切换后用 `uv sync --locked` 重试，版本与哈希不变；禁止 `--trusted-host` 与关闭校验。
- 切换原因、镜像与耗时写入生成记录；公共 GitHub 代理使用前必须探测。

## 9. 排障顺序

1. `uv run doctor` 看环境。
2. `uv run app --mock` 排除数据库影响。
3. 看 `%LOCALAPPDATA%\<App>\crashes\` 的崩溃 JSON 与结构化日志。
4. 仍无法定位时，用 `--self-test` 做最小复现。
