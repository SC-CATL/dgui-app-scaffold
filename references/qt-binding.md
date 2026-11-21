# Qt 绑定与自动降级

主选 PySide6（Qt6）。当目标环境拿不到 PySide6 wheel 时，构建时自动降级到 PySide2（Qt5）。

> 术语纠正：Qt5 的官方绑定叫 **PySide2**，不存在 "PySide5"。文档与代码统一写 `pyside2` / `pyside6`。

## 1. 绑定矩阵

| 绑定 | Qt | 支持 Python | Windows 架构 | 状态 |
|---|---|---|---|---|
| PySide6 | Qt6 | 3.9 - 3.13 | 仅 64 位 | 主选，持续维护 |
| PySide2 | Qt5 | 3.5 - 3.10 | 32 / 64 位 | 仅兼容模式，上游已停止维护 |

结论：

- 32 位 Windows 或 Win7/8/8.1 只能走 PySide2。
- 走 PySide2 时 Python 必须钳制到 3.10 及以下。
- PySide2 是兼容通道，不是默认值；新项目一律 PySide6。

## 2. 选择规则（`qt_binding=auto`）

按顺序判断，命中即停：

1. 显式覆盖：`--data qt_binding=pyside6|pyside2`。
2. 架构为 32 位 -> `pyside2`，`python_version = min(detected, 3.10)`。
3. 目标系统低于 Windows 10 -> `pyside2`，同上钳制。
4. 探测 PySide6 是否存在匹配 wheel：
   - 查 PyPI JSON：`https://pypi.org/pypi/PySide6/json`，匹配 `win_amd64.whl`，且文件名含 `abi3` 或 `cp{major}{minor}`。
   - 网络不可用时保守判定为可用（避免误降级），并在生成记录里标注未验证。
   - 仍无法判断则按 `>=6.6` 最低可用版本假设。
5. 有匹配 wheel -> `pyside6`。
6. 无匹配 wheel 且 Python > 3.10 -> 逐级下调 Python；仍无 -> `pyside2` + 钳制 3.10。

探测只做一次，结果锁定，不在运行时反复判断。

## 3. 锁定与记录

选择结果写入四处，必须一致：

| 位置 | 内容 |
|---|---|
| `pyproject.toml` | 只声明一个绑定依赖，禁止同时出现 `PySide6` 与 `PySide2` |
| `uv.lock` | 与声明的绑定一致 |
| `.copier-answers.yml` | `qt_binding` 值，便于 `copier update` 保持稳定 |
| `reports/generation/<timestamp>.json` | `qt_binding`、`qt.version`、探测到的 Python、模板版本 |

`tests/test_render.py` 断言：渲染结果中的绑定依赖唯一、`requires-python` 满足绑定要求（PySide2 时为 `>=3.10`）。

规划：`baseline.json` 按绑定分别记录冷启动与内存基线，不跨绑定比较。

## 4. 构建时降级

`scripts/build.py` 在打包前做一次目标环境检查（仅启用打包时存在该脚本；以下为规划行为，当前脚本只做 onedir 构建 + 产物 `--self-test`）：

```text
读取锁定绑定
  -> 检查目标环境是否有可用 wheel（目标架构 / 目标 Windows 版本）
  -> 不可用则切 pyside2 + 钳制解释器 + 使用 constraints/qt5.txt
  -> 重装依赖 -> 重新构建
```

规则：

1. 降级只发生在构建时，不改源码，不改业务代码。
2. 降级必须打印原因，并写 `reports/build/<timestamp>.json`（原绑定、目标绑定、原因、解释器版本）。
3. 降级后必须重跑 `--self-test` 与冒烟，失败则本次构建失败，不产出半成品。
4. 不允许"静默降级后照常发布"：发布说明必须列出实际绑定。
5. 同一版本的不同产物绑定不同时，安装包文件名带后缀：`MyApp-1.2.0-qt6.exe`、`MyApp-1.2.0-qt5.exe`。

## 5. 唯一导入入口

业务代码禁止直接 `import PySide6` / `import PySide2`，只允许：

```python
from myapp.platform.ui.qt import QtCore, QtGui, QtWidgets, Signal, Slot, QAction, BINDING
```

`platform/ui/qt.py` 是唯一绑定适配点。绑定在**生成期**由 Jinja 分支静态选定，不写运行时 `try/except ImportError`：

