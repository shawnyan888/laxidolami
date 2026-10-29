# 软件开发环境 Specification

| 项目 | 内容 |
|---|---|
| 文档版本 | v0.1（初版草案） |
| 适用范围 | ace 项目组全部软件仓库（IC 验证环境 / 上位机 / 公共库） |
| 生效日期 | 【待确认】 |
| 维护人 | 【待确认】 |

> 标注【待确认】的条目为初版假设，需评审后定稿；path 类命名一经启用不可再改（clone URL、CI、submodule 引用全部失效），必须在建库前定死。

---

## 1. 为什么选择 GitLab

选择**自建 GitLab**（Self-hosted / Self-managed）作为唯一的代码托管与 DevOps 平台，理由如下：

1. **数据自主可控**：验证环境、协议 agent、上位机代码均属公司资产，自建实例不出内网，满足保密要求；EDA 主机通常无外网，自建服务器可直接内网访问。
2. **一体化平台**：Git 托管 + Issue/Milestone + Merge Request 评审 + CI/CD + Wiki + 制品库（Package Registry）在一个系统内闭环，不需要额外拼装 Jenkins/Gerrit。
3. **CI/CD 内置且声明式**：`.gitlab-ci.yml` 随仓库版本化，pipeline 支持 MR 触发、定时（nightly）触发、手动触发，满足验证回归的需求。
4. **Merge Request 工作流成熟**：强制评审、流水线通过后才可合并、讨论串与代码行绑定，适合验证团队代码质量管理。
5. **权限模型清晰**：Group / Subgroup / Project 三级继承，配合 Protected Branch 与 CODEOWNERS，可以按角色（开发者/评审员/维护者）精细控权。
6. **成本**：Community Edition（免费版）已覆盖上述全部需求；如后续需要审批规则、合规框架再评估升级 Premium。

对比备选方案的结论：
- GitHub/Gitee（公有云）：代码出内网，保密不可接受。
- 裸 Git + Jenkins：需要维护两套系统，MR 评审体验差，维护成本高。
- Gerrit：评审模型强但学习曲线陡，CI 需外挂，团队规模不划算。

---

## 2. GitLab Group / Subgroup / Project 命名规范

### 2.1 总体约定

- **一律全小写 + 连字符（kebab-case）**：GitLab 的 project path（即 clone URL 中的 slug）强制小写，Name 大小写混用会造成显示名与实际 URL 不一致，故 Name 与 Path 统一小写。
- 名字用**名词**，不用动词；缩写用行业通用形式（如 `verif`）。
- 层级承载上下文：group/subgroup 已表达项目归属后，project 名保持简短，不重复前缀以外的冗余词。

### 2.2 Group / Subgroup 结构

```
ace/                        ← 顶层 Group：项目代号
├── core/                   ← Subgroup（可选层级，仓库多时再启用）
└── (projects)
```

| 层级 | 名称 | Path | 说明 |
|---|---|---|---|
| Group | ACE Project | `ace` | 项目代号。注意：与 AMBA ACE（AXI Coherency Extensions）协议撞名，所有文档中首次出现必须写 "ace 项目"，协议一律写全称 "AMBA ACE" |
| Subgroup | 【待确认】 | 【待确认】 | 初版建议先不建 subgroup，仓库数 > 6 时再按域拆分（如 `ace/ip`、`ace/tools`） |

### 2.3 Project 命名

| 仓库 | Path（最终 clone URL 用名） | 职责 |
|---|---|---|
| 公共库 / UVM 基类 | `ace/ace-core` | 公共 class、seq 库、寄存器模型基类 |
| 协议 Agent | `ace/ace-agents` | 各协议 agent（复数，预留多协议；单协议子目录如 `axi/`、`apb/`） |
| 上位机 GUI | `ace/ace-ui` | 上位机软件 |
| 验证环境 / 回归 | `ace/ace-verif` | testbench 顶层、测试用例、回归列表 |

