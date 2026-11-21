# 代码规范：函数 + 写法

强度：严格。正文中文，代码标识英文。行宽 100。

## 0. 总则

1. 可读性优先于技巧。
2. 显式优先于隐式。
3. 单一职责：一个函数只做一件事。
4. 边界清晰：domain 纯、service 编排、adapter IO、ui 渲染。
5. 规范即门禁：每条规则都要有对应的 lint / 类型 / 测试 / 审查。

## 1. 函数规范

### 1.1 规模

- 函数 ≤ 30 行（不含签名与 docstring）。
- 文件 ≤ 300 行；类 ≤ 200 行。
- 参数 ≤ 5 个；超过 3 个使用 `*` 强制 keyword-only。
- 嵌套 ≤ 3 层；超过时用早返回或抽取子函数。
- 圈复杂度 ≤ 10（ruff PLR0912 / PLR0915）。
- 一个函数只在一个抽象层级上工作。

### 1.2 命名

- 动词开头：`get_`、`load_`、`build_`、`parse_`、`validate_`、`ensure_`。
- 用动词区分副作用：
  - `get_x()` 纯读取，不改变状态。
  - `load_x()` 触发 IO。
  - `ensure_x()` 幂等创建。
  - `build_x()` 纯构造。
- 布尔返回用 `is_`、`has_`、`can_`、`should_`。
- 禁止 `handle`、`process`、`do_`、`manage` 等含糊动词。
- 私有函数 `_` 前缀。

### 1.3 参数

- 超过 3 个参数强制 keyword-only（`*`）。
- 禁止布尔位置参数；改为两个具名函数，或 keyword-only + Enum。
- 公共 API 禁止裸 `*args` / `**kwargs`；确有需要必须写明用途与类型。
- 默认值不得为可变对象。
- 依赖通过参数注入，禁止函数内取全局单例。

### 1.4 返回

- 早返回，减少嵌套。
- 返回类型必须显式标注。
- 禁止用 `None` 与错误码混用表达失败。
- 可预期失败返回 `Result[T, E]`；意外失败抛异常。
- 集合无结果时返回空集合，不返回 `None`。
- 不返回可变全局对象。

### 1.5 错误

- 自定义异常继承 `AppError`，携带 `code`、`message`、`context`。
- 禁止裸 `except:` 与 `except Exception: pass`。
- 只在能处理的地方捕获，否则上抛。
- 异常链使用 `raise ... from exc`。
- 同一异常只记录一次，避免层层打印。

### 1.6 副作用

- 纯逻辑与 IO 分离：切片内不新建 `domain.py`，纯函数放 `service.py` 顶层或 `platform/core/`；无 IO、无全局状态、不直接读时间（通过 Clock 注入）。
- IO 只在 `platform/adapters/*` 与切片 `repository.py` / `service.py`。
- 随机数、时间、环境变量通过参数注入。
- 不修改传入的可变参数，除非函数名明确 `_inplace`。

### 1.7 docstring（Google 风格）

公共函数必须写：一句话摘要 + Args + Returns + Raises + Example。私有函数可省略，但复杂逻辑必须有 why 注释。摘要用祈使句，以句号结尾。

```python
def load_user(user_id: int, *, include_roles: bool = False) -> Result[User, UserNotFound]:
    """加载用户。

    Args:
        user_id: 用户主键。
        include_roles: 是否同时加载角色。

    Returns:
        成功返回 User，失败返回 UserNotFound。

    Raises:
        DatabaseError: 数据库不可用时抛出。

    Example:
        >>> load_user(1).unwrap().name
        'alice'
    """
```

### 1.8 反例与改法

| 反例 | 问题 | 改法 |
|---|---|---|
| `def process(data, flag, timeout, retry)` | 含糊 + 位置布尔 | `def sync_users(*, source, dry_run, timeout_s, retries)` |
| `except Exception: pass` | 吞异常 | 上抛，或记录后转 Result |
| `def get_data()` 内部写文件 | 命名与副作用不符 | 改名为 `load_report()` |
| 5 层 if 嵌套 | 难读 | 早返回 + 提前校验 |
| 返回 `None` 表示失败 | 语义不清 | `Result[T, E]` |

## 2. 写法规范

### 2.1 格式

- 行宽 100；4 空格缩进；UTF-8；LF。
- 导入顺序：标准库 / 三方 / 本项目；每组内字母序；禁止通配导入。
- 一行一个语句；禁止分号。
- 括号与换行由 `ruff format` 统一，不手工调整。

### 2.2 类型

