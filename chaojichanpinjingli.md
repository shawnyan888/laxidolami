# 软件开发环境规范（Specification）

| 项目 | 内容 |
|------|------|
| 文档名称 | 软件开发环境规范 |
| 版本 | v0.1（初版） |
| 日期 | 2026-10-29 |
| 作者 | 待填 |
| 状态 | 初版，待评审 |
| 适用范围 | 全体开发/验证成员（15 人以上团队） |

## 0. 目标与成功指标

本规范统一代码托管平台、命名、分支与提交、CI 流水线、邮件通知、调试环境与构建脚本的约定。目标是：新成员快速上手，协作流程可自动化，任何变更可追溯。

成功指标（量化验收）：

| 指标 | 目标值 | 验收方式 |
|------|--------|---------|
| 新成员环境搭建耗时 | ≤ 1 个工作日 | 新人按本文档实操计时 |
| MR 提交到 CI 反馈时长 | ≤ 10 分钟 | CI 运行统计 |
| nightly 失败通知送达 | ≤ 5 分钟 | 值班记录 |
| 分支/提交格式合规率 | ≥ 95% | 流水线自动检查统计 |
| 增量构建耗时 | ≤ 全量构建的 20% | make 计时对比 |

落地阶段：

| 阶段 | 优先级 | 内容 | 验收标准 |
|------|--------|------|---------|
| 阶段 1 | P0 | GitLab 部署、group/project 创建、保护分支与命名规则配置 | 全员可登录、规则实际生效 |
| 阶段 2 | P0 | CI 流水线（MR 触发 + nightly）+ 邮件通知 | 一条 MR 全流程跑通，失败通知送达 |
| 阶段 3 | P1 | VSCode 调试环境与 Makefile 模板推广 | 试点仓库可断点调试、可增量构建 |
| 阶段 4 | P1 | 规范培训与月度合规统计 | 合规率 ≥ 95% |

---

## 1. 版本管理平台选型：为什么选择 GitLab

**结论：选 GitLab（自托管）。** 对 15 人以上的内部研发团队，GitLab 在代码主权、权限分层、CI 一体化上综合最优。

选择理由：

1. **私有部署、代码主权**：可部署在内网，代码不离开公司网络，满足保密与合规要求。
2. **内置 CI/CD**：GitLab CI 与仓库深度集成，MR 流水线开箱即用，无需额外拼接 CI 系统。
3. **权限模型适合分层管理**：Group / Subgroup / Project 三级结构 + 角色（Guest / Reporter / Developer / Maintainer / Owner），适合 15 人以上团队按部门、模块划分职责。
4. **MR 评审流程完善**：审批规则（approval rules）、受保护分支、squash 合并、diff 内逐行讨论，代码评审闭环完整。
5. **Runner 可注册到任意服务器**：CI 任务可以指定跑到专用服务器（如仿真服务器、EDA 主机）上，用 tags 区分资源池。
6. **一站式功能**：issue、wiki、container registry、pages、snippets，减少工具碎片化。
7. **成本低**：社区版（CE/免费层）已覆盖上述需求；如需更细的审批流与安全审计可再评估企业版。

备选方案对比（简述）：

| 平台 | 优势 | 不选原因 |
|------|------|---------|
| GitHub | 生态最大、上手快 | 云托管为主，内网部署依赖企业版且成本高；CI 与权限深度绑定平台 |
| Gitea | 极轻量、部署简单 | CI（Actions 兼容层）与权限模型偏弱，15 人以上团队协作能力不足 |
| Gerrit | 评审模型严谨（Change 机制） | 学习曲线陡、需要外接 Jenkins 做 CI，集成成本高 |

**待确认项**：部署版本（CE/EE）、部署主机与备份策略（见第 9、10 节）。

---

## 2. GitLab 命名规范（Group / Subgroup / Project）

### 2.1 总体原则

- 全小写字母、数字、短横线 `-`；禁止大写、下划线、空格、点号。
- 见名知义，禁止无意义缩写（缩写必须在术语表登记）。
- 层级不超过 3 层：`group / subgroup / project`，超过 3 层说明需要重新划分。
- **禁止与业界协议、标准、工具名撞名**（如 `amba`、`axi`、`uvm`、`git`），避免检索歧义与文档混乱。
- 禁止用个人姓名命名 project（个人仓库放个人 subgroup 或 fork）。

### 2.2 层级规则

