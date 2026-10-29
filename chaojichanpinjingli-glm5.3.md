# 软件开发环境 Specification（初版）

| 项目 | 内容 |
| --- | --- |
| 文档版本 | v0.1（初版） |
| 文档状态 | Draft / 待评审 |
| 作者 | 超级产品经理 · 产出引擎 |
| 适用范围 | 全体研发团队（后端 / 前端 / 测试 / DevOps） |
| 变更方式 | 通过 Merge Request 修改本文档，至少 1 名架构组成员 Approve |

---

## 0. 文档目的

本文档定义团队统一的软件开发环境规范，覆盖代码托管平台选型、GitLab 组织结构与命名、分支与提交信息规范、CI/CD 流水线任务定义、邮件提醒规则、VSCode 调试环境与插件配置、以及基于 Makefile 的构建脚本约定。

所有工程行为（建仓、建组、开分支、提交、触发流水线、配置本地环境、编写构建脚本）均以本文档为准；与本文档冲突的个人习惯视为不合规。

---

## 1. 为什么选择 GitLab

### 1.1 结论

团队统一选用 **GitLab 自托管版本** 作为代码托管与 DevOps 平台，兼顾私有化部署的安全要求与开箱即用的 CI/CD 能力。

### 1.2 选型理由

#### （1）CI/CD 与代码托管一体化，降低维护成本
- GitLab 内置 GitLab CI/CD，通过仓库根目录一个 `.gitlab-ci.yml` 即可定义完整流水线，无需单独部署、对接 Jenkins / TeamCity 等外部 CI 系统。
- Pipeline 与 Merge Request、分支、Tag 天然关联，Pipeline 状态直接展示在 MR 界面上，无需插件或二次开发。

#### （2）内建代码评审与协作流程
- Merge Request 是一等公民：Approve 规则、行级评论、讨论线程、Draft MR、Squash & Merge 全部原生支持。
- 权限模型（Guest / Reporter / Developer / Maintainer / Owner）粒度到 Group / Subgroup / Project 三级，方便按团队与角色授权。

#### （3）私有化部署满足数据安全与合规要求
- 自托管实例使核心代码不出内网，满足公司对源代码资产的安全与审计要求。
- 支持备份、恢复、审计日志（Audit Events）、IP 白名单等企业级能力。

#### （4）对 Merge Request 的强制门禁
- 可配置「Pipeline 必须通过才能 Merge」「必须获得 N 个 Approve 才能 Merge」，从流程上杜绝未测试代码进入主干。

#### （5）替代方案对比

| 维度 | GitLab（自托管） | GitHub Enterprise | Gitee 企业版 | Jenkins + Git |
| --- | --- | --- | --- | --- |
| CI/CD 内置 | 原生内置，`.gitlab-ci.yml` 即可 | 需 Actions，自托管 Runner 较繁琐 | 支持，定制能力一般 | 需单独维护 Jenkins 及插件 |
| 私有化部署 | 成熟 | 成熟但成本高 | 支持 | 需自行拼装 |
| MR 评审流程 | 原生，规则可强制 | 原生 | 原生 | 需插件 |
| 运维成本 | 单一平台，低 | 中 | 低（但数据在第三方） | 高（双系统） |
| 与内网部署/审计契合 | 高 | 中 | 低 | 中 |

**结论**：GitLab 在「一体化 CI/CD + 私有化 + 流程强制」三点上的综合得分最高，故选之。

---

## 2. GitLab Group / Subgroup / Project 命名规范

### 2.1 总体结构

```
<公司级顶层 Group>                # 例：laxidolami
 ├── <业务线 Subgroup>            # 例：platform
 │    ├── <服务 Project>          # 例：platform/user-service
 │    └── <服务 Project>          # 例：platform/order-service
 ├── <公共组件 Subgroup>          # 例：common
 │    └── <组件 Project>          # 例：common/logger-lib
 └── <基础设施 Subgroup>          # 例：infra
      └── <Project>               # 例：infra/ci-templates
```

### 2.2 命名规则

| 对象 | 规则 | 示例 |
| --- | --- | --- |
| 顶层 Group | 全小写，公司/组织英文标识，唯一 | `laxidolami` |
| Subgroup | 全小写，kebab-case，按业务域或职能划分 | `platform`、`common`、`infra`、`data` |
| Project | 全小写 kebab-case；服务类以 `-service` 结尾，库类以 `-lib` 结尾 | `user-service`、`logger-lib` |
| 项目内部仓库用途 | 前端项目以 `-web` / `-admin` 结尾；文档类以 `-docs` 结尾 | `portal-web`、`team-docs` |

