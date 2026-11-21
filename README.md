# DGUIApp 开发脚手架技能

一套给 AI 编码代理使用的 **DGUIApp（PySide6）桌面应用工程规范**，以 OpenCode / Claude 兼容的 Skill 形式分发。

技能本身不包含可执行代码，而是把已经定稿的工程约定沉淀成可检索的规则：Copier 模板、横纵分层（`platform/` + `features/`）、ports/adapters、SQLite 可切 MySQL、MVVM + UI Runtime、严格代码规范、质量门禁、DX 一键启动与热重载，以及下载慢时的镜像切换策略。

## 这个技能解决什么

- 从零生成 PySide6 桌面项目时，避免每次重新讨论目录、依赖方向和启动链。
- 在已有项目里新增业务切片、页面、Alembic 迁移或适配器时，给出固定落点。
- 审查项目是否违反分层契约、样式令牌、线程模型或质量门槛。
- 工程开始前与开发中，依赖、解释器、模板或工具下载变慢时，按统一阈值判定并立即切换镜像，留下可复现记录。
- 维护 Copier 模板本身：参数、profile、渲染矩阵与升级迁移。

不适用于 Web、移动端、服务端项目，也不适用于 Tkinter / wxPython 等非 PySide6 GUI。

## 安装

把仓库放进代理的技能目录即可。OpenCode 全局技能目录：

```text
~/.config/opencode/skills/dgui-app-scaffold/
├── SKILL.md
└── references/
```

也可以放在项目的 `.opencode/skills/dgui-app-scaffold/` 下，只对当前项目生效。

```bash
git clone https://github.com/SC-CATL/dgui-app-scaffold.git \
  ~/.config/opencode/skills/dgui-app-scaffold
```

安装后技能 ID 为 `dgui-app-scaffold`，显示名为 `DGUIApp开发脚手架`。

## 使用

直接对代理描述任务即可，代理会按 `description` 自动发现并加载；也可以显式引用：

```text
@dgui-app-scaffold 帮我新建一个 PySide6 桌面项目
@dgui-app-scaffold 审查这个项目是否符合分层与门禁
```

技能入口是 `SKILL.md`，其余细节按需读取 `references/`，不会一次性塞进上下文。

## 技能结构

```text
.
├── SKILL.md                     # 入口：流程、红线、任务路由、验收清单
└── references/
    ├── architecture.md          # 目录、依赖方向、切片骨架
    ├── ui-runtime.md            # 页面、弹窗、主题、任务、生命周期
    ├── code-standards.md        # 函数与写法规范
    ├── quality-gates.md         # CI、门禁、发布判断
    ├── database.md              # 建表、迁移、ORM、事务
    ├── lifecycle.md             # 生成记录、审查报告、归档
    ├── dx.md                    # 一键启动、热重载、生成器
    ├── template-spec.md         # Copier 参数、渲染矩阵、实施路线
    ├── qt-binding.md            # PySide6 / PySide2 兼容与降级
    └── mirrors.md               # 下载慢速判定、镜像与离线缓存
```

## 核心约定（摘录）

- `features/*` 可以依赖 `platform/*`，反向禁止；跨切片只走 `api.py` 或事件总线。
- Qt 绑定只在 `platform/ui/qt.py` 导入，其他文件出现 `PySide6` / `PySide2` 即违规。
- 页面不写颜色与样式，统一走 `resources/tokens.toml`。
- UI 线程不做 IO，跨线程只经 `platform/ui/kit/tasks.py`。
- 每种持久化数据都有对应的表；`QSettings` 只存引导信息。
- 下载源慢时立即切镜像：单文件低于 200 KB/s 持续 10 秒或 15 秒无进度即判定为慢，切换后仍用 `uv sync --locked` 保持版本与哈希。

完整规则见 `SKILL.md` 与对应 reference。

## 贡献

见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 许可证

[MIT](LICENSE)
