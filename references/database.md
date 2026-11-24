# 数据库设计规范

适用范围：模板仓库与所有生成项目。双库：SQLite（本地）/ MySQL 8（同步与集成测试）。

## 0. 核心原则

**每种数据都要有对应的表。**

持久化的业务数据、状态、历史、配置，一律落库成表，不允许散落在 `QSettings`、INI、JSON 文件、注册表或代码常量里。

只允许三种例外，且必须显式登记：

| 例外 | 允许内容 | 约束 |
|---|---|---|
| 引导信息 | 数据库位置、`install_id`、安装根目录 | 只能用于"打开数据库之前"必须知道的信息 |
| 文件本体 | 安装包、日志、崩溃 dump、导出文件 | 必须另有元数据表登记路径、sha256、大小、时间 |
| 密钥明文 | 仅本机 `<用途>-noupdate.*` 文件 | 不得入库、不得提交；业务侧只存 `key_id` 与指纹，运行时优先 Windows DPAPI 或系统密钥库 |

判断标准：这个值如果需要跨会话保留、需要查询、需要审计、需要同步，就必须是表。

开源敏感配置（API Key、Token、私有连接串）在本机文件里必须使用 `-noupdate` 后缀并保持被 git 忽略，详见 `architecture.md` §7.1。

### 0.1 当前实现状态

模板当前只生成一张业务表 `app_settings`（`key` 主键 + `value`，公共列 `created_at` / `updated_at`）。本文件其余内容是目标规范：新建表时必须遵守，但以下两条与现状不同，需按实际情况处理：

- 「主键统一 `id`」是默认；键值型配置表允许用自然键（如 `app_settings.key`）作主键，避免无意义的代理键。
- `version`（乐观锁）与 `deleted_at`（软删除）是**按需列**，不是每张表都必带；只有需要并发更新或多端同步的表才加。

## 1. 初始表清单

下表是目标形态，`已实现` 列标明当前模板是否生成：

| 数据 | 表 | 归属 | 已实现 |
|---|---|---|---|
| 应用设置 | `app_settings` | `features/settings` | 是 |
| 窗口与布局状态 | `window_state` | `features/settings` | 否 |
| 最近打开 | `recent_items` | `features/settings` | 否 |
| 首启向导进度 | `onboarding_state` | `features/onboarding` | 否 |
| 用户 | `users` | `features/users` | 否 |
| 角色 / 权限 | `roles`、`permissions`、`user_roles`、`role_permissions` | `features/users` | 否 |
| 登录会话 | `sessions` | `features/users` | 否 |
| 任务与队列 | `tasks`、`task_events` | `platform/ui/kit/tasks`（规划持久化） | 否 |
| 同步变更日志 | `sync_outbox` | `features/sync` | 否 |
| 同步游标 | `sync_state` | `features/sync` | 否 |
| 更新状态 | `update_state` | `features/updates` | 否 |
| 崩溃报告元数据 | `crash_reports` | `platform/observability` | 否（当前 dump 写文件） |
| 功能开关 | `feature_flags` | `platform/config` | 否 |
| 审计日志 | `audit_log` | `platform/observability` | 否 |
| 迁移版本 | `alembic_version` | Alembic 自动维护 | 是 |

新增数据时必须先回答三个问题：归属哪个切片、主键怎么定、保留多久。答不上来不允许建表。

## 2. 命名

| 对象 | 规则 | 示例 |
|---|---|---|
| 表 | snake_case 复数 | `users`、`sync_outbox`、`task_events` |
| 关联表 | 两表名按字母序，单数形式 | `user_roles`、`role_permissions` |
| 字段 | snake_case | `display_name`、`last_login_at` |
| 主键 | 统一 `id` | `id` |
| 外键 | `<单数实体>_id` | `user_id`、`task_id` |
| 布尔 | `is_` / `has_` / `can_` | `is_active`、`has_synced` |
| 时间 | `_at`（UTC） | `created_at`、`deleted_at` |
| 日期 | `_on` | `birth_on` |
| 金额 | `_minor`（最小货币单位） | `amount_minor` |
| 索引 | `ix_<表>_<列>` | `ix_tasks_status` |
| 唯一 | `uq_<表>_<列>` | `uq_users_email` |
| 检查 | `ck_<表>_<列>` | `ck_tasks_status` |
| 外键 | `fk_<表>_<引用表>` | `fk_tasks_users` |