### 2.3 约束与约定

1. **禁止**使用中文、拼音缩写无说明、大写字母、下划线（`_`）作为 Group / Project 名。
2. Project 必须挂在正确的 Subgroup 下；跨业务线共用的代码放 `common`，禁止复制多份。
3. 新建 Group / Project 需由 Maintainer 以上角色执行，并在群公告或 Wiki 中登记用途、Owner、技术栈。
4. 每个 Project 的 **Description 必填**（一句话说明用途），**README 必须包含**：项目简介、本地启动方式、构建命令（`make` 目标）、部署说明。
5. 默认分支统一为 `main`。
6. Project 可见性：默认 `Private`；如需开源内部组件，在 Group 层面统一定 `Internal`，不允许个人私自改 `Public`。

---

## 3. 分支命名与 Commit Message 规范

### 3.1 分支模型

采用简化版 Git Flow：长期分支仅保留 `main`（受保护），其余分支均为短生命周期分支，合入即删。

| 分支 | 用途 | 来源 | 生命周期 |
| --- | --- | --- | --- |
| `main` | 随时可发布的稳定主干 | — | 永久，受保护 |
| `feature/*` | 新功能开发 | `main` | 合入 MR 后删除 |
| `bugfix/*` | 缺陷修复 | `main` | 同上 |
| `hotfix/*` | 生产紧急修复 | 生产版本 tag | 修复后同时合回 `main` |
| `release/*` | 版本发布前的冻结与验收 | `main` | 发布打 tag 后删除 |
| `chore/*` | 构建/依赖/文档等杂项 | `main` | 合入后删除 |

### 3.2 分支命名规范

格式：`<类型>/<issue 编号>-<短横线小写描述>`

```
feature/1234-user-login
bugfix/1301-fix-token-expiry
hotfix/1409-payment-timeout
release/2.3.0
chore/1500-upgrade-go-version
```

规则：
1. 全小写，单词间用 `-`，禁止下划线与空格；
2. 有 issue 的必须带 issue 编号，便于 MR 自动关联与追溯；
3. 描述控制在 3～5 个单词以内，能看懂即可；
4. 禁止 `dev`、`test`、`tmp`、个人姓名等无意义常驻分支。

### 3.3 Commit Message 规范

格式（Angular/Conventional Commits 风格）：

```
<type>(<scope>): <subject>

[可选 body]

[可选 footer]
```

#### type 取值

| type | 含义 |
| --- | --- |
| `feat` | 新功能 |
| `fix` | 缺陷修复 |
| `docs` | 仅文档变更 |
| `style` | 代码格式调整（不影响逻辑） |
| `refactor` | 重构（既非 feat 也非 fix） |
| `perf` | 性能优化 |
| `test` | 测试新增/修正 |
| `build` | 构建系统或外部依赖变更（Makefile、Dockerfile 等） |
| `ci` | CI 配置变更（.gitlab-ci.yml） |
| `chore` | 其他杂项 |

#### 规则
1. **subject**：祈使句、小写开头、不加句号，不超过 50 字符；中文场景允许中文，但同一仓库内保持语言统一。
2. **scope**：模块/目录名，如 `auth`、`api`、`makefile`，可省略。
3. **footer**：关闭 issue 用 `Closes #1234`；不兼容变更加 `BREAKING CHANGE: <说明>`。
4. 一个 commit 只做一件事；MR 合入时统一 **Squash**，Squash 后的 commit message 由 MR Title 生成（MR Title 同样遵循本规范）。

#### 示例

```
feat(auth): 支持 OAuth2 手机号登录

新增 /api/v1/auth/mobile 接口，接入运营商一键登录 SDK，
登录成功后签发双 token。

Closes #1234
```

```
fix(order): 修复订单金额为 0 时支付网关报错

金额小于最小支付单位时直接标记订单为已支付，不再调用网关。
```

---

## 4. GitLab CI Pipeline 任务定义

### 4.1 流水线总览

| 触发方式 | 阶段 | 目的 |
| --- | --- | --- |
| Merge Request（MR）触发 | lint → build → unit-test | 门禁：阻止不合规代码合入 |
| push 到 `main` / tag | build → unit-test → package → deploy-dev | 集成与部署 |
| Nightly（定时） | full-test → e2e → security-scan → report | 全量回归与安全扫描 |

### 4.2 MR 触发的 Pipeline

