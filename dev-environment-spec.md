# 软件开发环境规范（初版）

> 文档状态：初版 Draft v0.1
> 适用范围：研发团队所有新建与在维护的代码仓库
> 维护人：平台 / 研发效能组

---

## 目录

1. [为什么选择 GitLab](#1-为什么选择-gitlab)
2. [Group / Subgroup / Project 命名规范](#2-group--subgroup--project-命名规范)
3. [Branch 命名规范与 Commit Message 规则](#3-branch-命名规范与-commit-message-规则)
4. [GitLab Pipeline 任务定义](#4-gitlab-pipeline-任务定义)
5. [Mail 提醒规则](#5-mail-提醒规则)
6. [VS Code 调试环境](#6-vs-code-调试环境)
7. [基于 Makefile 的构建脚本](#7-基于-makefile-的构建脚本)

---

## 1. 为什么选择 GitLab

### 1.1 核心理由

| 维度 | 说明 |
| --- | --- |
| **一体化平台** | 代码托管（SCM）、CI/CD、Issue 跟踪、Code Review、Wiki 全部内建，避免工具链碎片化。 |
| **原生 CI/CD** | GitLab CI 与仓库深度集成，Pipeline 即代码（`.gitlab-ci.yml`），无需另接 Jenkins 等外部系统。 |
| **强代码评审** | Merge Request（MR）支持多轮评审、行内评论、Approval 规则与 Required Reviewers。 |
| **精细权限模型** | 通过 Group / Subgroup 实现层级权限继承，角色从 Guest 到 Owner 五档可控。 |
| **自托管与合规** | 支持私有化部署，满足内网、数据不出域与审计要求。 |
| **可观测性** | Pipeline 状态、安全扫描（SAST/DAST）、产物（Artifacts）统一管理。 |

### 1.2 与其他方案对比（摘要）

- **vs GitHub**：GitHub 更偏开源生态，GitLab 在私有化、内置 CI 与权限层次上更适合企业内研发。
- **vs Gerrit**：Gerrit 评审强但体验偏重、需额外搭 CI；GitLab 开箱即用、学习成本低。
- **vs Bitbucket + Jenkins**：组合方案维护成本高，GitLab 单平台降低运维负担。

### 1.3 结论

统一采用 GitLab 作为唯一代码与持续集成平台，所有仓库、流水线与评审流程均在 GitLab 内闭环。

---

## 2. Group / Subgroup / Project 命名规范

采用「**组织 → 部门 → 团队/产品线 → 项目**」四级结构，保证路径唯一、语义清晰、便于权限继承。

### 2.1 命名规则（通用约束）
- 组织和部门采用公司内部称谓

- 统一使用 **kebab-case**：全部小写字母、数字、连字符 `-`（如 `payment-service`），不使用空格、下划线、中文、大写。
- 名称需具业务语义，避免缩写歧义。
- 路径（path）与显示名（name）保持一致风格。

### 2.2 结构示例

```
<org>                          # Group: 公司/组织顶层
└── <domain>                   # Subgroup: 业务域/部门，如 backend / frontend / platform
    └── <team-or-product>      # Subgroup: 团队或产品线
        └── <project>          # Project: 具体仓库
```

### 2.3 命名方案

```
DeepArray                  # Group: 公司
├── eda-team               # Subgroup: 业务部门
│   ├── fusion             # Subgroup: 产品线
│   │   ├── ace-core       # Project
│   │   ├── ace-agent      # Project
│   │   ├── ace-ui         # Project
│   │   ├── ace-tests      # Project
│   │   └── ace-build      # Project
│   └── next-fusion        # Subgroup: 产品线
│        ....
└── rc-team                # Subgroup: 业务部门
    ├── fusion             # Subgroup: 产品线
    │   ├── core           # 项目
    │   ├── agent          # 项目
    │   ├── ui             # 项目
    │   ├── ace-tests      # 项目
    │   └── ace-build      # 项目  
    └── fusion-next-generation

```


## 3. Branch 命名规范与 Commit Message 规则

### 3.1 长期分支（Protected）

| 分支 | 说明 | 保护策略 |
| --- | --- | --- |
| `main` | 生产可发布分支 | 受保护，仅通过 MR 合并，需 Approvals |
| `develop` | 集成分支（如采用 Git Flow） | 受保护，需 MR 合并 |

### 3.2 临时分支命名

格式：`<type>/<short-description>[-<issue-id>]`

| 类型 | 用途 | 示例 |
| --- | --- | --- |
| `feature/` | 新功能 | `feature/user-onboarding-123` |
| `bugfix/` | 缺陷修复 | `bugfix/login-timeout-456` |
| `hotfix/` | 生产紧急修复 | `hotfix/payment-crash-789` |
| `release/` | 发布分支 | `release/1.4.0` |
| `chore/` | 构建/工具/依赖 | `chore/bump-deps` |
| `docs/` | 文档变更 | `docs/api-readme` |

约束：小写、连字符分隔、避免超过 50 字符、不以 `/` 结尾。

### 3.3 Commit Message 规则（Conventional Commits）

格式：

```
<type>(<scope>): <subject>

<body>            # 可选，解释 why

<footer>          # 可选，如 BREAKING CHANGE / 关联 issue
```

**type 列表**：

| type | 含义 |
| --- | --- |
| `feat` | 新功能 |
| `fix` | 缺陷修复 |
| `docs` | 文档 |
| `style` | 格式（不影响逻辑） |
| `refactor` | 重构 |
| `perf` | 性能优化 |
| `test` | 测试 |
| `build` | 构建系统/依赖 |
| `ci` | CI 配置 |
| `chore` | 杂项 |
| `revert` | 回滚 |

**示例**：

```
feat(auth): support OAuth2 device flow

Enable device authorization grant for CLI login.

Closes #123
```

```
fix(payment): handle nil pointer on timeout

Prevent panic when upstream returns empty response.

Fixes #456
```

约束：subject 不超过 72 字符、使用祈使语气、结尾无句号。

---

## 4. GitLab Pipeline 任务定义

以 `.gitlab-ci.yml` 为核心，定义 `stages` 与 `rules`。

### 4.1 推荐 Stages

```yaml
stages:
  - build
  - test
  - lint
  - security
  - deploy
```

### 4.2 Merge Request 触发

使用 `rules` 仅在 MR 事件时运行：

```yaml
build:
  stage: build
  script:
    - make build
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

也可组合：MR 时跑，主干推送时也跑。

```yaml
test:
  stage: test
  script:
    - make test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### 4.3 Nightly 触发（定时）

使用 GitLab **Schedules**（Web UI：CI/CD → Schedules），配置 cron：

```
0 2 * * *      # 每天 02:00
```

在 `.gitlab-ci.yml` 中识别：

```yaml
nightly-full-test:
  stage: test
  script:
    - make test-all
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
```

并通过 `CI_PIPELINE_SCHEDULE` 变量区分不同定时任务。

### 4.4 完整示例（节选）

```yaml
stages: [build, test, lint, security, deploy]

variables:
  IMAGE_TAG: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

build:
  stage: build
  script: [make build]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

lint:
  stage: lint
  script: [make lint]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

security-scan:
  stage: security
  script: [make scan]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'   # 仅 nightly
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-staging:
  stage: deploy
  script: [make deploy-staging]
  environment: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 5. Mail 提醒规则

通过 **GitLab → Settings → Integrations / Notifications** 与项目级通知配置实现。

### 5.1 触发场景与接收人

| 场景 | 接收人 | 渠道 |
| --- | --- | --- |
| MR 被创建 / 需要评审 | 指定 Reviewer、Maintainer | 邮件 + GitLab 通知 |
| MR 被批准 / 被合并 | MR 作者 | 邮件 |
| Pipeline 失败（主干） | 提交人 + 团队 Lead | 邮件 |
| Nightly Pipeline 失败 | 值班人（On-call）/ 平台组 | 邮件 + 群机器人 |
| 安全扫描高危告警 | 安全负责人 + Maintainer | 邮件 |
| Tag / Release 发布 | 全员订阅者 | 邮件 |

### 5.2 配置要点

- 项目开启 **Email on push / Pipelines**、**Merge Request** 通知。
- 设置 **Group-level** 邮件分发组（如 `team-payments@corp.com`），避免逐人配置。
- 失败通知使用 `CI_PIPELINE_SOURCE == "schedule"` 或 `main` 分支才发送，避免 MR 噪声。
- 推荐结合 Webhook 转发至企业 IM（如 Slack/钉钉/飞书）做实时提醒，邮件作为留痕。

### 5.3 默认通知级别建议

- 个人 GitLab 通知：参与项目 → **Participate**（仅与我相关）。
- 主干失败、release、安全事件 → **Watch**（强制邮件）。

---

## 6. VS Code 调试环境

### 6.1 推荐插件（按语言/用途）

| 类别 | 插件 |
| --- | --- |
| 通用 | GitLens、Git Graph、EditorConfig、Prettier、Error Lens |
| C/C++ | C/C++、C/C++ Test Explorer、CMake Tools |
| Python | Python、Pylance、Python Test Explorer |
| Go | Go |
| Rust | rust-analyzer、CodeLLDB |
| JS/TS | ESLint、vscode-typescript |
| 容器 | Docker、Kubernetes、Remote - Containers / WSL |
| 数据库 | SQLTools |
| 其他 | REST Client、Thunder Client |

### 6.2 调试配置（`.vscode/launch.json`）

通用 C++ / Python 示例：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C++: Launch",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/app",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb"
    },
    {
      "name": "Python: Current File",
      "type": "python",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal",
      "justMyCode": true
    }
  ]
}
```

### 6.3 断点（Breakpoint）

- **行断点**：行号左侧点击。
- **条件断点**：右键断点 → 编辑条件（如 `x > 100`）。
- **命中次数断点**：设置 `hitCount > 50` 才中断。
- **日志断点**（Logpoint）：不中断，仅打印表达式，适合生产模拟。
- **函数断点**：按函数名下断点，避免找不到源码位置。

### 6.4 调试控制

- 启动：`F5`；继续：`F5`；单步跳过：`F10`；单步进入：`F11`；单步跳出：`Shift+F11`；重启：`Ctrl+Shift+F5`；停止：`Shift+F5`。
- **Watch**：添加变量表达式实时观察。
- **Call Stack**：查看函数调用链。
- **Variables / Locals**：查看当前作用域变量。

### 6.5 内存与性能查看

- **内存视图（C/C++）**：运行后切换到 **Memory** 面板，添加内存地址范围（如 `&buffer` 起 256 字节）直接查看十六进制/ASCII。
- **反汇编**：在断点处右键 → **Open Disassembly View**。
- **性能 / CPU Profiling**：
  - Python：使用 `python -m cProfile` 或 Debugpy 的 **Sampling** 模式。
  - Node：Launch 配置加 `"profile": true`。
  - C/C++：配合 `perf` / Valgrind（Massif 查内存占用，Memcheck 查泄漏）。
- **内存泄漏**：C/C++ 用 Valgrind；Python 用 `tracemalloc`；可在 `launch.json` 启动前 `preLaunchTask` 注入检测脚本。
- **附加到进程**：配置 `"request": "attach"`，通过 `processId` 附加到已运行服务（容器内需开启 `ptrace`）。

### 6.6 远程 / 容器调试

- 使用 **Remote - SSH / WSL / Containers** 扩展，确保本地调试环境与 CI 一致。
- 容器内调试需安装 `gdb`/`delve` 并暴露调试端口。

---

## 7. 基于 Makefile 的构建脚本

统一用 `make` 作为本地与 CI 的构建入口，降低环境差异。

### 7.1 推荐 Target 结构

```makefile
# Makefile
SHELL := /bin/bash
.DEFAULT_GOAL := help

.PHONY: help build test test-all lint scan clean deploy-staging format

help:  ## 显示可用目标
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2}'

build:  ## 编译构建
	@echo "==> building"
	$(MAKE) -C src build

test:  ## 单元测试
	@echo "==> unit tests"
	./scripts/run-tests.sh unit

test-all:  ## 全量测试（含集成）
	@echo "==> full test suite"
	./scripts/run-tests.sh all

lint:  ## 静态检查
	@echo "==> lint"
	./scripts/lint.sh

scan:  ## 安全扫描
	@echo "==> security scan"
	./scripts/security-scan.sh

format:  ## 代码格式化
	@echo "==> format"
	./scripts/format.sh

deploy-staging:  ## 部署到预发
	@echo "==> deploy staging"
	./scripts/deploy.sh staging

clean:  ## 清理产物
	rm -rf build dist .cache
```

### 7.2 使用约定

- 本地与 CI 使用相同 `make` 目标，保证一致。
- CI 中调用 `make build / test / lint / scan`，nightly 调用 `make test-all`。
- 任务复杂逻辑下沉到 `scripts/` 目录下的脚本，Makefile 仅做编排。
- 变量通过 `make VAR=value` 或 `.env` 注入；CI 通过 GitLab CI 变量覆盖。

### 7.3 跨平台注意

- Windows 开发者建议用 **WSL / Git Bash** 执行 `make`。
- 避免 Makefile 中写死绝对路径，使用 `$(CURDIR)` 与相对路径。

---

## 附录：落地检查清单

- [ ] 所有仓库归属到正确的 Group/Subgroup
- [ ] `main`/`develop` 已设为 Protected Branch
- [ ] MR 模板、Commit 规范已写入项目 README / CONTRIBUTING
- [ ] `.gitlab-ci.yml` 含 MR 触发与 nightly 触发
- [ ] 邮件分发组与通知级别已配置
- [ ] 仓库内置 `.vscode/launch.json` 与推荐插件（`.vscode/extensions.json`）
- [ ] 仓库根目录含 `Makefile` 与 `scripts/`

---

*本规范为初版，后续根据实际使用反馈迭代修订。*