| 层级 | 规则 | 示例 |
|------|------|------|
| Group（顶层） | 公司或部门名，全小写 | `<company>`、`dv` |
| Subgroup | 团队 / 产品线 / 模块域 | `common`、`module-a` |
| Project | 模块名，与顶层模块名一致；类型用后缀区分 | `module-a-rtl`、`module-a-tb`、`module-a-env` |

Project 后缀约定：`-rtl`（设计）、`-tb`（测试平台）、`-env`（环境/验证）、`-lib`（公共库）、`-tools`（工具脚本）、`-doc`（文档）。

### 2.3 示例目录树

```
<company>/                     # 顶层 group
├── dv/                        # subgroup：验证团队
│   ├── common/
│   │   ├── uvm-lib/           # 公共验证库
│   │   └── scripts/
│   └── module-a/
│       ├── module-a-rtl/
│       ├── module-a-tb/
│       └── module-a-env/
└── infra/
    └── spec/                  # 本规范所在仓库（建议）
```

### 2.4 创建流程（必须）

1. 建仓申请：提 issue / 工单，写明 group、project 名与用途。
2. 管理员按本规范创建，并配置保护分支与成员角色（见第 3、8 节）。
3. 创建后在团队群通知 + 邮件告知仓库地址与成员权限。
4. 禁止个人自行创建不规范命名的仓库；发现后由管理员归档或重命名。

---

## 3. 分支与提交规范（Branch / Commit Message）

### 3.1 分支模型

主干开发（trunk-based）为主：`main` 长期稳定可发布；短期 feature/bugfix 分支合并后即删除；`release/*` 按需创建。**待确认**：是否改用 Git Flow（见第 10 节）。

### 3.2 分支命名规则（必须）

格式：`<type>/<context>-<short-desc>`

| 类型 | 格式 | 说明 |
|------|------|------|
| feature | `feature/<issue-id>-<desc>` | 新功能，必须关联 issue |
| bugfix | `bugfix/<issue-id>-<desc>` | 缺陷修复，必须关联 issue |
| hotfix | `hotfix/<desc>` | 紧急修复，从 `main` 切出，修复后立即合并并打 tag |
| release | `release/<x.y>` | 版本发布分支 |
| personal | `personal/<name>/<desc>` | 个人实验分支，**禁止合并回 main** |

规则：全小写、`-` 分隔；`<desc>` 不超过 5 个单词；`<issue-id>` 为 issue 编号（如 `feature/1234-add-fifo`）。

### 3.3 保护分支（必须）

- `main`、`release/*` 为受保护分支：禁止直接 push，必须走 MR；禁止 force push。
- 合并条件：CI 流水线全部通过；核心仓库至少 **2 名** Maintainer 批准，一般仓库至少 1 名。
- 建议开启 squash 合并，保持 `main` 历史干净。

### 3.4 Commit Message 规则（必须）

采用 Conventional Commits 格式：

```
<type>(<scope>): <subject>

<body>          # 为什么改、怎么改；feat/fix 必须写

<footer>        # 引用 issue；破坏性变更加 BREAKING CHANGE
```

| 项 | 规则 |
|----|------|
| type | `feat` `fix` `docs` `style` `refactor` `perf` `test` `build` `ci` `chore` `revert` 之一 |
| scope | 可选，模块名，小写 |
| subject | ≤ 50 字符；动词开头；结尾不加句号 |
| body | 每行 ≤ 72 字符；解释「为什么」和「怎么做」 |
| footer | `Refs #<issue-id>` 或 `Closes #<issue-id>`；破坏性变更：`BREAKING CHANGE: <描述>` |

正例：

```
feat(fifo): 增加可配置深度的同步 FIFO

深度通过参数 FIFO_DEPTH 配置，默认 16。
新增 case：深度为 1 和 2 的边界场景。

Closes #1234
```

反例：

```
update code
```

```
FEAT(fifo): add fifo  # type 大写、无正文、无 issue 引用
```

### 3.5 自动检查（必须）

分支名与 commit message 在 CI `lint` 阶段自动检查（见第 4 节），不合规直接失败，不进入人工评审。合规率作为月度统计指标（见第 0 节）。

---

## 4. GitLab CI Pipeline 任务定义

### 4.1 总览

每个仓库根目录放置 `.gitlab-ci.yml`。统一 stage 划分：

```
lint → build → test/sim → report → (.post 通知)
```

| stage | 内容 | 触发场景 |
|-------|------|---------|
| lint | 分支名、commit message、代码静态检查 | 所有流水线 |
| build | 编译（make） | MR / push / nightly |
| test/sim | 单元测试或仿真回归 | MR（冒烟）、nightly（全量） |
| report | 覆盖率、结果归档 | nightly |
| .post | 失败通知（邮件/群） | 按规则 |

