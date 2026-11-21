# UI Runtime 契约

## 6. UI Runtime 契约 `platform/ui/kit`

所有 Qt 符号经 `platform/ui/qt.py` 导入（`QtCore` / `QtGui` / `QtWidgets` / `Signal` / `Slot` / `QAction` / `BINDING`），业务代码禁止直接 `import PySide6` 或 `import PySide2`。绑定选择与 Qt5 降级见 `qt-binding.md`。

`UiManager` 是门面，聚合六个可独立测试的子服务：

```python
class UiManager:
    theme: ThemeService        # kit/theme.py
    dialogs: DialogService     # kit/dialogs.py
    layout: LayoutService      # kit/layout.py
    pages: PageRouter          # kit/pages.py
    tasks: ExecutorService     # kit/tasks.py
    lifecycle: LifecycleService  # kit/lifecycle.py

    def create_shell(self, title: str) -> ShellWindow: ...
    def shutdown(self) -> None: ...   # lifecycle.shutdown() + tasks.shutdown()
```

### 6.1 色彩与主题（已实现）

- `resources/tokens.toml` 是唯一色彩与尺寸来源，`kit/qss.py` 负责令牌 -> QSS 文本。
- `theme.apply("light" | "dark" | "high_contrast")` 生成并设置全局 QSS，返回实际模式；`theme.current` 记录当前模式，切换时发 `changed(str)` 信号。
- 令牌文件按路径缓存（`_CACHE`），重复读取不重复解析。
- `tomllib` / `tomli` 用 `import_module` 按解释器版本选择，避免 3.10 的 mypy 与 3.12 的 ruff 冲突。
- `features/*/view.py` 内禁止出现 `#RRGGBB` 与 `setStyleSheet`（AST 扫描）。
- 规划：高对比度令牌组切换后的对比度自动校验、第三方主题加载入口。

### 6.2 弹窗（已实现）

```python
ui.dialogs.confirm(title, body) -> bool      # 模态，仅确认返回 True
ui.dialogs.notify(title, body) -> None       # 单按钮提示
ui.dialogs.error(title, body) -> None        # 危险色按钮
ui.dialogs.set_parent(widget) -> None        # 主窗口创建后回填父窗口
```

- 全部自绘（`_CardDialog`），样式只来自全局 QSS 的 objectName（`Card` / `PageTitle` / `Danger` / `Secondary`）。
- 统一最小宽度 420、按钮顺序「取消在左、确认在右」、ESC 关闭。
- 阻塞式模态，所以调用点不需要嵌套事件循环；返回值是语义结果（`confirm` / `cancel` / `close`）。
- 规划：`prompt` 输入框、`toast`、可取消 `progress` 对话框、5 秒错误去重。

### 6.3 布局（已实现）

- `ShellWindow`：左侧 `QListWidget` 导航（220px）+ 右侧 `QStackedWidget`，导航点击发 `nav_requested(str)`。
- `LayoutService` 延迟创建主窗口（`create_shell` 幂等），并提供 `add_page` / `set_route` / `shell` 属性。
- 路由切换由 `PageRouter` 驱动，`ShellWindow.set_route` 期间 `blockSignals`，避免回环。
- 规划：`PageLayout(header/content/footer)` 三段式、`SinglePage` / `SidebarPage` / `MasterDetail` / `FormGrid` 预设、断点统一下发、DPI 缩放策略。

### 6.4 页面与路由（已实现）

```python
ui.pages.register(route, title, factory)   # 工厂延迟实例化，重复路由抛 ValueError
ui.pages.navigate(route) -> QWidget        # 首次访问才构建控件
ui.pages.page(route) -> QWidget | None     # 未访问返回 None
ui.pages.routes() -> tuple[str, ...]
ui.pages.bind_shell()                      # 把 nav_requested 接到 navigate，只连一次
```

- `PageRouter(QObject)` 发 `navigated(str)`，供面包屑、日志或埋点使用。
- 页面基类当前是 `QWidget`；规划中的 `Page` 生命周期钩子（`on_create` / `on_enter` / `on_leave` / `on_dispose`）尚未落地。

### 6.5 生命周期（已实现）

```python
ui.lifecycle.register_cleanup(fn, name="...")  # 后进先出
ui.lifecycle.bind(app)                         # 挂 aboutToQuit
ui.lifecycle.shutdown()                        # 幂等，单个失败只记录
ui.lifecycle.shutting_down                     # 是否已进入退出流程
```

- 逆序执行清理，单个异常不阻断其余清理。
- 已进入退出流程后再注册会抛 `RuntimeError`，防止退出期间新增资源。
- `UiManager.shutdown()` 先跑清理钩子，再等待线程池（3s 宽限）。

### 6.6 线程池（已实现）

```python
ui.tasks.submit("io", fn, on_done=cb, on_error=cb)   # 位置参数：池名 + 可调用
ui.tasks.shutdown(wait_ms=3000)
```

- 三个命名池：`io`(4) / `serial`(1) / `cpu`(2)，各自独立 `QThreadPool`。
- 任务是 `QRunnable`，异常在线程内捕获，统一转成 `failed(str)` 信号，绝不向池抛异常。
- 每次提交创建独立 `_Signals(QObject)`，由服务持有引用直到回调完成，避免信号对象被 GC。
- 回调在主线程执行（Qt 队列连接），业务侧无需 `QMetaObject.invokeMethod`。
- 池名未知抛 `KeyError`；关闭时 `waitForDone(wait_ms)`。
- 规划：协作式取消（`CancellationToken`）、进度回传、同 key 去重、有界队列拒绝策略。

### 6.7 两个必须提前处理的坑

1. SQLite 跨线程：每任务独立 Session，开启 WAL + `busy_timeout`，写操作走 `serial` 池。
2. Qt 线程亲和：跨线程通信只经 `kit/tasks.py` 的 `done` / `failed` 信号回主线程，禁止 worker 直接改 UI 状态或触碰 QWidget。