规则：
- project 名 = `<代号>-<域>`，全小写；域名单词表：`core` / `agents` / `ui` / `verif`，新增域需评审。
- 协议相关文件/目录必须带协议全称小写（`axi`、`apb`、`amba-ace`），禁止裸写 `ace` 指代协议。
- CI 日志、脚本中引用项目一律用 `$CI_PROJECT_PATH`（含 group，可辨归属），不用 `$CI_PROJECT_NAME`。

### 2.4 分支（Branch）命名规范

| 类型 | 格式 | 示例 | 说明 |
|---|---|---|---|
| 主干 | `main` | — | 唯一长期分支，始终可发布；受保护，仅允许 MR 合入 |
| 特性 | `feat/<issue号>-<短描述>` | `feat/12-axi-burst-seq` | 从 `main` 拉出，合回 `main` |
| 修复 | `fix/<issue号>-<短描述>` | `fix/34-uvm-phase-leak` | 同上 |
| 发布 | `release/<版本>` | `release/1.2` | 需要维护多版本时启用【待确认】 |
| 个人试验 | `wip/<用户名>-<描述>` | `wip/syan-debug-env` | 不允许发起 MR 到 main，定期清理 |

规则：
- 类型前缀限定 `feat/ fix/ release/ wip/` 四种，短描述用 kebab-case，不超过 5 个词。
- 分支必须关联 Issue 号，便于 CI 与 MR 描述自动引用（`Closes #12`）。
- Protected Branches 设置：`main`（Maintainer 可合并，Developer 不可 push）、`release/*`（同 main）。
- 合并方式统一 **Squash commits + 合并后删除源分支**【待确认：若需保留完整提交史则改为 merge commit】。

### 2.5 Commit Message 规则

采用 Conventional Commits：

```
<type>(<scope>): <subject>

<body，可选，说明 why 而非 what>

<footer：Closes #12 / BREAKING CHANGE: ...>
```

- **type** 限定：`feat` `fix` `refactor` `test` `docs` `build` `ci` `chore` `revert`
- **scope**：仓库内模块名，如 `axi`、`apb`、`regmodel`、`gui`、`makefile`
- **subject**：祈使句、≤ 72 字符、结尾不加句号，中英文混排时中文不空格分隔【待确认：是否强制英文】
- **body**：每行 ≤ 100 字符；涉及验证行为变化必须写明影响的 test/回归项
- 一次 commit 只做一件事；禁止 `update`、`fix bug`、`修改` 这类无信息量 message
- MR 标题 = squash 后的 commit subject，同样遵循上述格式
- CI 中用 commitlint（或等价脚本）在 MR pipeline 校验格式，不合规直接 fail

示例：

```
feat(axi): add outstanding burst sequence for ACE-Lite master

Support up to 16 outstanding transactions; regression item
axi_burst_stress added to nightly list.

Closes #12
```

---

## 3. GitLab CI Pipeline 任务定义

### 3.1 Stage 划分

```yaml
stages: [lint, build, test, regression, report]
```

| Stage | 内容 | 失败后果 |
|---|---|---|
| lint | commit message 校验、代码风格（verible/clang-format --dry-run）、lint 工具（如 verilator --lint-only） | MR 阻塞 |
| build | 编译（仿真模型编译 / 上位机构建），产出制品 | MR 阻塞 |
| test | 冒烟测试（smoke test 列表，≤ 15 分钟） | MR 阻塞 |
| regression | 完整回归（仅 nightly / 手动触发） | 邮件报告 |
| report | 汇总覆盖率、回归结果，生成报告并归档 | 邮件报告 |

### 3.2 触发规则

| 触发 | 时机 | 执行范围 |
|---|---|---|
| **Merge Request pipeline** | MR 创建 / 有新 commit push | lint + build + test（smoke） |
| **Nightly pipeline** | Schedule 每日 02:00（服务器时区【待确认】），目标分支 `main` | lint + build + test + regression + report |
| **Main branch pipeline** | MR 合入 main 后 | build + test（smoke），确认合并未破坏主干 |
| **手动（web）** | 页面点 Run pipeline，可选 `FULL_REGRESSION=true` | 完整回归，用于发版前 |