### 4.2 触发矩阵

| 事件 | 触发内容 | 用途 |
|------|---------|------|
| push（非 main 分支） | lint + build | 快速反馈 |
| MR（`merge_request_event`） | 完整流水线（lint + build + 冒烟测试） | 合并门禁，全部通过才可合并 |
| push 到 `main` | 完整流水线 + 打 tag | 主线保护 |
| nightly（`schedule`） | 全量回归 + 覆盖率 + 归档 | 夜间质量守护 |

nightly 创建步骤：GitLab 仓库 → Settings → CI/CD → Schedules → New schedule，cron 示例 `0 2 * * *`（每天 02:00，注意时区），变量设 `NIGHTLY=true`。**具体时间与回归范围待确认**（见第 10 节）。

### 4.3 Runner 与资源

- 通用编译 job 用默认 runner；仿真/EDA 类 job 用 `tags: [linux, sim]` 指定专用 runner。
- nightly 长任务设 `timeout` 与 `retry`，避免无限挂起。

### 4.4 示例 `.gitlab-ci.yml`

```yaml
stages:
  - lint
  - build
  - sim
  - report

default:
  tags: [linux]

# 命名与提交信息检查：所有流水线必跑
lint:check:
  stage: lint
  script:
    - ./ci/check-commit.sh "$CI_COMMIT_MESSAGE"
    - ./ci/check-branch.sh "$CI_COMMIT_REF_NAME"

build:
  stage: build
  script:
    - make -j$(nproc) DEBUG=0 all
  artifacts:
    paths: [build/]
    expire_in: 1 day

# MR 冒烟测试：合并门禁
sim:smoke:
  stage: sim
  script:
    - make sim
  tags: [linux, sim]
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# 主线保护
sim:main:
  stage: sim
  script:
    - make sim
  tags: [linux, sim]
  rules:
    - if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH == "main"

# nightly 全量回归
sim:nightly:
  stage: sim
  script:
    - make sim-all
  tags: [linux, sim]
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $NIGHTLY == "true"
  timeout: 4h
  retry: 1

# nightly 覆盖率报告，保留 14 天
report:
  stage: report
  script:
    - make cov
  artifacts:
    paths: [coverage/]
    expire_in: 14 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $NIGHTLY == "true"

# 失败通知（详见第 5 节）
notify:
  stage: .post
  script:
    - ./ci/notify.sh "$CI_PIPELINE_NAME" "$CI_PIPELINE_STATUS"
  when: on_failure
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $NIGHTLY == "true"
```

---

## 5. 邮件提醒规则

### 5.1 总原则（结论先行）

**失败必达、成功静默、周期汇总、重要事件即时。** 邮件只用于必须留档的正式通知，日常讨论走群消息，避免邮件疲劳。

### 5.2 通知矩阵

| 事件 | 触发条件 | 收件人 | 时机 |
|------|---------|--------|------|
| 我的 MR 被指派/被 @/有新评论 | MR 活动 | MR 作者/被 @ 人 | 即时（平台可 10 分钟聚合） |
| 我的 MR 的 CI 失败 | pipeline 失败 | MR 作者 | 即时 |
| `main` 分支 CI 失败 | pipeline 失败 | 提交者 + 仓库 Maintainer | 即时 |
| nightly 失败 | pipeline 失败 | 项目负责人 + 当日值班 | ≤ 5 分钟 |
| nightly 成功 | pipeline 成功 | 不发邮件，进周报 | 每周汇总 |
| MR 合并/关闭 | MR 状态变更 | MR 作者 | 即时 |
| 本规范变更、权限变更 | 规范仓库 MR 合并 | 全员 | 即时（同步群通知，见第 8 节） |

### 5.3 频率与聚合规则

- 同一 pipeline 30 分钟内重复失败只发一封，避免轰炸。
- 个人事件即时发送；团队级汇总每周一 09:00 由管理员导出周报。
- 禁止把 CI 日志全文塞进邮件：只放摘要（状态、失败 stage、首个报错）+ 流水线链接。
- 邮件标题统一格式：

```
[<group>/<project>] <状态> <分支> <commit subject>
例：[dv/module-a] FAILED main fix(tb): 修正时钟激励
```

### 5.4 实现方式

1. 个人通知：GitLab 用户级 Notification settings（参与的项目启用即时通知）。
2. 项目级通知：Integrations → Emails on push（仅 `main`/`release` 分支）。
3. 自定义通知：CI `.post` 阶段 `notify` job 通过 SMTP 发送（见 4.4 示例）。
4. 大型团队配套：值班表（on-call）+ 邮箱组（mail group），值班人轮换维护在 `infra/spec` 仓库。