禁止：

- 表名加前缀（`tbl_`、`t_`）。
- 字段拼音或缩写（`yh_id`、`dt`）。
- 保留字：`order`、`group`、`user`、`key`、`value`、`index`。
- 复数与单数混用同一套命名。

## 3. 主键与标识

- 本地主键默认 `id INTEGER PRIMARY KEY`（SQLite 自增 / MySQL `BIGINT AUTO_INCREMENT`）；键值型配置表可用自然键。
- 需要跨端同步或对外暴露的实体，额外加 `uid CHAR(26)` 存 ULID，`UNIQUE NOT NULL`。
- 业务编号（如订单号）另设 `code VARCHAR(32)`，不与主键混用。
- 禁止用业务字段当主键。
- 禁止复合主键，关联表除外（关联表也建议单独 `id`）。

## 4. 列类型

| 语义 | SQLAlchemy 类型 | 说明 |
|---|---|---|
| 主键 | `Integer` / `BigInteger` | 自增，无符号问题由 Alembic 处理 |
| ULID | `String(26)` | 大写 Crockford Base32 |
| 短文本 | `String(n)`，n ≤ 255 | 必须显式长度 |
| 长文本 | `Text` | 不进索引 |
| 布尔 | `Boolean` | 双库均为 0/1 存储 |
| 时间 | `DateTime` | 统一存 UTC，应用写入 |
| 金额 | `BigInteger` + `String(3)` | `amount_minor` + `currency` |
| 枚举 | `String(32)` + CHECK | 禁用 MySQL `ENUM` |
| JSON 扩展 | `JSON` | 仅放无结构扩展，不作查询主路径 |
| 二进制 | `LargeBinary` | ≤ 64KB，超限外置 |

禁止：`FLOAT` / `DOUBLE` 存金额；`DateTime` 存本地时间；`Text` 建索引；用 JSON 代替正常列。

## 5. 公共列

每张表必须有：

| 列 | 类型 | 规则 |
|---|---|---|
| 主键 | 自增 `id` 或自然键 | 必备 |
| `created_at` | `DateTime` | UTC，应用写入，`NOT NULL` |
| `updated_at` | `DateTime` | UTC，应用写入，`NOT NULL` |

按需追加：

| 列 | 类型 | 适用 |
|---|---|---|
| `version` | `Integer` | 乐观锁，默认 1，每次更新 +1；仅并发更新或多端同步的表 |
| `deleted_at` | `DateTime` | 仅按需软删除的表才有，索引 |

不使用数据库的 `ON UPDATE CURRENT_TIMESTAMP`：SQLite 不支持，双库会漂移。

## 6. 约束与完整性

- 每张表必须有主键。
- 外键必须显式声明，并单独建索引。
- 默认策略 `ON DELETE RESTRICT`；需要级联时显式写 `CASCADE`，并在迁移说明里写清影响。
- 列默认 `NOT NULL`；可空必须能说出理由。
- 唯一性用 `UNIQUE` 约束，不靠应用层先查后写。
- 取值范围与枚举用 `CHECK` 约束兜底。
- SQLite 连接必须 `PRAGMA foreign_keys = ON`。

双库差异：MySQL 外键要求被引用列有索引；SQLite 需要显式开 PRAGMA。两者都在 `platform/adapters/db/engine.py` 统一处理。

## 7. 索引

- 每个外键都要有索引。
- 联合索引遵循最左前缀，区分度高的列在前。
- 单表索引不超过 5 个。
- 不给低基数列单独建索引（如 `is_active`）。
- 排序与分页字段纳入索引。
- 慢查询必须用 `EXPLAIN`（MySQL）或 `EXPLAIN QUERY PLAN`（SQLite）验证。

部分索引与软删除的唯一性冲突：SQLite 用部分索引，MySQL 用生成列；两者必须在迁移里同时提供，不允许只在一端生效。

## 8. 软删除