通过 `rules` 控制，仅 MR 阶段运行快速检查，控制时长在 10 分钟以内：

```yaml
stages: [lint, build, unit-test, package, deploy, nightly]

default:
  image: registry.internal/laxidolami/ci-base:1.4

lint:
  stage: lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - make lint

build:
  stage: build
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - make build

unit-test:
  stage: unit-test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - make test
  coverage: '/TOTAL.*\s+(\d+%)%/'
  artifacts:
    paths: [reports/]
    when: always
    expire_in: 1 week
```

**MR 门禁（GitLab 设置，强制生效）**：
- `Pipelines must succeed`：Pipeline 失败禁止 Merge；
- MR 至少 1 名 Reviewer Approve（核心模块 2 名）；
- MR 标题不符合 commit 规范时由 Reviewer 退回。

### 4.3 Nightly 触发的 Pipeline

每晚固定时间执行全量任务，使用 `schedule`（Settings → CI/CD → Schedules，`Cron: 0 19 * * 1-5`，即工作日 19:00，按 UTC 配置时注意时区换算）：

```yaml
nightly-full-test:
  stage: nightly
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - make test-full
  artifacts:
    paths: [reports/, coverage/]
    expire_in: 30 days

nightly-e2e:
  stage: nightly
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - make e2e

nightly-security-scan:
  stage: nightly
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - make security-scan
  allow_failure: true          # 初期不阻断，仅报告
```

Nightly 约定：
1. Nightly Pipeline 失败需在次日上午由轮值同学处理，处理结果回复到对应的告警邮件；
2. Nightly 结果产出报告（测试覆盖率、扫描结果）归档为 artifacts 保留 30 天；
3. 禁止把 Nightly 专用变量（如全量测试 flag）写死在 MR Pipeline 中，统一用 `NIGHTLY=true` 预定义变量区分。

### 4.4 阶段与速度要求

- MR Pipeline 全流程 ≤ 10 分钟（lint ≤ 1 分钟、build ≤ 3 分钟、unit-test ≤ 5 分钟）；超时需优化或拆分。
- 失败任务遵循「红了先修，再提交新代码」原则，禁止连红多日。

---

## 5. 邮件提醒规则

### 5.1 通知源与范围

邮件提醒统一走 GitLab 内建通知（User → Settings → Notifications），团队在项目层面统一约定如下：

| 事件 | 接收人 | 级别 |
| --- | --- | --- |
| MR 被指派为 Reviewer / Assignee | 当事人 | 立即通知 |
| Pipeline 失败（MR 阶段） | 提交人 + MR 作者 | 立即通知 |
| Nightly Pipeline 失败 | 项目 Owner + 轮值值班人 | 立即通知（同时进值班邮件组） |
| MR 获得Approve / 被评论 | 作者与参与者 | 每次通知 |
| 分支被合入 `main` | 项目 Watcher（默认全组） | 摘要通知 |
| 生产 Tag / Release 发布 | 全组邮件组 | 立即通知 |
| 合并冲突导致 MR 不可自动合并 | 作者 | 立即通知 |

### 5.2 团队约定

1. 每人将 GitLab 通知级别设为 **Watch**（本人参与的项目），避免漏看 MR 评审请求；
2. 成立值班邮件组（如 `dev-oncall@company.com`），Nightly 失败邮件抄送该组；
3. Pipeline 失败 30 分钟内响应：作者先自行排查，无法解决则升级到值班同学；
4. 邮件通知**只做提醒不做审批**——所有审批动作仍在 GitLab 界面完成，禁止通过邮件回复处理 MR；
5. 垃圾事件（如个人 fork 的测试流水线）应将通知级别下调为 Participating，减少噪音。

---

## 6. VSCode 调试环境

### 6.1 统一插件清单

| 插件 | 用途 | 是否强制 |
| --- | --- | --- |
| GitLab Workflow（GitLab 官方） | 在 IDE 内查看 MR、Pipeline 状态 | 强制 |
| EditorConfig for VS Code | 统一缩进/编码格式 | 强制 |
| Prettier / 语言对应格式化插件 | 格式化 | 强制 |
| ESLint（或对应 Linter） | 代码检查 | 强制（后端仓库按语言替换） |
| Error Lens | 行内直接显示诊断错误 | 推荐 |
| GitLens | blame / 历史追溯 | 推荐 |
| REST Client / Thunder Client | 接口自测 | 推荐 |
| Docker | 容器调试 | 按需 |
| 各语言官方扩展（如 Go、Python、C/C++、Java 扩展包） | 语言支持与调试器 | 强制（按仓库技术栈） |