**待确认项**：SMTP 服务器地址、邮箱组名单（见第 10 节）。

---

## 6. VSCode 调试环境

### 6.1 插件清单

| 插件 | 用途 | 必要性 |
|------|------|--------|
| C/C++（ms-vscode.cpptools） | IntelliSense + gdb 调试核心 | 必须 |
| Makefile Tools（ms-vscode.makefile-tools） | make 目标集成与运行 | 必须 |
| GitLab Workflow（gitlab.gitlab-workflow） | 仓库内创建/评审 MR | 必须 |
| GitLens（eamodio.gitlens） | blame、历史追溯 | 推荐 |
| Remote-SSH（ms-vscode-remote.remote-ssh） | 连接服务器远程开发调试 | 推荐 |
| MemoryView（mcu-debug.memory-view） | 内存区间可视化 | 推荐 |
| Hex Editor（ms-vscode.hexeditor） | 二进制文件查看 | 可选 |
| clangd（llvm-vs-code-extensions.vscode-clangd） | 备选 IntelliSense（与 C/C++ 二选一） | 可选 |

### 6.2 工程配置

`.vscode/tasks.json`（构建任务）：

```jsonc
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "make build (debug)",
      "type": "shell",
      "command": "make",
      "args": ["DEBUG=1", "build", "-j4"],
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": ["$gcc"]
    }
  ]
}
```

`.vscode/launch.json`（调试配置）：

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "debug app (gdb)",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/app",
      "args": [],
      "cwd": "${workspaceFolder}",
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [{ "text": "-enable-pretty-printing" }],
      "preLaunchTask": "make build (debug)"
    }
  ]
}
```

调试前提（必须）：编译加 `-g -O0`（或 `-Og`），即 `make DEBUG=1 build`；release 产物不用于调试。

### 6.3 调试方法

| 能力 | 操作 |
|------|------|
| 行断点 | 行号左侧点击（F9） |
| 条件断点 | 右键断点 → Edit Breakpoint → 写条件（如 `i == 100`） |
| 函数断点 | 调试视图断点区 + 号 → Function Breakpoint |
| 日志断点（Logpoint） | Edit Breakpoint 选 Log Message，不中断程序 |
| 数据断点（watchpoint） | 调试中右键变量 → Add Data Breakpoint，变量被改写时中断 |
| 单步 | F10 跳过 / F11 进入 / Shift+F11 跳出 |
| 变量与表达式 | Variables 面板；Watch 面板加表达式（如 `ptr[3]`）；悬停查看 |
| 调用栈 | Call Stack 面板切换栈帧，查看各帧局部变量 |
| 内存查看 | 调试控制台 `-exec x/16bx &var`（打印 16 字节）；或 MemoryView 插件可视化地址区间 |
| 反汇编 | Disassembly 视图（对应栈帧切换） |
| core dump 分析 | 终端 `gdb build/app core.<pid>`，或配置 launch 类型 attach 到核心文件 |
| 远程调试 | Remote-SSH 连服务器直接用 gdb；无 GUI 环境用 gdbserver：目标机 `gdbserver :2345 ./build/app`，本机配置 `"miDebuggerServerAddress": "host:2345"` + attach |

常用快捷键：F5 启动/继续、F9 断点、F10/F11 单步、Shift+F5 停止、Ctrl+Shift+D 调试视图。

---

## 7. 基于 Makefile 的构建脚本

### 7.1 约定（必须）

- 每个 project 根目录一个 Makefile；目标名全仓库统一。
- 统一目标：`all`、`build`、`clean`、`test`（或 `sim`）、`cov`、`help`。
- `make help` 必须列出全部目标及说明。
- `clean` 只删除生成物目录（`$(BUILD_DIR)`），禁止未限定路径的 `rm -rf`。
- 支持 `DEBUG=1` 切换调试构建；路径用变量，不写死绝对路径。
- 用 `-MMD -MP` 生成依赖实现增量编译，配合 `-j` 并行。

### 7.2 变量约定

| 变量 | 含义 | 默认 |
|------|------|------|
| `CC` / `CXX` | 编译器 | `gcc` / `g++` |
| `CFLAGS` / `CXXFLAGS` | 编译选项 | 按仓库设定，debug 加 `-g -O0` |
| `CPPFLAGS` | 预处理选项（include 路径） | `-Isrc -Iinclude` |
| `LDFLAGS` | 链接选项 | 按仓库设定 |
| `BUILD_DIR` | 生成物目录 | `build` |

### 7.3 骨架示例

```makefile
# 通用构建骨架：各仓库保持目标名一致
CXX       ?= g++
CXXFLAGS  ?= -std=c++17 -Wall -Wextra
CPPFLAGS  += -Isrc -Iinclude
LDFLAGS   ?=
BUILD_DIR ?= build
DEBUG     ?= 0