```python
{% if qt_binding == "pyside2" -%}
from typing import Any

from PySide2 import QtCore, QtGui, QtWidgets
from PySide2.QtCore import QObject, Qt

# PySide2 的 Signal / Slot 无类型描述符，统一降级为 Any，业务侧类型检查才不会误报。
Signal: Any = QtCore.Signal
Slot: Any = QtCore.Slot

BINDING = "PySide2"
QAction = QtWidgets.QAction
{%- else -%}
from PySide6 import QtCore, QtGui, QtWidgets
from PySide6.QtCore import QObject, Qt, Signal, Slot

BINDING = "PySide6"
QAction = QtGui.QAction
{%- endif %}
```

同一文件还提供跨版本助手，业务代码不得再写 `exec_()`：

- `exec_app(app)`：Qt6 走 `app.exec()`，Qt5 走 `app.exec_()`。
- `exec_dialog(dialog)`：同上。
- `enable_high_dpi()`：Qt6 设 `PassThrough` 策略；Qt5 设 `AA_EnableHighDpiScaling` / `AA_UseHighDpiPixmaps`（用 `getattr` 规避类型桩缺失）。

绑定相关额外约束：

- `QAction` 在 Qt6 属于 `QtGui`、在 Qt5 属于 `QtWidgets`，只在此处归一化。
- 静态检查：除 `platform/ui/qt.py` 外，任何文件出现 `PySide6` / `PySide2` 导入即失败（`tests/architecture/test_layers.py`）。
- PySide2 路径的 `pytest-qt` 必须固定 `>=4.2,<4.3`；4.5 起上游移除 PySide2 支持，装新版会让整个测试会话在启动时失败。
- 实测记录：PySide6 6.12 + Python 3.12.10、PySide2 5.15.2.1 + Python 3.10.22 两条链路均通过 ruff / mypy / pytest / `--self-test`。

## 6. Qt6 与 Qt5 差异清单

必须避开的高危差异：

| 项目 | Qt6 | Qt5 | 做法 |
|---|---|---|---|
| 高分屏 | 默认开启 | 需手动属性 | 在 `qt.py` 暴露 `enable_high_dpi()`，Qt5 时设 `AA_EnableHighDpiScaling` 与 `AA_UseHighDpiPixmaps`，且在 QApplication 之前调用 |
| `QAction` 位置 | `QtGui` | `QtWidgets` | 经 `qt.py` 导入 |
| 正则 | `QRegularExpression` | `QRegExp` 已废弃 | 禁用 `QRegExp`，统一 `QRegularExpression` |
| 文本宽度 | `horizontalAdvance` | `width` 已废弃 | 只用 `horizontalAdvance` |
| 屏幕信息 | `QScreen` | `QDesktopWidget` 已废弃 | 只用 `QScreen` |
| 事件循环 | `app.exec()` | `exec_()` 仍可用 | 统一走 `qt.exec_app()` / `qt.exec_dialog()` |
| 枚举 | 作用域枚举 | 兼容旧写法 | 统一 `Qt.AlignLeft` 旧式写法，两版都支持 |
| 高 DPI 图标 | 自动 | 需 `AA_UseHighDpiPixmaps` | 图标一律 SVG |

禁止清单（AST 扫描拦截）：`QRegExp`、`QDesktopWidget`、`QFontMetrics.width`、`exec_()`。

## 7. 打包差异

仅当 `include_packaging=true` 时适用；默认不生成 spec 与 build 脚本。

- PySide6：PyInstaller 用官方 hook，`--onedir`，体积约 150-250MB。
- PySide2：hook 不同，需排除 `QtWebEngine` 等未用模块，体积略小。
- `pyinstaller.spec` 里按 `BINDING` 条件收集 datas 与 hiddenimports，不写死绑定名。
- 两种绑定分别构建，各自跑 `--self-test` 冒烟，互不替代。

## 8. CI 矩阵

| 任务 | 绑定 | Python | 目的 |
|---|---|---|---|
| 主路径 | PySide6 | 锁定版本 | 常规门禁与覆盖率 |
| 兼容路径 | PySide2 | 3.10 | 保证降级通道可用 |
| 打包 | 两个绑定 | 对应版本 | 启用打包时，两个安装包均可启动 |

兼容路径可以只在 nightly 或发布前跑，但发布版本必须包含一次双绑定构建记录。

## 9. 风险与边界

- PySide2 上游已停止维护，只作兼容通道；新功能不得依赖 Qt5 独有能力。
- 降级会带来性能差异；规划中的 `baseline.json` 按绑定分别记录，不跨绑定比较。
- 兼容路径覆盖率实测 94.46%，与主路径基本持平；可接受下限仍是 70%。
- 每个版本发布说明必须写明实际绑定；连续两个版本走 Qt5 时，需在技术债登记中给出 Qt6 迁移计划。