> 强制插件随仓库提交 `.vscode/extensions.json`（`recommendations` 字段），新人 clone 后 VSCode 会自动提示一键安装。

### 6.2 统一工作区配置

仓库内提交 `.vscode/settings.json`（示例）：

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.fixAll": "explicit" },
  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "editor.rulers": [100],
  "[go]":   { "editor.defaultFormatter": "golang.go" },
  "[python]": { "editor.defaultFormatter": "ms-python.black-formatter" }
}
```

### 6.3 调试方法（debug / 断点 / 查看内存等）

#### （1）launch.json 统一约定

调试配置随仓库提交于 `.vscode/launch.json`，入口统一命名为 `Debug: <模块名>`，禁止使用「临时乱配的默认配置」提交。

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug: user-service",
      "type": "go",                       // 按语言替换：python / cppdbg / node 等
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/user-service",
      "args": ["--config", "config/dev.yaml"],
      "env": { "ENV": "dev" },
      "preLaunchTask": "make-debug-build" // 复用 Makefile 构建产物
    },
    {
      "name": "Attach: user-service (remote)",
      "type": "go",
      "request": "attach",
      "processId": "${command:pickProcess}"
    }
  ]
}
```

#### （2）断点使用规范

- **行断点**：日常调试默认手段，点击行号左侧空白即设；
- **条件断点**：右键断点 → Edit Breakpoint，用于循环/高频路径（如 `i == 1024`、`order.Amount > 10000`），避免每次命中都停；
- **日志断点（Logpoint）**：不打断执行的「打点日志」，用于不便停止的线上复现型调试；
- **函数断点**：Debug 面板 Breakpoints 区 + 号输入函数名，适合入口函数排查；
- **命中计数**：配合 Hit Count 表达式（如 `>= 100`）定位间歇性问题。

#### （3）运行时数据查看

| 需求 | 操作 |
| --- | --- |
| 变量 / 表达式 | Variables 面板查看；Watch 面板添加任意表达式（含函数调用） |
| 调用栈 | Call Stack 面板逐帧切换，结合面板旁的「已停用帧数据」提示 |
| 内存查看 | Debug Console 输入语言对应的内存检查命令（如 C/C++ 用 `&var`、`x/16xb ptr`；Go/Python 一般通过 Data Breakpoint 与对象展开代替裸内存） |
| 数据断点（值变化即停） | 变量上右键 → Break on Value Change / Read，追踪「谁改了这个值」 |
| 多线程 | Debug 工具栏 → Threads 面板，冻结/切换线程定位竞态 |

#### （4）调试纪律

1. 断点、`launch.json` 中个人临时的配置**不得提交**（用 `launch.local.json` 或提交前还原）；
2. 优先单测复现 + 断点定位，避免依赖打印大法污染代码；
3. 调试远程环境（测试服容器内进程）用 Attach 模式，端口仅对内网开放，调试结束立即断开；
4. 禁止在共享分支上提交仅为调试服务的代码改动。

---

## 7. 基于 Makefile 的构建脚本

### 7.1 设计原则

1. **Makefile 是唯一构建入口**：本地开发、CI 流水线、发布脚本统一通过 `make <target>` 完成构建，保证「本地能跑通，CI 就能跑通」；
2. 仓库根目录必须提供 Makefile，`make help` 列出全部可用目标；
3. Target 命名全小写、用 `-` 分隔（如 `unit-test`），禁止与内置目标冲突；
4. 复杂逻辑放到 `scripts/` 目录的独立脚本，Makefile 只做编排，保持可读。

### 7.2 标准 Target 约定（全仓库统一）

| Target | 作用 | CI 阶段对应 |
| --- | --- | --- |
| `make help` | 打印所有 target 说明 | — |
| `make setup` | 安装依赖、初始化本地环境 | — |
| `make lint` | 代码风格与静态检查 | lint |
| `make build` | 编译/打包出可部署产物 | build / package |
| `make test` | 快速单元测试（CI MR 门禁用） | unit-test |
| `make test-full` | 全量测试（含集成测试） | nightly |
| `make e2e` | 端到端测试 | nightly |
| `make security-scan` | 依赖漏洞/安全扫描 | nightly |
| `make run` | 本地启动（开发用） | — |
| `make debug` | 带调试信息构建（配合 VSCode preLaunchTask） | — |
| `make clean` | 清理构建产物 | — |
| `make docker` | 构建容器镜像 | package |
| `make deploy-dev` | 部署到开发环境 | deploy |