ifeq ($(DEBUG),1)
CXXFLAGS += -g -O0
else
CXXFLAGS += -O2
endif

SRCS := $(shell find src -name '*.cpp')
OBJS := $(SRCS:src/%.cpp=$(BUILD_DIR)/%.o)
DEPS := $(OBJS:.o=.d)

all: build

build: $(BUILD_DIR)/app

$(BUILD_DIR)/app: $(OBJS)
	$(CXX) $(OBJS) -o $@ $(LDFLAGS)

$(BUILD_DIR)/%.o: src/%.cpp
	@mkdir -p $(dir $@)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -MP -c $< -o $@

test: build
	./$(BUILD_DIR)/app --self-test

cov: build
	./$(BUILD_DIR)/app --self-test --cov

clean:
	rm -rf $(BUILD_DIR)

help:
	@echo "make [all|build|test|cov|clean|help]"
	@echo "  DEBUG=1 make build   # 调试构建（-g -O0）"

-include $(DEPS)

.PHONY: all build test cov clean help
```

验收标准：任一仓库克隆后执行 `make DEBUG=1 build` 可成功；`make clean && make build` 全量正确；连续两次 `make build` 第二次为空构建（增量生效）。

---

## 8. 协作与通知机制（规范落地与变更）

本规范适用于 15 人以上团队，变更必须走正式流程（群通知 + 邮件留档，+1 升级）：

| 场景 | 通知对象 | 方式 | 时限 |
|------|---------|------|------|
| 规范细节调整（不影响排期） | 相关模块负责人 | 群 @ 相关人 | 4 小时内 |
| 规范变更影响排期/流程 | 全体 + 直属上级（+1） | 群 @all + 邮件 + 更新本文档 | 2 小时内 |
| 平台/流水线紧急故障 | 全体 + 直属上级 + 总监 | 电话 + 群 @all | 15 分钟内 |

规范变更流程：在规范仓库提 MR → 评审（至少 1 名 Maintainer + 各模块负责人代表）→ 合并 → 通知全员 → 更新修订记录。

违规处理：分支名/commit 格式由流水线自动拦截（第 3.5 节）；人工发现的违规在月度周报统计并通报。

---

## 9. 风险与应对

| 风险 | 等级 | 应对策略 |
|------|------|---------|
| 命名规范与团队历史习惯冲突，落地阻力大 | 高 | 流水线自动检查替代人工约束；试点仓库先行；给 2 周迁移窗口 |
| CI runner 资源不足（15 人以上并发） | 高 | tags 分流专用 runner；nightly 与人工任务错峰；job 设 timeout/retry |
| 邮件疲劳导致重要通知被忽略 | 中 | 失败必达、成功静默策略（第 5.1 节）；值班表兜底 |
| 规范文档与实际操作漂移 | 中 | 规范仓库版本管理；季度评审例会修订 |
| GitLab 平台不可用 | 中 | 每日备份 + 异地备份；应急流程另行制定（**待补充**） |
| 远程调试链路复杂（服务器/VNC 环境） | 中 | Remote-SSH + gdbserver 方案（第 6.3 节）；输出调试操作手册 |

---

## 10. 待定问题（Open Questions）

以下事项需在评审会上确认后更新进正文：

1. GitLab 部署版本（CE/EE）、主机与备份方案。
2. 分支模型最终确认：trunk-based 还是 Git Flow。
3. nightly 的 cron 时间、时区与全量回归范围。
4. SMTP 服务器地址、邮箱组名单与值班表制度。
5. MR 模板与 squash 合并是否强制。
6. 本规范仓库的存放位置（建议 `infra/spec`，见 2.3 节）。
7. 是否引入 commitlint 等本地 pre-commit 钩子（与 CI 检查互补）。

---

## 11. 修订记录

| 版本 | 日期 | 修订内容 | 作者 |
|------|------|---------|------|
| v0.1 | 2026-10-29 | 初版：按 7 项要求起草（平台选型/命名/分支提交/CI/邮件/VSCode/Makefile），含目标指标、协作机制、风险与待定项 | 待填 |
