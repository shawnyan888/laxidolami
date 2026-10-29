# AGENTS.md

<!-- 示例：Python 后端项目（FastAPI + PostgreSQL）。
     HTML 注释是写给你看的讲解，实际项目中应删掉。 -->

FastAPI + PostgreSQL + Redis 的订单服务。包管理用 Poetry，Python 3.11。

<!--
第一段 = 一句话项目定位。
AI 第一眼就知道技术栈和包管理器，避免猜错（比如误用 pip/requirements.txt）。
注意没有罗列目录结构——AI 自己会 ls，写了就是浪费 token。
-->

## 命令

```bash
# 安装依赖（首次 / poetry.lock 变更后）
poetry install

# 启动开发服务器（热重载）
poetry run uvicorn app.main:app --reload --port 8000

# 运行全部测试（自动拉起临时 Postgres 容器）
poetry run pytest

# 只跑单个文件 / 单个用例
poetry run pytest tests/test_orders.py -v
poetry run pytest tests/test_orders.py::test_create_order -v

# 代码风格检查与修复（提交前必须通过）
poetry run ruff check app tests
poetry run ruff check --fix app tests

# 数据库迁移
poetry run alembic revision --autogenerate -m "描述"
poetry run alembic upgrade head
```

<!--
命令段的核心标准：每条都能原样复制到终端执行。
"提交前必须通过"是可验证的完成标准，比"注意代码风格"有效得多。
-->

## 架构

- `app/api/` 路由层：只做参数校验和调用 service，不写业务逻辑、不直接访问数据库。
- `app/services/` 业务层：所有业务逻辑在这里，通过 repository 访问数据。
- `app/repositories/` 数据访问层：SQLAlchemy 查询，一个表一个 repository。
- `app/models/` SQLAlchemy 模型；`app/schemas/` Pydantic 模型，二者不互相 import。

依赖方向固定：api → services → repositories → models，禁止跨层或反向 import。

<!--
架构段讲的是"查不到的约定"：分层的职责边界和依赖方向。
模型与 schema 互不 import 是真实项目里高频踩的坑，值得占一行。
-->

## 数据库约定

- 表名与列名用 snake_case，表名用复数（`orders`、`order_items`）。
- 修改模型后必须生成 Alembic 迁移文件，禁止手写 `ALTER TABLE` SQL 去改线上库。
- 迁移文件一旦合入 main 不可修改，只能新增。

## 测试约定

- 测试文件镜像 `app/` 目录结构：`app/services/orders.py` 对应 `tests/test_services/test_orders.py`。
- 用 pytest fixture 注入测试数据库会话，禁止测试直连开发库。
- 新增接口必须有正常路径 + 参数校验失败 + 业务异常三类用例。

<!--
"三类用例"给了明确的需求下限——AI 写测试时会照此展开，而不是只写一个 happy path。
-->

## 安全红线（硬性）

- 禁止把密钥、连接串提交进仓库，一律走环境变量（`.env` 已在 .gitignore）。
- 禁止在迁移或代码中删除用户数据表；删除字段用"先停用、下版本再删"两步走。

<!--
红线段是禁令的合法位置：真正的不可逾越项。
只有两行，因为禁令会"唤醒"被禁止的行为，非红线的偏好都用上面的正面表述。
-->

## 相关文档

- 接口鉴权与 Token 刷新细节见 `docs/auth.md`——改动登录相关代码前先读。
- 性能问题排查步骤见 `docs/performance.md`——处理慢查询、内存上涨时参考。

<!--
指针式引用：低频场景的细节放到独立文档，主文件只留一句
"什么情况下读它"。触发条件写在句首（"改动登录相关代码前"），
AI 才能在对的时机去读。
-->