### 3.3 `.gitlab-ci.yml` 骨架

```yaml
stages: [lint, build, test, regression, report]

variables:
  SMOKE_LIST: "tests/smoke.list"
  REGRESSION_LIST: "tests/regression.list"

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "web"

# --- MR / main：冒烟 ---
smoke-test:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - make regress LIST=$SMOKE_LIST

# --- Nightly：完整回归 ---
nightly-regression:
  stage: regression
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - make regress LIST=$REGRESSION_LIST
  artifacts:
    when: always
    paths: [logs/, coverage/]
    expire_in: 30 days

coverage-report:
  stage: report
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - make coverage-report
  artifacts:
    when: always
    paths: [reports/]
```

### 3.4 Runner 约定

- Runner 部署在 EDA 主机 / 编译服务器上，注册为 group-level shared runner（`ace` group 共享）。
- 使用 shell executor 或 docker executor【待确认：取决于 EDA 工具 license 是否允许容器内运行】。
- 每个 job 打 tag：`[eda-sim]`（需要仿真器 license）、`[build]`（纯编译）、`[light]`（lint/脚本类），按资源需求调度。
- 并发上限与 license 数量挂钩【待确认 license 池大小】。

---

## 4. 邮件提醒规则

原则：**失败必达责任人，成功不打扰**。

| 事件 | 收件人 | 机制 |
|---|---|---|
| MR pipeline 失败 | 提交者 | GitLab 用户设置：`Settings → Notifications → Failed pipeline`；无需服务端配置 |
| Nightly pipeline 失败 | 提交者 + 验证组邮件列表 `ace-verif@<内网域名>`【待确认】 | ① 用户级 Failed pipeline 邮件；② report stage 末尾 `mail`/SMTP 脚本向组列表发送失败摘要 |
| Nightly 回归结果报告（成败都发） | 验证组邮件列表 | report stage 生成 HTML/文本摘要（通过率、失败用例列表、覆盖率增量），SMTP 发送；主题格式 `[ace][nightly][PASS/FAIL] <date> <main-shorthash>` |
| MR 请求评审 | 被 assign/review 的人 | GitLab 站内通知 + 邮件（用户设置 On mention / On participation） |
| Issue 指派 | 被指派人 | 同上 |
| main 分支被直接 push（异常） | Maintainer 列表 | Protected branch 审计 + push event webhook【待确认是否启用】 |

服务端前置条件：GitLab Admin Area 配置 SMTP（内网邮件服务器地址【待确认】），并开启 `Incoming email`（可选，支持邮件回复 Issue）。

规则细化：
- 邮件主题一律带 `[ace]` 前缀 + 仓库 path，方便过滤。
- 邮件正文包含：pipeline 链接（`$CI_PIPELINE_URL`）、失败 job 名、最近 commit（`$CI_COMMIT_TITLE` by `$CI_COMMIT_AUTHOR`）。
- 禁止用邮件代替 MR 评审结论；评审意见必须落在 MR 讨论串。

---

## 5. VSCode 开发与调试环境

### 5.1 工具链（Windows 便携版，无需管理员权限）

| 工具 | 版本 | 路径 |
|---|---|---|
| CMake | 4.4.3 | `D:\debugging\tools\cmake\bin` |
| MinGW-w64 GCC (UCRT, POSIX, WinLibs) | 16.2.0 | `D:\debugging\tools\mingw64\bin` |
| GDB | 随 MinGW-w64 | `D:\debugging\tools\mingw64\bin\gdb.exe` |

使用前加入 PATH：

```bash
export PATH="/d/debugging/tools/cmake/bin:/d/debugging/tools/mingw64/bin:$PATH"
```

【待确认】Linux/EDA 主机侧使用系统 GCC 10.2；两平台构建统一走 Makefile（见第 6 章），CMake 仅用于本机快速原型。