- 公共与私有函数都要标注参数与返回类型。
- 用 `X | None` 而非 `Optional[X]`；用 `list[str]` 而非 `List[str]`。
- 禁止裸 `Any`；必要时 `# type: ignore[code]` 并写明原因。
- 领域模型用 `@dataclass(frozen=True)` 或 Pydantic，不用裸 dict 传业务数据。
- 类型别名使用 `TypeAlias`。

### 2.3 变量与常量

- 变量名用名词，函数名用动词。
- 常量 UPPER_SNAKE，集中在 `constants.py` 或模块顶部。
- 禁止魔法数字，提取为具名常量。
- 变量靠近使用处声明，缩短生命周期。
- 禁止可变默认参数与模块级可变状态。

### 2.4 控制流

- 早返回；正常路径保持最左缩进。
- 复杂布尔表达式提取为具名变量。
- 禁止 `if x == True`。
- 循环内避免重 IO；批量处理。
- 多分支用 `match` 或策略表，不用长 if-elif 链。

### 2.5 异常与日志

- 异常只用于意外；业务分支用 Result。
- 日志使用 structlog 键值：`log.info("user.loaded", user_id=user_id)`。
- 禁止 print；禁止字符串拼接日志。
- 敏感字段（密码、token、身份证）必须脱敏。
- 同一异常只记录一次，避免重复噪音。

### 2.6 注释

- 注释解释 why，不解释 what。
- 公共 API 用 docstring；行内注释用 `#` 加空格开头。
- 禁止注释掉的死代码，用 git 历史追溯。
- TODO 必须带负责人与 issue：`# TODO(alice, #123): ...`。
- 中文注释允许，代码标识保持英文。

### 2.7 类与模块

- 一个模块一个主要职责；模块名用名词。
- 类 ≤ 200 行；超过拆分。
- 优先组合而非继承；继承不超过 2 层。
- 数据类用 `@dataclass(frozen=True, slots=True)`。
- 禁止在 `__init__` 里做 IO。

### 2.8 测试写法

- AAA：Arrange / Act / Assert，用空行分隔。
- 命名 `test_<行为>_<条件>_<期望>`。
- 一测一关注点；断言失败信息可读。
- fixture 放 conftest，作用域最小。
- 禁止 sleep；用 fake clock 或事件等待。
- 禁止跨测试共享可变状态。

### 2.9 Git 写法

- Conventional Commits：`type(scope): subject`。
- type：feat / fix / refactor / test / docs / chore / perf / build / ci。
- subject 用祈使句、≤ 72 字符、不加句号。
- 一个提交一件事；PR ≤ 400 行有效改动。
- 分支：`feat/<slice>-<desc>`、`fix/<issue>-<desc>`。

## 3. 工具映射

| 规范 | 工具 / 规则 | 状态 |
|---|---|---|
| 行宽 / 格式 | ruff format | 已接入 |
| 命名 | ruff N | 已接入 |
| 未使用 / 导入 | ruff F | 已接入；vulture 未接入 |
| 复杂度 / 长度 | ruff PLR0912、PLR0915 | 已接入 |
| 类型 | mypy strict、ruff ANN | 已接入 |
| docstring | ruff D | 已接入；interrogate 未接入 CI |
| 安全 | ruff S | 已接入；bandit 未接入 |
| 切片边界 | import-linter + AST 测试 | 已接入 |
| 重复代码 | jscpd ≤ 3% | 规划 |
| 提交信息 | commitlint | 规划 |
| 测试 | pytest、pytest-cov | 已接入 |

## 4. 评审对照表

- [ ] 函数 ≤ 30 行、文件 ≤ 300 行、类 ≤ 200 行、参数 ≤ 5、嵌套 ≤ 3
- [ ] 参数超 3 个已用 keyword-only
- [ ] 返回类型显式，失败用 Result
- [ ] 无裸 except、无吞异常
- [ ] 纯逻辑无 IO，IO 只在 adapters / repository / service 的边界函数
- [ ] 公共 API 有 Google docstring
- [ ] 无魔法数字、无可变默认参数
- [ ] 日志结构化、敏感字段脱敏
- [ ] 测试 AAA、命名规范
- [ ] 提交符合 Conventional Commits

## 5. 生效方式

- 生成项目当前只带 `docs/index.md`（`include_docs=true`）；完整 `docs/dev/code-style.md` 正文按需补，规范以本文件为准。
- 规范变更走 ADR + 模板 PR，并附门禁变更说明。
- 每条新规范必须先有自动检查，再加文档。
