# 下载与镜像

目标：**任何联网下载都不因源慢而卡住**。工程开始前先选源，开发中一旦判定慢就立即切换，并且可回滚、可追溯、可复现。

本文件覆盖：Python 解释器、PyPI 依赖（含 PySide6 大 wheel）、GitHub 仓库与 Copier 模板、uv 本体；npm 与容器镜像仅在对应可选能力开启时适用。

## 1. 开工前检查（工程开始前必做）

1. 探测三类源：PyPI、GitHub、Python 解释器下载源，各 5 秒超时。
2. 首字节超过 2 秒或不可达，直接标记为慢，为本次会话设置镜像。
3. 有 `uv.lock` 时只用 `uv sync --locked`，切换镜像不得升级版本。
4. 私有依赖确认走私有源或官方源，绝不经第三方代理。
5. 把选源结果写进生成记录（模板见第 8 节）。

探测只代表"可达与首字节"，大文件吞吐在下载阶段按第 2 节复核。

## 2. 慢速判定（统一口径）

满足任一条即判定 **SLOW**，立即进入第 3 节流程：

| 指标 | 阈值 | 判定方式 |
|---|---|---|
| 持续吞吐 | 低于 200 KB/s 且持续 10 秒 | 看实际下载阶段，不看首字节 |
| 无进度 | 15 秒内字节数不增长 | 进度条或日志停住 |
| 单任务耗时 | 超过 60 秒且预计剩余超过 5 分钟 | uv / pip 给出的 ETA |
| 连接失败 | 连续 2 次超时、重置或 5xx | 不再重试第三次 |
| 基线回归 | 全新 `uv sync` 超过 2 分钟 | `dx.md` 的反馈基线 |

动作顺序固定：**暂停 → 探测候选 → 选最快 → 应用 → 重试 → 记录**。

禁止：对同一源连续重试超过 2 次；靠加大超时"等它下完"；在源慢时先跑完整测试再处理。

## 3. 探测候选源

PowerShell（Windows 默认环境）：

```powershell
$sources = [ordered]@{
    'pypi-official' = 'https://pypi.org/simple/pyside6/'
    'tuna'          = 'https://pypi.tuna.tsinghua.edu.cn/simple/pyside6/'
    'aliyun'        = 'https://mirrors.aliyun.com/pypi/simple/pyside6/'
    'ustc'          = 'https://mirrors.ustc.edu.cn/pypi/simple/pyside6/'
    'tencent'       = 'https://mirrors.cloud.tencent.com/pypi/simple/pyside6/'
}
foreach ($name in $sources.Keys) {
    $result = curl.exe -sS -o NUL -L --max-time 5 `
        -w '%{http_code} %{time_starttransfer}' $sources[$name] 2>$null
    '{0,-14} {1}' -f $name, $result
}
```

bash / macOS / Linux：

```bash
for url in \
  https://pypi.org/simple/pyside6/ \
  https://pypi.tuna.tsinghua.edu.cn/simple/pyside6/ \
  https://mirrors.aliyun.com/pypi/simple/pyside6/ \
  https://mirrors.ustc.edu.cn/pypi/simple/pyside6/ \
  https://mirrors.cloud.tencent.com/pypi/simple/pyside6/; do
  curl -sS -o /dev/null -L --max-time 5 -w "%{http_code} %{time_starttransfer} $url\n" "$url"
done
```

选 `200` 且 `time_starttransfer` 最小者；全部不可达时先修网络或代理，不要反复重试。

## 4. 按资源切换

### 4.1 Python 包：uv（首选）

会话级（临时、可回滚）：

```powershell
$env:UV_DEFAULT_INDEX = 'https://mirrors.aliyun.com/pypi/simple/'
uv sync --locked
```

```bash
UV_DEFAULT_INDEX='https://mirrors.aliyun.com/pypi/simple/' uv sync --locked
```

项目级（团队共用，需评审）：

```toml
# pyproject.toml
[[tool.uv.index]]
name = "mirror"
url = "https://mirrors.aliyun.com/pypi/simple/"
default = true
```

规则：

- 用 **default index 整体替换**，不要用 `extra-index-url` 叠加。uv 默认 `first-index` 策略，叠加源既不会自动回退，也增加依赖混淆风险。
- 镜像缺版本时，仅该次命令回官方：`uv sync --locked --default-index https://pypi.org/simple`；不要顺手 `uv lock --upgrade`。
- `uv.lock` 里的哈希是完整性闸门；切换镜像不得关闭校验。
- 首次生成项目尚无 `uv.lock` 时用 `uv sync` 生成；之后一律 `--locked` 或 `--frozen`。