### 5.2 必装插件

| 插件 | 用途 |
|---|---|
| C/C++ (ms-vscode.cpptools) | IntelliSense、GDB 调试 |
| C/C++ Extension Pack | 附带 CMake Tools、Themes |
| CMake Tools | 配置/构建/调试 CMake 工程 |
| Makefile Tools | Makefile 工程识别与 target 执行 |
| GitLens | blame、提交历史、行级追溯 |
| GitLab Workflow | 在 VSCode 内查看 MR、pipeline、Issue |
| Clang-Format（随 cpptools） | 保存时格式化 |
| SystemVerilog 插件（如 eirikpre.systemverilog / Verible）【待确认，仅验证仓库需要】 | SV 语法高亮、跳转、lint |
| Code Spell Checker | message/注释拼写 |

团队统一：仓库内提交 `.vscode/extensions.json`（推荐插件列表）与 `.vscode/settings.json`（格式化、排除目录），克隆即提示安装，保证环境一致。

### 5.3 调试配置（launch.json）

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "gdb 调试当前 target",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/bin/app.exe",   // 按实际产物路径修改
      "args": [],
      "cwd": "${workspaceFolder}",
      "environment": [{ "name": "PATH", "value": "D:\\debugging\\tools\\mingw64\\bin;${env:PATH}" }],
      "MIMode": "gdb",
      "miDebuggerPath": "D:\\debugging\\tools\\mingw64\\bin\\gdb.exe",
      "setupCommands": [
        { "text": "-enable-pretty-printing" },
        { "text": "set print pretty on" }
      ],
      "preLaunchTask": "make debug"
    },
    {
      "name": "attach 到运行中进程",
      "type": "cppdbg",
      "request": "attach",
      "program": "${workspaceFolder}/build/bin/app.exe",
      "MIMode": "gdb",
      "miDebuggerPath": "D:\\debugging\\tools\\mingw64\\bin\\gdb.exe"
    }
  ]
}
```

配套 `tasks.json` 提供 `make debug` / `make release` / `make test` 三个 preLaunchTask。

### 5.4 调试方法与规范

- **构建带调试信息**：调试一律用 `make debug`（`-O0 -g3 -gdwarf-4`）；禁止在 `-O2` 产物上排查逻辑问题（变量被优化掉、行号漂移）。GCC 16 默认 DWARF 5，若 GDB 版本旧则显式 `-gdwarf-4`。
- **断点**：
  - 普通断点 F9；条件断点（右键 → Add Conditional Breakpoint）用于循环中特定迭代，如 `i == 137`；
  - 数据断点（监视内存写）：调试面板 Breakpoints → `+` → Data Breakpoint，填表达式与字节数，用于抓"谁改了这个变量"；
  - `catch throw` / `catch signal`（GDB）在 Debug Console 里执行，定位异常抛出点。
- **查看内存/变量**：
  - Variables / Watch 窗口：加入表达式实时监视；STL 容器依赖 pretty-printer（setupCommands 已开启）；
  - Memory 面板：Debug Console 执行 `-data-read-memory-bytes 0xADDR 64`，或对指针变量右键 → "View Memory"；
  - Debug Console 可直接执行 GDB 命令（`-exec x/16xw ptr`、`-exec p *this`、`-exec bt`）。
- **常用手段**：Call Stack 回溯、Step Out 跳出深调用、Logpoint（不中断打日志，右键行号 → Add Logpoint）、`rr`/时间旅行调试暂不引入【待确认需求】。
- **远程调试（EDA 主机）**：Linux 侧 `gdbserver :1234 ./simv`，本机 launch.json 用 `"miDebuggerServerAddress": "<eda-host>:1234"` 的 remote attach 配置；源码路径用 `sourceFileMap` 映射。
- **core dump 分析**：Linux 侧 `ulimit -c unlimited`，事后 `gdb ./app core` → `bt full`。

---

## 6. 基于 Makefile 的构建脚本规范

原则：**入口统一为 make**，屏蔽底层工具差异；Windows（MinGW make）与 Linux（GNU make）用同一份 Makefile。

### 6.1 目录结构

```
<repo>/
├── Makefile            # 顶层入口，只定义 target 与依赖
├── mk/                 # 可复用片段
│   ├── common.mk       # 工具链探测、通用 flag
│   ├── rules.mk        # 模式规则（%.o: %.cpp 等）
│   └── regress.mk      # 回归列表驱动
├── src/
├── include/
├── tests/
│   ├── smoke.list
│   └── regression.list
└── build/              # 全部产物，git 忽略
    ├── obj/  bin/  lib/  logs/  coverage/
