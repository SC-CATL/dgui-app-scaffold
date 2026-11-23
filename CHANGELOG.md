# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [Unreleased]

## [0.3.0] - 2026-11-23

### Added

- 生成项目新增 `.vscode/settings.json`、`launch.json`、`extensions.json`：默认解释器固定为项目 `.venv`，提供运行与 `--self-test` 两个 debugpy 配置。
- `main.py` 增加解释器兜底：非项目 `.venv` 时自动 `os.execv` 切换，解决系统 Python 直接运行导致的 `ModuleNotFoundError: structlog/PySide6`。
- `SKILL.md` 红线、验收清单、常见错误与输出要求同步补充解释器与编辑器配置。

## [0.2.0] - 2026-11-23

### Added

- 新增 `references/mirrors.md`：工程开始前与开发中的下载慢速判定（吞吐、无进度、超时、基线回归）、PyPI / 解释器 / uv 本体 / GitHub 模板的镜像切换、配置落点与回滚、离线缓存与记录模板。
- `SKILL.md` 增加「下载与镜像」硬性流程、任务路由与常见错误；`dx.md` 增加下载加速小节。
- GitHub Issue / PR 模板，以及 `check` 工作流：校验 frontmatter、必备文件与相对链接。
- `THIRD_PARTY_NOTICES.md`（Contributor Covenant 署名）与 `CITATION.cff`。

## [0.1.0] - 2026-11-21

### Added

- 首次公开发布。
- 技能入口 `SKILL.md`：生成流程、锁定参数、架构红线、代码硬指标、质量门禁、任务路由与验收清单。
- 九份 reference：架构、UI Runtime、代码规范、质量门禁、数据库、生命周期、DX、模板规格、Qt 绑定兼容。
- MIT 许可证、README、贡献指南与 `.gitignore`。