### 4.2 Python 包：pip

```powershell
$env:PIP_INDEX_URL = 'https://mirrors.aliyun.com/pypi/simple/'
python -m pip install -r requirements.txt
```

- 持久化位置：Windows `%APPDATA%\pip\pip.ini`，Linux/macOS `~/.config/pip/pip.conf`。
- 只有旧版工具链才需要 trusted-host；优先修证书，不要绕过 TLS。

### 4.3 Python 解释器（uv managed）

```powershell
$env:UV_PYTHON_INSTALL_MIRROR = 'https://<镜像>/astral-sh/python-build-standalone/releases/download'
uv python install 3.12
```

- 该变量只替换前缀 `https://github.com/astral-sh/python-build-standalone/releases/download`，镜像必须保留后续路径结构。
- 支持 `file://` 本地目录，适合离线或内网分发。
- 没有可信镜像时，优先使用已安装的系统解释器（`uv sync --python 3.12`），不要为了下载解释器等十分钟。
- 安装后核对：`uv python list`，确认版本与架构。

### 4.4 uv 本体与自更新

```powershell
# uv >= 0.11.14：替换 releases.astral.sh 基址
$env:UV_ASTRAL_MIRROR_URL = 'https://<镜像>/releases.astral.sh'
uv self update

# 或只替换 GitHub Releases 基址
$env:UV_INSTALLER_GITHUB_BASE_URL = 'https://<镜像>/astral-sh/uv/releases/download'
```

更新后执行 `uv --version` 确认；版本异常时删除镜像变量重装。

### 4.5 GitHub 仓库与 Copier 模板

优先策略：**先把模板仓库拉到本地，再用本地路径渲染**，避免 Copier 在渲染过程中反复访问远端。

```powershell
git clone --depth 1 https://<镜像>/<owner>/<repo>.git .cache/template
uvx copier copy --trust .cache/template myapp
git -C .cache/template rev-parse HEAD   # 记录模板提交，便于复现
```

- 公共加速域名（如 `ghfast.top`、`gh-proxy.com`、`ghproxy.net`）**随时可能失效或限速**，使用前必须按第 3 节探测；连续失败就换下一个或改用官方源。
- 私有仓库、含 token 的 URL、需要凭证的模板**不得**经过第三方代理。
- 需要让 Copier 直接走镜像时，用命令级 git 重写，不写全局配置：

```powershell
$env:GIT_CONFIG_COUNT   = '1'
$env:GIT_CONFIG_KEY_0   = 'url.https://<镜像>/https://github.com/.insteadOf'
$env:GIT_CONFIG_VALUE_0 = 'https://github.com/'
uvx copier copy --trust gh:<owner>/<repo> myapp
Remove-Item Env:GIT_CONFIG_COUNT, Env:GIT_CONFIG_KEY_0, Env:GIT_CONFIG_VALUE_0
```

- 渲染前核对模板 tag/commit；公共代理传回的内容必须与服务端记录一致，执行 `--trust` 前先看 diff。

### 4.6 可选：npm 与容器镜像

- npm：`npm install --registry https://registry.npmmirror.com`，或项目 `.npmrc` 写 `registry=...`；锁文件不要重写。
- 容器：镜像加速在 Docker Desktop / daemon 的 `registry-mirrors` 配置，属于主机级改动，需要重启守护进程；仅在 `include_docker` 开启时处理。

## 5. 配置落点与回滚

| 落点 | 影响范围 | 典型设置 | 回滚 |
|---|---|---|---|
| 单次命令 | 当前进程 | `uv sync --default-index <url>` | 命令结束即失效 |
| 会话环境 | 当前终端 | `$env:UV_DEFAULT_INDEX` | `Remove-Item Env:UV_DEFAULT_INDEX` |
| 项目配置 | 所有协作者 | `pyproject.toml` / `uv.toml` / `pip.ini` / `.npmrc` | 删除对应段落并提交 |
| 用户全局 | 本机所有项目 | `%APPDATA%\uv\uv.toml`、`~/.config/uv/uv.toml` | 删除对应段落 |