```

### 6.2 标准 target（所有仓库必须实现）

| Target | 行为 |
|---|---|
| `make` / `make all` | 等价 `make release` |
| `make debug` | `-O0 -g3` 构建到 `build/debug/` |
| `make release` | `-O2` 构建到 `build/release/` |
| `make test` | 构建 debug 版并跑 smoke list |
| `make regress LIST=<file>` | 按列表跑回归，结果汇总到 `build/logs/` |
| `make lint` | 风格检查 + 静态 lint，零副作用 |
| `make format` | 就地格式化（clang-format / verible-format） |
| `make coverage-report` | 生成覆盖率 HTML 到 `build/coverage/` |
| `make clean` / `distclean` | 清产物 / 连 build 目录一起清 |
| `make help` | 列出全部 target 及说明（默认 target 设为 help） |

### 6.3 编写规则

```makefile
# mk/common.mk 片段示意
CXX      ?= g++
BUILD    ?= release                      # make BUILD=debug 覆盖
ifeq ($(BUILD),debug)
  CXXFLAGS += -O0 -g3 -gdwarf-4 -DDEBUG
else
  CXXFLAGS += -O2 -DNDEBUG
endif
CXXFLAGS += -std=c++17 -Wall -Wextra -Werror
```

- 工具链变量一律 `?=` 可覆盖，禁止硬编码绝对路径（CI 与个人机路径不同）。
- 开启 `-Wall -Wextra -Werror`【待确认：存量代码可先降为 warning】。
- 用 `-MMD -MP` 自动生成头文件依赖，禁止手写依赖列表。
- 支持并行：所有规则必须 `make -j$(nproc)` 安全（产物路径不冲突）。
- 每条命令前缀 `@` 减少回显噪音；关键步骤 echo 一行人类可读说明。
- Makefile 缩进必须是 TAB，`.editorconfig` 中显式声明，防止 VSCode 转空格。
- 回归列表（`*.list`）是纯文本一行一个 case，`regress.mk` 读取驱动，新增用例不改 Makefile。
- CI 与本地使用完全相同的 target（CI 里就是 `make lint && make test && make regress ...`），杜绝"CI 专用脚本"与本地行为漂移。

---

## 7. 附录：待确认事项清单

| # | 事项 | 建议 |
|---|---|---|
| 1 | `ace-core` 装的是公共库还是 DUT RTL | 影响仓库职责划分与 CI lint 工具选择 |
| 2 | 是否启用 subgroup | 初版不建，仓库 > 6 再拆 |
| 3 | 合并策略 squash vs merge commit | 建议 squash |
| 4 | commit subject 强制英文与否 | 建议英文 |
| 5 | 是否维护 release/* 多版本分支 | 初版只有 main |
| 6 | Runner executor（shell vs docker）与 license 池 | 取决于 EDA 工具授权 |
| 7 | nightly 时间与服务器时区 | 建议 02:00 |
| 8 | 组邮件列表地址与 SMTP 服务器 | 需 IT 分配 |
| 9 | `-Werror` 是否一步到位 | 建议新代码强制、存量限期整改 |
| 10 | SystemVerilog 工具链（VCS/Xcelium/Questa?）与 verible lint 规则集 | 影响 lint/regress stage 具体命令 |