- 默认硬删除。
- 只有用户数据与需要审计的记录使用 `deleted_at`。
- 查询默认过滤 `deleted_at IS NULL`，由仓储层统一封装，业务代码不手写。
- 唯一约束必须与 `deleted_at` 组合，避免"删了不能重建"。
- 日志、同步、任务表禁止软删除。

## 9. 迁移

- 单一 Alembic 版本链，SQLite 与 MySQL 共用。
- 文件名 `YYYYMMDD_<verb>_<object>`，如 `20261116_add_user_roles`。
- 每个迁移必须实现 `downgrade()`，且经过回滚测试。
- 结构与数据迁移分开，不在同一个 revision 里既改表又刷数据。
- SQLite 改列必须用 `batch_alter_table`。
- 已发布的迁移禁止修改，只能追加。
- 迁移里禁止依赖应用模型，使用原生 SQL 或独立表定义。

## 10. ORM 规范

- SQLAlchemy 2.0 typed：`Mapped[int]` + `mapped_column`。
- `Base`、时间戳混入、软删除混入放 `platform/adapters/db/base.py`。
- 每个切片的模型放 `features/<name>/models.py`，与 `repository.py` 分离，避免单文件超 300 行。
- 关系加载显式：`selectinload` / `joinedload`，禁止隐式 lazy load 跨 Session。
- 禁止 `SELECT *`；查询列显式或用完整实体。
- 分页必须带唯一排序键，禁止深 `OFFSET`，超过 1 万行改用键集分页。
- 事务边界在 service 层，`UnitOfWork` 统一提交与回滚。

## 11. 事务与并发

- 每个后台任务独立 Session，禁止跨线程共享。
- SQLite 开启 WAL + `busy_timeout=5000`。
- 写操作走 `serial` 池，保证单写者。
- 事务内禁止网络 IO、文件 IO、弹窗。
- 长事务超过 2 秒记告警。
- 并发更新用 `version` 乐观锁，冲突返回 `Result`，不静默覆盖。

## 12. 数据保留

| 表 | 保留期 | 清理方式 |
|---|---|---|
| `tasks` / `task_events` | 30 天 | 定时任务按 `created_at` 删除 |
| `crash_reports` | 90 天 | 上传成功后按 `created_at` 删除元数据 |
| `audit_log` | 180 天 | 归档后删除 |
| `recent_items` | 30 天 | 按 `opened_at` 截断 |
| `sync_outbox` | 同步成功后 7 天 | 按 `synced_at` 清理 |

- 清理任务必须幂等，且记录执行日志。
- 每月 `VACUUM`（SQLite）/ `OPTIMIZE TABLE`（MySQL）一次。
- 删除前先备份，备份路径进 `archive/`。

## 13. 反例与改法

| 反例 | 问题 | 改法 |
|---|---|---|
| 主题、语言存 `QSettings` | 无法审计、无法同步 | 入 `app_settings` |
| 最近文件存 JSON 文件 | 并发写丢数据 | 入 `recent_items` |
| 任务进度存内存字典 | 重启丢失 | 入 `tasks` + `task_events` |
| 金额用 `FLOAT` | 精度丢失 | `amount_minor BIGINT` |
| 布尔用 `String("Y"/"N")` | 语义模糊 | `Boolean` + CHECK |
| 外键不建索引 | 删除与联查变慢 | 立即补索引 |
| 一张 `config` 表存所有东西 | 无结构、无约束 | 按数据类型拆表 |
| 用 `Text` 存 JSON 并频繁查询 | 无法索引 | 拆成正常列，JSON 只放扩展 |

## 14. 验收清单

- [ ] 每种持久化数据都能指向一张表。
- [ ] 所有表有主键、`created_at`、`updated_at`；需要并发/同步的表另有 `version`。
- [ ] 所有外键有索引，删除策略显式。
- [ ] 没有 `QSettings` / JSON 文件存业务数据。
- [ ] 每个迁移可回滚，且双库测试通过。
- [ ] 金额、时间、布尔、枚举类型符合本规范。
- [ ] 软删除表有 `deleted_at` 索引与唯一约束处理。
- [ ] 保留期与清理任务已登记。