- 优先级：单次命令 > 会话环境 > 项目配置 > 用户全局。
- 会话级优先，其次项目级；**写用户全局必须先告知用户并给出回滚命令**。
- 项目级镜像要写进 README 或生成记录，避免换机器后行为不一致。

回滚示例：

```powershell
Remove-Item Env:UV_DEFAULT_INDEX -ErrorAction SilentlyContinue
Remove-Item Env:UV_PYTHON_INSTALL_MIRROR -ErrorAction SilentlyContinue
git config --global --unset-all 'url.https://<镜像>/https://github.com/.insteadOf'
python -m pip config list          # 确认没有残留 index-url
```

## 6. 安全与一致性

- 只用 HTTPS 且证书有效的镜像；禁用 `--trusted-host`、`UV_INSECURE_HOST`、`UV_NO_VERIFY_HASHES`。
- 企业 TLS 检查导致失败时：`UV_SYSTEM_CERTS=1` 使用系统证书库，或通过 `SSL_CERT_FILE` 指向企业根证书。
- 锁文件哈希不一致时立即停止并排查；不要用"关校验"换速度。
- 私有包、凭证、token 只走官方或私有 index；不要把 token 拼进镜像 URL。
- 镜像可能同步滞后：以 `uv.lock` 的版本为准，缺版本时临时回官方，不做隐式升级。
- 大 wheel（PySide6 通常几十到上百 MB）下载完成后保留缓存，不要为"干净"随意清缓存。

## 7. 离线与团队缓存

```powershell
uv cache dir                        # 缓存位置
$env:UV_CACHE_DIR = 'D:\uv-cache'   # 可选：放更快的盘
uv sync --offline                   # 缓存齐全时断网安装
uv export --format requirements-txt --no-hashes > requirements.txt
python -m pip download -r requirements.txt -d wheelhouse
```

- 内网统一用 devpi / Nexus / Artifactory 做唯一可信源，比每人各配一个公网镜像更稳。
- 共享缓存目录可行，但要评估文件锁与权限；CI 优先缓存 `~/.cache/uv` 并允许用仓库变量覆盖镜像。
- 交付离线包时同时提供 `uv.lock`、`wheelhouse/` 与安装脚本，并校验哈希。

## 8. 记录模板

写入生成记录、PR 描述或故障单：

| 时间 | 资源 | 原源实测 | 切换镜像 | 结果 |
|---|---|---|---|---|
| 2026-11-21 10:20 | PySide6 6.8 wheel | 35 KB/s，ETA 18min | 阿里云 PyPI | 3.1 MB/s，42s 完成 |

最少记录：资源、原源、判定依据（第 2 节哪一条）、镜像、最终耗时。切过镜像但未记录，视为流程未完成。

## 9. 常见错误

| 现象 | 原因 | 处理 |
|---|---|---|
| `uv sync` 长时间无进度 | 源慢或被限速 | 按第 2 节判定并切镜像，不要反复重试 |
| 镜像缺少锁定版本 | 镜像同步滞后 | 该次命令回官方源，保持版本不变 |
| 配了镜像仍然慢 | 镜像本身慢，或代理与镜像叠加 | 重新探测，只保留一层加速 |
| GitHub 加速域名超时 | 公共代理不稳定 | 换镜像或改用官方源，先探测再使用 |
| 切换镜像后依赖版本变了 | 误跑了 `uv lock --upgrade` | 回到 `uv.lock`，用 `--locked` 重装 |
| TLS 证书错误 | 企业证书未被信任 | `UV_SYSTEM_CERTS=1` 或配置企业根证书 |
| 私有包解析到公网同名包 | 叠加了 extra index | 只用单一 default index，私有包固定来源 |
| 解释器下载卡住 | python-build-standalone 源慢 | 设 `UV_PYTHON_INSTALL_MIRROR` 或改用系统解释器 |
| 换机器后行为不一致 | 镜像只配在会话环境 | 需要团队一致时落到项目配置并记录 |