### 7.3 标准模板（骨架）

```makefile
# ============ 基础变量 ============
SHELL        := /bin/bash
.SHELLFLAGS   := -eu -o pipefail -c
.DEFAULT_GOAL := help
BIN          := bin/user-service
PKG          := ./...
VERSION      := $(shell git describe --tags --always --dirty)
BUILD_TIME   := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

# 颜色输出
CYAN  := \033[36m
RESET := \033[0m

.PHONY: help setup lint build test test-full e2e security-scan run debug clean docker deploy-dev

# ============ 目标定义 ============
help: ## 显示帮助
	@awk 'BEGIN {FS = ":.*##"} /^[a-zA-Z_-]+:.*##/ \
	  {printf "  $(CYAN)%-18s$(RESET) %s\n", $$1, $$2}' $(MAKEFILE_LIST)

setup: ## 安装依赖并初始化环境
	scripts/setup.sh

lint: ## 静态检查
	scripts/lint.sh

build: ## 编译产物（注入版本信息）
	CGO_ENABLED=0 go build -ldflags "-X main.version=$(VERSION) -X main.buildTime=$(BUILD_TIME)" -o $(BIN) ./cmd/user-service

test: ## 快速单元测试
	go test $(PKG) -count=1 -coverprofile=reports/cover.out

test-full: ## 全量测试（含集成）
	go test $(PKG) -tags=integration -count=1 -v | tee reports/test-full.log

e2e: ## 端到端测试
	scripts/e2e.sh

security-scan: ## 安全扫描
	scripts/security-scan.sh

run: build ## 本地启动
	./$(BIN) --config config/dev.yaml

debug: ## 带调试信息构建（VSCode 调试用）
	go build -gcflags "all=-N -l" -o $(BIN) ./cmd/user-service

clean: ## 清理产物
	rm -rf bin/ reports/

docker: build ## 构建镜像
	docker build -t registry.internal/laxidolami/user-service:$(VERSION) .

deploy-dev: docker ## 部署开发环境
	scripts/deploy.sh dev
```

### 7.4 编写规则

1. **`.PHONY` 必写**，避免与同名文件/目录冲突；
2. **每个 target 必须有 `## 说明`**，供 `make help` 自动提取；
3. Shell 命令失败必须使 make 失败（模板中 `.SHELLFLAGS` 已带 `-e`），禁止吞错继续执行；
4. 构建产物统一输出到 `bin/`，报告输出到 `reports/`（已被 `.gitignore`，同时作为 CI artifacts 目录）；
5. 版本号从 git tag 派生（`git describe`），禁止手写版本常量；
6. CI 中的 `script` 一律写成 `make xxx`，不允许在 `.gitlab-ci.yml` 里堆砌裸命令——流水线变更走 `ci:` commit，构建逻辑变更走 `build:` commit，二者解耦。

---

## 8. 验收与执行检查清单（Checklist）

新仓库初始化时逐项核对：

- [ ] Group/Subgroup/Project 命名符合第 2 节，Description 已填写
- [ ] `main` 分支已设置保护：禁止直接 push、MR 必须 Pipeline 通过 + Approve
- [ ] `.gitlab-ci.yml` 含 MR 门禁（lint/build/unit-test）与 Nightly 任务
- [ ] Nightly Schedule 已创建并绑定值班邮件组
- [ ] `.vscode/extensions.json`、`settings.json`、`launch.json` 已提交
- [ ] Makefile 含全部标准 target，`make help` 可用
- [ ] `make lint / build / test` 本地可完整跑通
- [ ] README 含启动、构建、部署说明

---

## 9. 后续迭代方向（Roadmap）

| 方向 | 说明 |
| --- | --- |
| v0.2 | 补充各语言仓库的 Makefile 模板细则（Go / Python / Node / C++） |
| v0.2 | 补充 Docker / K8s 本地开发环境（docker-compose、skaffold）规范 |
| v0.3 | GitLab 安全扫描（SAST/DAST/依赖扫描）从「仅报告」升级为「阻断门禁」 |
| v0.3 | 引入 MR 模板与 issue 模板，规范评审上下文 |

---

*本文档由超级产品经理 · 产出引擎基于输入的 7 条要求独立撰写，未参考工作区内的任何既有文件；内容为初版草案，欢迎通过 MR 提出 revision。*
