# 软件开发环境规范（Specification）

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 软件开发环境规范 |
| 文档标识 | SW-ENV-SPEC |
| 版本 | v0.1（初稿） |
| 状态 | 讨论稿，待评审 |
| 日期 | 2026-10-29 |
| 适用范围 | 团队内所有软件仓库：代码托管、评审、CI/CD、通知、本地调试与构建 |
| 配套文件 | 本文档不包含具体业务实现，只定义环境与流程约束 |

---

## 0. 文档说明

### 0.1 目的

本文档定义团队软件开发环境的统一规范，解决以下问题：

1. 代码托管与协作平台的选择依据和部署形态；
2. 仓库（Group / Subgroup / Project）如何命名与分层；
3. 分支、提交、合并请求的书写规则，保证历史可读、可追溯；
4. 持续集成流水线在什么时机跑什么任务，以及质量门禁的边界；
5. 邮件通知"谁在什么时候收到什么"，避免漏通知与告警疲劳；
6. 每位开发者本地调试环境的统一搭法（VSCode + GDB）；
7. 基于 Makefile 的统一构建入口，保证本地与 CI 行为一致。

### 0.2 适用对象

- 新加入团队的开发者：按本文档从零搭建环境；
- 项目维护者：按本文档配置仓库、分支保护和流水线；
- 平台/运维人员：按本文档部署 GitLab、Runner、SMTP 与备份。

### 0.3 术语

| 术语 | 含义 |
| --- | --- |
| Group / Subgroup | GitLab 的命名空间层级，用于组织仓库与统一管理权限 |
| Project | 一个 Git 仓库，可独立构建、测试、发布的最小单元 |
| MR | Merge Request，合并请求 |
| Pipeline | 一次完整的 CI/CD 执行，包含若干 Stage |
| Runner | 执行 Pipeline 任务的机器或容器 |
| Nightly | 每晚定时触发的全量构建与回归流水线 |
| Trunk-based | 以主干为主线、短生命周期分支的开发模型 |

### 0.4 版本记录

| 版本 | 日期 | 修改人 | 说明 |
| --- | --- | --- | --- |
| v0.1 | 2026-10-29 | — | 初稿，覆盖 7 个主题，待评审确认第 9 章事项 |

> 命名示例（如 `icdev`、`verif`）仅用于演示规则，最终名称在落地时按第 9 章确认后固化。

---

## 1. 为什么选择 GitLab

### 1.1 选型需求

在选型前先明确我们真正需要的能力，避免"为了工具而工具"：

| 编号 | 需求 | 权重 | 说明 |
| --- | --- | --- | --- |
| R1 | 代码不出内网 | 高 | 源码、构建产物、测试数据均受控 |
| R2 | 代码评审可追溯 | 高 | 谁改的、谁批的、跑了什么检查，必须留痕 |
| R3 | CI 能跑在构建机/仿真机旁 | 高 | 编译依赖许可证服务、专用硬件、大内存机器 |
| R4 | 权限与审计满足客户/合规要求 | 高 | 分支保护、审批规则、操作日志 |
| R5 | 一体化，减少工具拼接 | 中 | 减少账号、权限、通知的多套体系 |
| R6 | 自托管、可备份、可迁移 | 高 | 数据主权与长期可用性 |
| R7 | 团队学习成本可控 | 中 | 开发者日常只需要 MR + Pipeline 两个概念 |

### 1.2 选型结论

**采用自托管 GitLab（Community Edition 起步，按需升级）作为唯一的代码托管与 CI/CD 平台。**
GitLab 同时覆盖 R1–R7，且不需要把"仓库、评审、CI、制品、文档"拆到四五个系统里维护。

### 1.3 关键理由

1. **一体化，降低协作摩擦**
   仓库、MR、Code Review、Issue、CI/CD、制品库（Package Registry）、Wiki 在同一平台。评审意见、流水线结果、合入记录天然关联，不需要跨系统拼链接。

2. **可完全内网部署**
   GitLab 支持 Omnibus、Docker、Kubernetes 多种部署形态，适合隔离网络。代码和 CI 日志不出内网，满足 R1。

3. **Runner 可以部署在"正确的位置"**
   这是最关键的一点：编译需要许可证服务器，回归需要仿真器或专用板卡。GitLab Runner 可以用标签（tag）把任务调度到指定机器，而不是强迫把代码搬到公有云。相比之下，纯 SaaS 方案在这一层几乎无解。

4. **CI 配置与代码同源、版本化**
   `.gitlab-ci.yml` 跟随仓库走，改流水线和改代码在同一个 MR 里评审，历史可回溯。

5. **权限模型足够细**
   Group / Project 两级成员权限、Protected Branch、Push Rules、Approval Rules、CODEOWNERS、Protected Environment，能表达"核心模块必须两人批准""只有维护者能打 tag"这类规则。

6. **审计与合规**
   Audit Events 记录成员变更、权限变更、保护分支变更、导出等敏感操作，便于应对审计。

7. **迁移与集成路径清晰**
   支持从 SVN、其他 Git 平台导入；支持 LDAP/AD 统一登录；支持 Webhook 对接内部通知系统。

8. **成本可控**
   社区版自托管无许可费用，主要成本是服务器资源与运维人力；企业版可按需采购。

### 1.4 与备选方案对比

| 维度 | GitLab（自托管） | GitHub Enterprise | Gitea + Jenkins | SVN + 自建脚本 |
| --- | --- | --- | --- | --- |
| 仓库 + 评审 + CI 一体化 | 强 | 强 | 弱（需拼接） | 无 |
| 内网/隔离网络部署 | 支持 | 支持（成本高） | 支持 | 支持 |
| CI 调度到专用构建机 | 原生支持（Runner 标签） | 需自建 Runner，配置较繁 | 需自行设计 | 需自行设计 |
| 分支保护与审批规则 | 完善 | 完善 | 基础 | 无 |
| 审计日志 | 完善 | 完善 | 基础 | 需自建 |
| 运维成本 | 中 | 中高 | 低（但工具多、集成成本高） | 低（但能力弱） |
| 学习曲线 | 中 | 低 | 低 | 低 |
| 适合场景 | 内网研发 + 专用构建资源 | 已深度使用 GitHub 生态 | 小团队、轻量需求 | 仅需版本管理 |

**结论**：Gitea + Jenkins 在"轻量"场景有优势，但评审、权限、审计、制品管理需要大量自建胶水；GitHub Enterprise 在公网/混合云场景合适，但内网 Runner 与合规成本不占优。我们的场景（内网 + 专用构建资源 + 审计要求）与 GitLab 最匹配。

### 1.5 代价与应对（必须正视）

| 代价 | 具体表现 | 应对措施 |
| --- | --- | --- |
| 运维负担 | 升级、备份、存储增长、故障恢复 | 指定责任人；每日备份并每季度做恢复演练；升级先在测试实例验证 |
| 资源占用 | 主实例建议 ≥ 4 vCPU / 8 GB RAM，随仓库与流水线增长 | 与 Runner 分离部署；制品设置过期策略；监控磁盘水位 |
| 版本迭代快 | 每年多个版本，升级节奏紧 | 固定"每半年升级一次小版本"的节奏，不追最新 |
| Runner 安全 | Shell Executor 隔离弱，任务可影响宿主机 | 优先 Docker Executor；必须用 Shell 时，Runner 机器只跑 CI，不放敏感数据 |
| 配置复杂 | 权限、保护分支、审批规则组合多 | 用模板项目 + 组级设置统一，避免每个项目手配 |

### 1.6 部署形态建议

| 组件 | 建议 |
| --- | --- |
| GitLab 主实例 | 独立服务器，Docker 或 Omnibus，域名走内网 HTTPS |
| 存储 | 仓库与制品分盘；开启对象存储（可选） |
| Runner（通用） | Docker Executor，标签 `linux-x86_64`，负责编译与单元测试 |
| Runner（回归） | Shell Executor，标签 `regress`，靠近仿真器/板卡，限制并发为 1 |
| Runner（大内存） | 标签 `bigmem`，仅 nightly 使用，避免占用日常资源 |
| 备份 | 每日全量 + 保留 30 天；异地一份；恢复演练季度一次 |
| 监控 | 磁盘、内存、Pipeline 排队时长、Runner 在线状态 |

---

## 2. 命名规范：Group / Subgroup / Project

### 2.1 层级模型

采用"组织 → 产品线 → 组件"三层，最多不超过三层：

```
<org>                        组织/部门级命名空间（固定，不随意新增）
└── <product-line>           产品线或项目群
    ├── <product>-<component>    组件仓库
    ├── <product>-<component>    组件仓库
    └── <product>-tests          集成测试/回归仓库
```

原则：

- **Group 表达"归属"**（谁负责、属于哪个产品），不表达技术栈（不要 `cpp-group`、`python-group`）。
- **Project 表达"可独立构建发布的最小单元"**。如果一个目录不能独立构建、独立测试、独立发版，它就不该是独立仓库。
- **层级不是越深越好**。超过三层通常意味着产品划分不清，应先在 Group 内用 Project 区分，而不是继续嵌套。

### 2.2 通用字符规则

Group、Subgroup、Project 的 path 统一遵守：

| 规则 | 说明 |
| --- | --- |
| 字符集 | 仅小写字母 `a-z`、数字 `0-9`、连字符 `-` |
| 首尾 | 不以 `-` 开头或结尾 |
| 连续符号 | 不允许 `--` |
| 长度 | ≤ 30 个字符（Project path 建议 ≤ 40） |
| 语言 | 禁止中文、空格、下划线、点号、驼峰 |
| 缩写 | 避免与领域内已有缩写冲突（协议名、工具名、标准名），冲突时改用完整单词 |
| 版本号 | 不写进名字（不要 `tool-v2`、`tool-2024`），版本用 tag 表达 |
| 人员 | 不出现人名、工号 |
| 保留字 | 避开 GitLab 保留路径，如 `admin`、`api`、`help`、`uploads`、`assets` |

> 说明：GitLab 的 URL path 与显示名可以不同，但规范要求**显示名与 path 完全一致**，避免"搜不到、链接对不上"。

### 2.3 Group / Subgroup 命名

- 一级 Group 固定为组织缩写（示例：`icdev`），由平台管理员统一创建，普通成员无权新建。
- 二级 Subgroup 对应产品线（示例：`verif`、`soc`），由产品线负责人申请创建。
- 三级 Subgroup 仅在确实需要"再分一层权限边界"时创建，例如外包协作、跨部门共建。
- Subgroup 名称使用完整单词或团队公认且无歧义的缩写；一个缩写只允许一种含义。
- 每个 Group 必须填写 Description（负责团队 + 联系方式），并设置默认分支保护策略。

### 2.4 Project 命名

- 格式：`<产品>-<组件>`（在二级 Group 下可省略产品前缀，避免重复）。
- 名称体现职责，不体现实现：用 `packet-parser`，不用 `cpp-parser`。
- 单一职责：一个仓库对应一个可交付物（库、可执行程序、测试集、文档站）。
- 配套仓库：测试/回归使用 `<产品>-tests`；基础设施使用 `<产品>-infra`；不要建 `misc`、`tmp`、`test1` 这类垃圾仓。
- 归档仓库使用 GitLab 的 Archive 功能，**不要改名加 `-old`**，否则历史链接全部失效。

### 2.5 示例

| 类型 | 正例 | 反例 | 原因 |
| --- | --- | --- | --- |
| Group | `icdev` | `IC_Dev` | 含大写与下划线 |
| Subgroup | `verif` | `Verification_Platform` | 含大写、下划线、过长 |
| Project | `verif-core` | `CoreLib_V2` | 含大写、版本号 |
| Project | `verif-tests` | `test` | 无归属、无含义 |
| Project | `packet-parser` | `pp` | 缩写歧义 |
| Project | `build-infra` | `infra-new` | 用 `new` 表达版本 |
| 分支 | `feature/142-uart-timeout` | `dev_zhangsan` | 人名、下划线、无 issue |
| Tag | `v1.4.0` | `release_1.4` | 格式不统一 |

### 2.6 仓库初始化要求

每个新仓库创建后必须包含：

1. `README.md`：一句话说明用途、构建命令、责任人、依赖；
2. `.gitignore`：至少忽略 `out/`、`build/`、`*.o`、`*.d`、IDE 本地配置；
3. `.gitlab-ci.yml`：从模板复制，不允许为空；
4. `Makefile`：提供 `all` / `clean` / `test` 三个基础目标；
5. `CODEOWNERS`：核心目录必须有 owner；
6. `.editorconfig`、`.clang-format`（C/C++ 项目）；
7. MR 模板与 Issue 模板（可放在组级模板仓库统一维护）。

默认分支统一为 `main`，创建时即设置为 Protected。

---

## 3. 分支与提交规范

### 3.1 分支模型

采用 **Trunk-based + 短生命周期分支**，不采用重量级 GitFlow：

- `main` 是唯一长期分支，始终保持可构建、可发布；
- 所有改动通过短分支 + MR 合入；
- 分支存活时间目标 ≤ 5 个工作日，超过需拆分或说明原因；
- 仅在需要并行维护旧版本时创建 `release/x.y` 长期分支。

| 分支 | 类型 | 来源 | 合并目标 | 说明 |
| --- | --- | --- | --- | --- |
| `main` | 长期 | — | — | 受保护，禁止直推 |
| `release/x.y` | 长期（按需） | `main` 或 tag | `main`（仅修复合回） | 旧版本维护，受保护 |
| `feature/*` | 短期 | `main` | `main` | 新功能 |
| `bugfix/*` | 短期 | `main` | `main` | 非紧急缺陷修复 |
| `hotfix/*` | 短期 | `main` 或 tag | `main` + `release/*` | 线上/现场紧急修复 |
| `chore/*` | 短期 | `main` | `main` | 构建、依赖、配置调整 |
| `docs/*` | 短期 | `main` | `main` | 纯文档 |
| `ci/*` | 短期 | `main` | `main` | 流水线调整 |
| `exp/*` | 短期 | `main` | 不合入 | 技术验证，验证完删除 |

### 3.2 分支命名

格式：

```
<type>/<issue-id>-<kebab-case-slug>
```

规则：

- `<type>` 取上表前缀，全小写；
- `<issue-id>` 为 GitLab Issue 编号（无 Issue 的 `docs`、`ci` 可省略）；
- slug 用 2–5 个英文单词描述改动，全小写、连字符分隔；
- 总长度 ≤ 50 字符；
- 禁止：中文、空格、下划线、大写、人名、`tmp`、`test`、`new`。

示例：

```
feature/142-uart-timeout
bugfix/201-null-deref-on-empty-input
hotfix/315-license-check-crash
chore/88-bump-toolchain
docs/90-update-build-guide
ci/102-cache-ccache
```

### 3.3 分支保护（`main` 与 `release/*`）

必须开启：

| 配置项 | 取值 |
| --- | --- |
| Allowed to push | No one（仅通过 MR） |
| Allowed to merge | Maintainer |
| Force push | 禁止 |
| 删除分支 | 禁止 |
| MR 必须通过 Pipeline | 是 |
| 至少批准数 | 1（核心模块 2） |
| 未解决讨论 | 必须全部解决才能合并 |
| Squash 合并 | 允许，且为默认 |
| 合并后删除源分支 | 自动 |

### 3.4 Merge Request 规范

- **标题**遵循 Commit 规范（squash 后它就是 `main` 上的提交信息）。
- **描述**必须包含：改了什么、为什么改、怎么验证、关联 Issue（`Closes #123`）、风险与回滚方式。
- **粒度**：一个 MR 只做一件事；目标改动 ≤ 400 行（不含生成代码与测试数据），超出应拆分。
- **Draft MR**：未完成时用 `Draft:` 前缀，避免被误评审；可配置 Draft 不触发完整流水线。
- **评审 SLA**：提交后 24 小时内必须有首次反馈；阻塞性问题用 `blocking` 标记。
- **合并策略**：功能分支用 **Squash merge**，保持 `main` 线性历史；`release` 回合 `main` 用 Merge commit，保留来源。
- **合并前**：Pipeline 全绿、审批满足、讨论解决、目标分支已同步（rebase 或 merge）。
- **禁止**：自审自合（特殊紧急情况需在 MR 中记录原因并事后补审）。

### 3.5 Commit Message 规范

采用 Conventional Commits 1.0.0：

```
<type>(<scope>)!: <subject>

<body>

<footer>
```

**type**（必填，英文小写）：

| type | 用途 |
| --- | --- |
| `feat` | 新功能 |
| `fix` | 修复缺陷 |
| `docs` | 文档 |
| `style` | 格式调整（不影响逻辑） |
| `refactor` | 重构（不改行为） |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `build` | 构建系统、依赖 |
| `ci` | 流水线配置 |
| `chore` | 杂项维护 |
| `revert` | 回滚 |

**scope**（可选）：模块名，如 `parser`、`uart`、`build`、`ci`。

**subject**：英文，祈使句，首字母小写，结尾不加句号，≤ 72 字符。

**body**：说明"为什么"而不是"做了什么"（代码本身说明了 what）；每行 ≤ 100 字符；可用中文。

**footer**：关联 Issue、破坏性变更、共同作者。

正例：

```
feat(parser): support nested include paths

原实现只解析单层 include，遇到嵌套头文件会静默截断，
导致部分配置项失效且无任何告警。

改为递归解析并增加深度上限（默认 32），超限返回 E_DEPTH_LIMIT。

Closes #142
```

```
fix(uart): avoid blocking read when peer disconnects

BREAKING CHANGE: read_timeout() 现在返回 ETIMEDOUT 而非 0
```

反例（禁止）：

```
update
fix bug
修改
临时提交
1
```

其他约束：

- 一个 commit 只做一件事；纯格式化改动单独提交，避免污染 diff；
- 不提交构建产物、日志、临时文件、IDE 本地配置；
- 二进制/大文件走 Git LFS，并在 README 说明；
- 重要提交使用 `Signed-off-by`（DCO），便于追溯来源；
- **禁止 amend 已推送的公共分支提交**；个人分支可 rebase，但合并前需说明。

---

## 4. GitLab Pipeline

### 4.1 设计原则

1. **单一入口**：所有编译、测试、检查都通过 `make` 目标执行，CI 只做编排，不在 YAML 里写业务命令。
2. **快反馈**：MR Pipeline 目标 ≤ 15 分钟；超过视为失败，需优化或拆分。
3. **一致性**：CI 与本地使用同一镜像/同一工具链版本，避免"本地能编、CI 编不过"。
4. **可重现**：构建不依赖 Runner 本地残留状态；Cache 只用于加速，不参与正确性。
5. **最小权限**：普通 Job 只读；发布 Job 使用 Protected Environment + 单独 token。
6. **失败可诊断**：保留 Job 日志与报告 Artifact，失败摘要直接可读。

### 4.2 Stage 划分

| Stage | 目的 | 典型 Job | MR | main | Nightly | Tag |
| --- | --- | --- | --- | --- | --- | --- |
| `build` | 编译 | `build:debug`、`build:release` | 是 | 是 | 是 | 是 |
| `test` | 单元测试、集成测试 | `test:unit`、`test:integration` | 是 | 是 | 是 | 是 |
| `regress` | 长回归、仿真 | `test:regression` | 否 | 否 | 是 | 是 |
| `analyze` | 静态分析、内存检查 | `analyze:static`、`analyze:sanitize` | 部分 | 部分 | 是 | 是 |
| `package` | 打包、校验和 | `package:tarball` | 否 | 否 | 否 | 是 |
| `release` | 发布、归档 | `release:publish` | 否 | 否 | 否 | 是（手动） |

### 4.3 触发方式

| 触发源 | 条件 | 跑什么 | 目的 |
| --- | --- | --- | --- |
| Merge Request 事件 | 创建、更新、重新运行 | 增量/快速全量：build + unit + 轻量静态检查 | 合入前门禁 |
| Push 到 `main` | 合并后 | build + unit + 轻量检查 | 保证主干始终健康 |
| 定时任务（Nightly） | 每天 01:00 | 全量 build（debug+release）+ 全量回归 + 全量静态分析 + Sanitizer | 深度质量保障 |
| 定时任务（Weekly） | 每周六 02:00 | 依赖漏洞扫描、许可证检查、长时压力测试 | 周期性治理 |
| Tag 推送 | `v*.*.*` | build + test + package + 手动发布 | 版本交付 |
| 手动触发 | Webhook / 手动 Run pipeline | 按变量选择 | 现场问题复现、临时验证 |

### 4.4 MR Pipeline

要求：

- 每个 MR 必须产生 Pipeline，且作为合并门禁（"Pipelines must succeed"）。
- 只跑与改动相关的必要任务，长回归不进 MR。
- 允许 `allow_failure: true` 的仅限非门禁项（如覆盖率报告、代码风格提示）；**编译、单元测试、静态检查的阻断项不得 allow_failure**。
- 同一分支的新提交自动取消旧 Pipeline（`interruptible: true`），节省资源。
- 目标时长 ≤ 15 分钟；单元测试优先跑受影响模块，全量留给 Nightly。

### 4.5 主干 Pipeline

- 合并到 `main` 后自动触发，结果作为"主干健康度"指标。
- `main` 变红视为最高优先级事件：**停止合并新 MR，先修复**（Stop the line）。
- 30 分钟内无法修复的，回滚最近一次合并，再排查。
- 主干 Pipeline 失败必须通知提交者与 Maintainer（见第 5 章）。

### 4.6 Nightly Pipeline

配置位置：`Settings > CI/CD > Pipeline schedules`。

| 项目 | 建议值 |
| --- | --- |
| 频率 | 每天 01:00（`0 1 * * *`），时区 `Asia/Shanghai` |
| 目标分支 | `main` |
| 变量 | `NIGHTLY=1`，用于 Job 规则判断 |
| 内容 | debug + release 全量构建、全量单元测试、长回归、ASan/UBSan、全量静态分析、覆盖率 |
| 超时 | 回归 Job 单独放宽（如 4 小时），其余 30 分钟 |
| Artifact | 报告保留 30 天；失败日志保留 90 天 |
| 失败处理 | 生成汇总，次日 08:30 前发出（见第 5 章） |

Nightly 的定位是"发现问题"，不是"阻塞开发"，因此：

- 失败不回滚 `main`，但必须当日定位；
- 偶发失败（flaky）必须登记并隔离（quarantine），不允许长期靠重试掩盖。

### 4.7 Tag / Release Pipeline

- Tag 格式：`vMAJOR.MINOR.PATCH`（如 `v1.4.0`），保护 Tag，仅 Maintainer 可创建。
- Tag Pipeline 产出：源码包、二进制包、校验和（SHA256）、Changelog、版本清单（编译器版本、依赖版本）。
- 发布 Job 设为 `when: manual`，并绑定 Protected Environment，需审批后执行。
- 制品命名：`<project>-<version>-<os>-<arch>.tar.gz`，内容可复现。

### 4.8 参考 `.gitlab-ci.yml`

以下为骨架，实际项目按需裁剪：

```yaml
stages:
  - build
  - test
  - analyze
  - package

variables:
  GIT_DEPTH: "20"
  GIT_STRATEGY: fetch

default:
  tags: ["linux-x86_64"]
  interruptible: true
  timeout: 20m

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_COMMIT_TAG'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# ---------- 构建 ----------
.make:
  stage: build
  script:
    - make BUILD=$BUILD -j"$(nproc)"
  artifacts:
    paths:
      - out/$BUILD/bin/
    expire_in: 3 days

build:debug:
  extends: .make
  variables:
    BUILD: debug
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    - if: '$CI_PIPELINE_SOURCE == "schedule"'

build:release:
  extends: .make
  variables:
    BUILD: release
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_COMMIT_TAG'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
      allow_failure: true

# ---------- 测试 ----------
test:unit:
  stage: test
  needs: ["build:debug"]
  script:
    - make BUILD=debug test
  artifacts:
    when: always
    reports:
      junit: out/debug/reports/*.xml
    expire_in: 7 days

test:regression:
  stage: test
  needs: ["build:release"]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_COMMIT_TAG'
  script:
    - make BUILD=release regression
  timeout: 4h
  artifacts:
    when: always
    paths:
      - out/release/reports/
    expire_in: 30 days

# ---------- 分析 ----------
analyze:static:
  stage: analyze
  needs: ["build:debug"]
  script:
    - make analyze
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  # 门禁项：失败即阻断，不允许放过
  allow_failure: false

analyze:sanitize:
  stage: analyze
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  script:
    # 必须 clean：复用普通构建的目标文件不会带上 sanitizer 参数
    - make clean
    - make SAN="-fsanitize=address,undefined -fno-omit-frame-pointer" test

# ---------- 打包 ----------
package:tarball:
  stage: package
  needs: ["build:release", "test:unit"]
  rules:
    - if: '$CI_COMMIT_TAG'
  script:
    - make BUILD=release package
    - sha256sum out/release/dist/*.tar.gz | tee out/release/dist/SHA256SUMS
  artifacts:
    paths:
      - out/release/dist/
    expire_in: never
```

> 说明：`make analyze` 在 MR 上建议只检查本次改动文件（diff-based），全量检查放 Nightly，以控制时间。实现方式由 `Makefile` 中的 `analyze` 目标决定。

### 4.9 Runner 与资源

| Runner | 标签 | Executor | 用途 | 并发 |
| --- | --- | --- | --- | --- |
| runner-general | `linux-x86_64` | docker | 编译、单元测试、静态分析 | 4–8 |
| runner-regress | `regress` | shell | 长回归、仿真、板卡 | 1 |
| runner-bigmem | `bigmem` | shell | Nightly 大内存任务 | 1 |

约束：

- 使用 Docker Executor 时，镜像固定到 digest，避免"今天和昨天环境不同"；
- 回归机使用 `resource_group` 串行化，避免多个 Job 抢同一块板卡/许可证；
- Shell Executor 的机器不做开发用途，不存放个人数据；
- Runner 定期更新，离线超过 30 分钟告警（见第 5 章）；
- Job 镜像中预装编译器、make、gdb、clang-tidy、bear 等固定版本工具链。

### 4.10 失败处理

- 失败原因分类：代码问题、测试问题、环境问题、资源问题；
- **仅环境/资源问题允许重试**（`retry: when: [runner_system_failure, stuck_or_timeout]`，最多 2 次）；
- flaky 测试不允许无限重试，必须标记并分配责任人修复；
- 每次 `main` 失败记录到"主干健康"看板，月度复盘；
- Pipeline 日志与报告是排障第一现场，禁止只发截图不看日志。

---

## 5. 邮件提醒规则

### 5.1 通道与范围

- GitLab 邮件使用内网 SMTP，发件人 `gitlab-noreply@<domain>`；
- 管理员配置 SPF/DKIM，避免进入垃圾箱；
- 项目级使用组邮箱别名（如 `verif-dev@<domain>`）而不是个人列表，人员变动不改规则；
- 邮件只做"通知"，讨论与结论必须回到 MR / Issue，避免结论散落在邮件里。

### 5.2 通知矩阵

| 事件 | 触发时机 | 收件人 | 渠道 | 频率 |
| --- | --- | --- | --- | --- |
| MR 被指派/请求评审 | 实时 | Assignee、Reviewer | 邮件 | 即时 |
| MR 有新评论或要求修改 | 实时 | MR 作者 | 邮件 | 即时 |
| MR 已合并 | 实时 | 作者、Reviewer | 邮件 | 即时 |
| MR 超过 48 小时未合并 | 每日 09:30 | 作者、Reviewer、组长 | 汇总邮件 | 每日 1 封 |
| MR Pipeline 失败 | 实时 | MR 作者 | 邮件 | 每次失败 1 封 |
| `main` Pipeline 失败 | 实时 | 提交者、Maintainer、值班人 | 邮件 + IM | 即时 |
| Nightly 失败 | 次日 08:30 | 项目全员（组别名）、值班人 | 汇总邮件 | 每日 1 封 |
| 覆盖率下降超过阈值 | 次日 08:30 | 项目全员 | 汇总邮件 | 每日 1 封 |
| 新建 Tag / 发布 | 实时 | 项目全员 | 邮件 | 即时 |
| 成员/权限/保护分支变更 | 实时 | 项目 Owner、平台管理员 | 邮件 | 即时 |
| Runner 离线 > 30 分钟 | 实时 | 平台管理员 | 邮件 + IM | 每事件 1 封 |
| 仓库 90 天无提交 | 每月 1 日 | Owner | 邮件 | 每月 1 封 |

### 5.3 实现方式（原生能力与自建脚本的边界）

必须明确：**GitLab 的邮件通知以"个人偏好"为主，无法强制全员**。因此分两层实现：

| 层 | 能力 | 配置位置 | 适用规则 |
| --- | --- | --- | --- |
| 个人通知 | 参与/指派/自己 Pipeline 失败等 | `Preferences > Notifications` | MR 评审、评论、合并 |
| 项目集成 | Push 事件、Pipeline 事件的邮件 | `Settings > Integrations > Emails on push` | `main` 失败、Tag 发布 |
| 组级 Webhook | 转发到内部通知服务 | `Group > Webhooks` | 统一格式、去重、汇总 |
| 定时流水线 | 跑脚本生成日报/周报 | `CI/CD > Schedules` | Nightly 汇总、MR 停留时长 |
| 管理员配置 | SMTP、发件人、频率限制 | `Admin Area > Settings > Email` | 全局 |

建议默认设置：

- 新成员入职时由管理员统一设置为 **Custom**：只接收"被指派/被提及/自己触发的 Pipeline 失败"；
- 关键角色（Maintainer、值班人）必须开启 `main` 失败与 Nightly 失败通知；
- 平台类告警（Runner 离线、磁盘水位）配置到管理员组别名，不发给普通成员。

### 5.4 邮件格式

主题统一模板，便于过滤与检索：

```
[<项目>][<分支或标签>][<状态>] <事件摘要> — Pipeline #<id> / MR !<iid>
```

示例：

```
[verif-core][main][失败] build:debug — Pipeline #1287
[verif-core][!342][需评审] feat(parser): support nested include paths
[verif-core][nightly][汇总] 2026-10-28 回归结果：3 失败 / 214 通过
```

正文必须包含：

1. 触发人、提交 SHA 短号、提交标题；
2. 失败 Job 名称与首条错误信息（不是整段日志）；
3. 直达链接（Job Log、MR、Pipeline）；
4. 建议的下一步（重试、回滚、联系谁）。

禁止：只发"构建失败"四个字、只发截图、附完整日志（用链接代替）。

### 5.5 降噪规则

- 同一 Pipeline 的多个 Job 失败合并为 1 封，不按 Job 逐封发送；
- 非工作时间（22:00–08:00）仅 P0（`main` 失败、发布失败）实时发送，其余进入次日摘要；
- Nightly 结果只发汇总，不逐 Job 发；
- 不使用 `@all` 群发；确需全员知会时由负责人手工发送；
- 邮件不得包含许可证信息、客户数据、完整测试向量；
- 提供"退订"说明：成员可调整个人级别，但值班与 Maintainer 角色不在退订范围内。

### 5.6 值班与响应

| 事项 | 响应时限 | 责任人 |
| --- | --- | --- |
| `main` Pipeline 失败 | 30 分钟 | 提交者，值班人兜底 |
| Nightly 失败 | 次工作日 12:00 前定位 | 对应模块 Owner |
| Runner 离线 | 1 小时 | 平台管理员 |
| MR 评审 | 24 小时 | Reviewer |
| 发布失败 | 立即 | 发布负责人 |

值班按周轮换，值班表在项目 Wiki 公示；值班人负责"确认收到、分派、跟踪关闭"，不要求亲自修复所有问题。

---

## 6. VSCode 调试环境

### 6.1 总体方案

- 开发机（Windows 或 Linux 桌面）通过 **Remote - SSH** 连接编译/运行机（Linux），代码、编译、调试都在远端进行，本地只跑编辑器 UI；
- 编译统一走 `make BUILD=debug`，产物在 `out/debug/`；
- 调试使用 **GDB**（`cppdbg`）；需要 attach 或远程调试时用 `gdbserver`；
- 所有配置（`settings.json`、`tasks.json`、`launch.json`）提交到仓库 `.vscode/`，保证团队一致；
- 个人偏好放用户级设置，仓库级只放与项目相关的配置。

### 6.2 必备软件（远端）

| 软件 | 版本要求 | 说明 |
| --- | --- | --- |
| gcc/g++ | 与 CI 镜像一致（示例 GCC 10.2） | 编译 |
| gdb | ≥ 9.x，建议与 GCC 同源发行版 | 调试 |
| make | ≥ 4.x | 构建 |
| gdbserver | 与 gdb 同版本 | 远程/板卡调试 |
| clangd 或 cpptools 后端 | 可选 | 代码补全 |
| bear | 可选 | 生成 `compile_commands.json` |
| clang-format / clang-tidy | 与项目配置一致 | 格式化与静态检查 |

### 6.3 插件清单

| 插件 | ID | 必装 | 用途 |
| --- | --- | --- | --- |
| C/C++ | `ms-vscode.cpptools` | 是 | IntelliSense、GDB 调试前端 |
| Makefile Tools | `ms-vscode.makefile-tools` | 是 | 识别 Makefile 目标、一键构建 |
| Remote - SSH | `ms-vscode-remote.remote-ssh` | 是（Windows 开发机） | 连接远端开发机 |
| GitLens | `eamodio.gitlens` | 是 | 代码溯源、 blame |
| GitLab Workflow | `GitLab.gitlab-workflow` | 是 | MR、Pipeline、Issue 在编辑器内查看 |
| EditorConfig | `EditorConfig.EditorConfig` | 是 | 统一缩进与换行 |
| clangd | `llvm-vs-code-extensions.vscode-clangd` | 二选一 | 更准的跳转/补全（与 cpptools 的 IntelliSense 互斥） |
| CodeLLDB | `vadimcn.vscode-lldb` | 可选 | LLDB 后端（macOS/部分 Linux） |
| Hex Editor | `ms-vscode.hexeditor` | 可选 | 查看二进制与 core 文件 |
| clang-format | `xaver.clang-format` | 可选 | 保存时格式化 |

> 注意：`cpptools` 与 `clangd` 的 IntelliSense 同时开启会互相干扰，团队二选一并统一。建议：调试用 `cpptools`，代码导航用 `clangd` 时关闭 cpptools 的 IntelliSense（`"C_Cpp.intelliSenseEngine": "disabled"`）。

### 6.4 工作区配置

`.vscode/settings.json`：

```json
{
  "C_Cpp.default.compilerPath": "/usr/bin/g++",
  "C_Cpp.default.cppStandard": "c++17",
  "C_Cpp.default.cStandard": "c11",
  "C_Cpp.default.includePath": [
    "${workspaceFolder}/include",
    "${workspaceFolder}/src"
  ],
  "C_Cpp.default.defines": ["DEBUG"],
  "C_Cpp.default.compileCommands": "${workspaceFolder}/compile_commands.json",
  "C_Cpp.errorSquiggles": "enabled",
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "editor.formatOnSave": false,
  "editor.rulers": [100],
  "files.associations": {
    "*.h": "cpp"
  }
}
```

说明：

- 若存在 `compile_commands.json`，includePath/defines 以它为准，最准确。生成方式：
  ```bash
  make clean
  bear -- make BUILD=debug all
  ```
- `formatOnSave` 默认关闭，避免与 `git diff` 混入无关改动；需要时手动执行 `make format`。

### 6.5 tasks.json

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "make: debug",
      "type": "shell",
      "command": "make",
      "args": ["BUILD=debug", "-j4"],
      "options": { "cwd": "${workspaceFolder}" },
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": ["$gcc"]
    },
    {
      "label": "make: release",
      "type": "shell",
      "command": "make",
      "args": ["BUILD=release", "-j4"],
      "options": { "cwd": "${workspaceFolder}" },
      "problemMatcher": ["$gcc"]
    },
    {
      "label": "make: test",
      "type": "shell",
      "command": "make",
      "args": ["BUILD=debug", "test"],
      "options": { "cwd": "${workspaceFolder}" },
      "group": "test",
      "problemMatcher": ["$gcc"]
    },
    {
      "label": "make: clean",
      "type": "shell",
      "command": "make",
      "args": ["clean"],
      "problemMatcher": []
    }
  ]
}
```

`problemMatcher: $gcc` 能把编译错误解析到编辑器问题面板，点击直达源码行。

### 6.6 launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug: app (gdb)",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/out/debug/bin/app",
      "args": ["--config", "${workspaceFolder}/configs/dev.yaml"],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [
        {
          "description": "启用 GDB 美化打印（STL 容器可读）",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        },
        {
          "description": "Intel 汇编语法",
          "text": "-gdb-set disassembly-flavor intel",
          "ignoreFailures": true
        },
        {
          "description": "关闭分页，避免 --More-- 卡住",
          "text": "-gdb-set pagination off",
          "ignoreFailures": true
        }
      ],
      "preLaunchTask": "make: debug"
    },
    {
      "name": "Attach: 现有进程",
      "type": "cppdbg",
      "request": "attach",
      "program": "${workspaceFolder}/out/debug/bin/app",
      "processId": "${command:pickProcess}",
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [
        { "text": "-enable-pretty-printing", "ignoreFailures": true }
      ]
    },
    {
      "name": "Attach: gdbserver 远程",
      "type": "cppdbg",
      "request": "attach",
      "program": "${workspaceFolder}/out/debug/bin/app",
      "miDebuggerServerAddress": "127.0.0.1:2345",
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [
        { "text": "-enable-pretty-printing", "ignoreFailures": true }
      ]
    },
    {
      "name": "Debug: core dump",
      "type": "cppdbg",
      "request": "attach",
      "program": "${workspaceFolder}/out/debug/bin/app",
      "coreDumpPath": "${workspaceFolder}/core/app.core",
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "setupCommands": [
        { "text": "-enable-pretty-printing", "ignoreFailures": true }
      ]
    }
  ]
}
```

要点：

- `program` 必须与 `make` 产物路径一致（第 7 章约定 `out/<build>/bin/`）；
- `preLaunchTask` 让 F5 自动重新编译，避免调试旧二进制；
- 源码路径与编译机不一致时，用 `sourceFileMap` 映射；
- 板卡/远端调试时，远端执行 `gdbserver :2345 ./app`，本机用 Attach 配置连接。

### 6.7 断点

| 类型 | 设置方式 | 适用场景 |
| --- | --- | --- |
| 行断点 | 行号左侧单击（F9） | 常规 |
| 条件断点 | 右键 → Edit Breakpoint → Expression | 循环中特定条件，如 `i == 128` |
| 命中计数断点 | Edit Breakpoint → Hit Count | 第 N 次进入时停下 |
| 日志断点（Logpoint） | Edit Breakpoint → Log Message | 不中断，只打印变量，适合时序敏感场景 |
| 函数断点 | Debug Console: `-exec break func_name` | 无源码或不确定行号 |
| 数据断点（Watchpoint） | `-exec watch var` / `-exec watch *(int*)addr` | 变量被意外改写，定位"谁改的" |
| 异常断点 | `-exec catch throw`、`-exec catch signal SIGSEGV` | C++ 异常、崩溃信号 |

条件断点表达式示例：

```
strcmp(state, "READY") == 0
packet->len > 1500
idx == 0 && retry > 3
```

注意：条件断点在 GDB 侧求值，命中频繁时会显著变慢；高频循环优先用 Logpoint 或"临时断点 + 计数"。

### 6.8 调试操作（常用键位）

| 操作 | 快捷键 | 说明 |
| --- | --- | --- |
| 启动/继续 | F5 | 运行到下一个断点 |
| 暂停 | F6 | 暂停正在运行的程序 |
| 单步跳过 | F10 | 不进入函数 |
| 单步进入 | F11 | 进入函数 |
| 单步跳出 | Shift+F11 | 回到调用者 |
| 重启 | Ctrl+Shift+F5 | 重新开始 |
| 切换断点 | F9 | 当前行 |
| 查看调用栈 | 左侧 CALL STACK | 点击帧切换上下文 |
| 查看线程 | 左侧 THREADS | 多线程切换 |
| 查看变量 | 左侧 VARIABLES | Locals / Watch |
| 求值 | Debug Console 输入表达式 | 支持函数调用（可能有副作用，慎用） |

Debug Console 常用 GDB 命令（加 `-exec` 前缀）：

```
-exec bt                 # 完整调用栈
-exec info threads       # 线程列表
-exec thread 3           # 切换线程
-exec frame 2            # 切换栈帧
-exec p/x value          # 十六进制打印
-exec p *ptr@10          # 打印指针指向的 10 个元素
-exec info locals        # 当前帧所有局部变量
```

### 6.9 查看内存

VSCode 本身没有完整的十六进制内存视图，推荐两种方式：

**方式一：GDB 控制台（推荐，最灵活）**

```
-exec x/32xb &buffer            # 32 字节，十六进制
-exec x/16wx ptr                # 16 个 4 字节字
-exec x/8gx ptr                 # 8 个 8 字节双字
-exec x/s str                   # 按字符串显示
-exec x/10i $pc                 # 当前指令附近反汇编
-exec p/x *(uint32_t*)0x7ffffff0  # 指定地址取值
-exec info registers            # 寄存器全览
-exec info proc mappings        # 进程内存映射（栈/堆/共享库范围）
```

格式符：`x` 十六进制、`d` 十进制、`u` 无符号、`s` 字符串、`i` 指令；单位：`b` 字节、`h` 半字、`w` 字、`g` 双字。

**方式二：Hex Editor 插件**

- 打开二进制或 core 文件，直接按地址查看；
- 适合对照 `info proc mappings` 定位的地址区间；
- 只读查看，修改请用 GDB 的 `set {type}addr = value`。

**查看指针与数组的小技巧**

```
-exec p *node                 # 结构体内容
-exec p node->next            # 链表下一节点
-exec p *node@5               # 以 node 为起点连续 5 个结构体
-exec p arr[0]@5              # 数组前 5 个元素
-exec set $p = (struct pkt*)0x555555   # 手工构造指针变量
```

**定位内存问题的组合拳**

| 问题 | 手段 |
| --- | --- |
| 越界写 | ASan（`-fsanitize=address`）优先；GDB watchpoint 兜底 |
| 释放后使用 | ASan；`-exec watch` 观察对象地址 |
| 内存泄漏 | valgrind `--leak-check=full`；ASan `detect_leaks=1` |
| 栈溢出/踩栈 | `-exec info frame`、`-exec x/64gx $rsp` |
| 野指针 | `-exec p ptr`、`info proc mappings` 判断地址是否合法 |

### 6.10 Core Dump 调试

远端配置：

```bash
ulimit -c unlimited
cat /proc/sys/kernel/core_pattern     # 确认 core 存放路径
```

若 `core_pattern` 指向 `systemd-coredump`，使用：

```bash
coredumpctl list
coredumpctl dump <pid> --output=./core/app.core
```

然后在 VSCode 用 "Debug: core dump" 配置打开，可直接查看崩溃时的调用栈、变量与内存。注意：**core 必须与产生它的二进制、共享库版本一致**，否则栈会错位。

### 6.11 远程与多进程调试

**Remote-SSH 方式**（推荐）

1. 安装 Remote - SSH，连接编译机；
2. 在远端工作区安装 `cpptools`（扩展需安装在远端）；
3. 直接使用 6.5 / 6.6 的配置，路径全部按远端解析。

**gdbserver 方式**（板卡、受限环境）

```bash
# 目标机
gdbserver :2345 ./app --config configs/dev.yaml
# 开发机：VSCode 使用 "Attach: gdbserver 远程" 配置
```

**多进程 / fork**

```
-exec set follow-fork-mode child     # 跟踪子进程
-exec set detach-on-fork on          # 其余进程放行
-exec set follow-fork-mode parent    # 默认，跟踪父进程
```

**多线程**：在 THREADS 面板切换线程；断点默认对所有线程生效，可在断点设置中选择 "Thread" 限定。

### 6.12 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 断点是空心圆，不生效 | 未带 `-g`、源码路径不匹配、二进制已更新 | 检查构建参数；配置 `sourceFileMap`；重新 `make` |
| 变量显示 `<optimized out>` | `-O2` 优化 | 用 debug 构建；或改用 `-Og` 定位 |
| F5 启动即退出 | 参数/工作目录错误 | 先在终端手动运行同一命令 |
| Attach 失败 `ptrace: Operation not permitted` | `ptrace_scope` 限制 | `sudo sysctl kernel.yama.ptrace_scope=0`，或使用同一用户启动 |
| 看不到 STL 容器内容 | 未启用 pretty-printer | 确认 `-enable-pretty-printing`；gdb 版本与 gcc 匹配 |
| IntelliSense 大量报红但能编译 | includePath 不全 | 用 `bear` 生成 `compile_commands.json` |
| 调试卡顿 | 条件断点过多、日志断点打印大对象 | 减少条件断点；打印摘要而非整个容器 |
| 程序在 CI 崩溃，本地正常 | 优化级别/未定义行为差异 | 用 release + ASan/UBSan 复现；检查未初始化变量 |
| 源码与二进制不一致 | 调试旧产物 | 清理 `out/` 后重建；`preLaunchTask` 保证先编译 |

---

## 7. 基于 Makefile 的构建

### 7.1 目录结构

```
repo/
├── Makefile
├── include/
│   └── app/
├── src/
│   ├── main.cpp
│   └── core/
├── tests/
├── configs/
├── scripts/
├── .clang-format
├── .editorconfig
├── .gitignore
├── .gitlab-ci.yml
├── .vscode/
│   ├── settings.json
│   ├── tasks.json
│   └── launch.json
└── README.md
```

构建产物统一输出到 `out/<build>/`，`out/` 不提交：

```
out/
├── debug/
│   ├── obj/
│   ├── obj-tests/
│   ├── bin/
│   └── reports/
└── release/
    ├── obj/
    ├── bin/
    └── dist/
```

### 7.2 设计原则

1. **单一入口**：本地、IDE、CI 都执行 `make`，命令一致；
2. **构建类型隔离**：`debug` / `release` 输出目录分离，互不污染；
3. **无绝对路径**：不出现个人目录、机器路径；
4. **显式产物**：不把 `.o`、可执行文件、报告提交到 Git；
5. **正确增量**：头文件依赖自动生成（`-MMD -MP`），改头文件会触发重编；
6. **可并行**：`make -j` 安全，目录创建用 order-only 依赖；
7. **CI 对齐**：`make BUILD=release package` 在本地能产出与流水线一致的包；
8. **可诊断**：提供 `help`、`version` 目标，出错信息可读。

### 7.3 目标清单

| 目标 | 说明 | CI 使用 |
| --- | --- | --- |
| `all`（默认） | 构建 app 与单元测试 | 是 |
| `debug` / `release` | 以指定类型重新构建 | 否（用 `BUILD=`） |
| `test` | 运行单元测试并输出 XML 报告 | 是 |
| `regression` | 长回归（仿真/板卡） | Nightly |
| `analyze` | 静态分析 | 是 |
| `format` | clang-format 就地格式化 | 否（本地） |
| `package` | 生成发布包 + 校验和 | Tag |
| `compdb` | 生成 `compile_commands.json` | 否（本地/IDE） |
| `clean` | 删除 `out/` | 是 |
| `help` | 列出目标 | 否 |

### 7.4 参考 Makefile

```make
# ============================================================
# 顶层 Makefile
# 用法:
#   make [BUILD=debug|release] [target]
#   make SAN="-fsanitize=address,undefined" test
# ============================================================

SHELL := /bin/bash

CXX   ?= g++
BUILD ?= debug
JOBS  ?= $(shell getconf _NPROCESSORS_ONLN 2>/dev/null || echo 4)

SRC_DIR    := src
INC_DIR    := include
TEST_DIR   := tests
OUT_DIR    := out/$(BUILD)
OBJ_DIR    := $(OUT_DIR)/obj
OBJ_TEST   := $(OUT_DIR)/obj-tests
BIN_DIR    := $(OUT_DIR)/bin
REP_DIR    := $(OUT_DIR)/reports
TARGET     := $(BIN_DIR)/app
TEST_BIN   := $(BIN_DIR)/unit_tests

VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo 0.0.0-dev)

SAN ?=

COMMON_FLAGS := -std=c++17 -I$(INC_DIR) -MMD -MP \
                -Wall -Wextra -Wpedantic -Wshadow \
                -DVERSION='"$(VERSION)"'

ifeq ($(BUILD),debug)
  OPT_FLAGS := -O0 -g3 -fno-omit-frame-pointer
  DEFINES   := -DDEBUG
else ifeq ($(BUILD),release)
  OPT_FLAGS := -O2 -g -ffile-prefix-map=$(CURDIR)=. -D_FORTIFY_SOURCE=2
  DEFINES   := -DNDEBUG
else
  $(error BUILD 只能是 debug 或 release，当前为 '$(BUILD)')
endif

CXXFLAGS += $(COMMON_FLAGS) $(OPT_FLAGS) $(DEFINES) $(SAN)
LDFLAGS  += $(SAN)
LDLIBS   += -pthread

APP_SRCS  := $(shell find $(SRC_DIR) -name '*.cpp')
APP_OBJS  := $(patsubst $(SRC_DIR)/%.cpp,$(OBJ_DIR)/%.o,$(APP_SRCS))
MAIN_OBJ  := $(OBJ_DIR)/main.o
LIB_OBJS  := $(filter-out $(MAIN_OBJ),$(APP_OBJS))

TEST_SRCS := $(shell find $(TEST_DIR) -name '*.cpp')
TEST_OBJS := $(patsubst $(TEST_DIR)/%.cpp,$(OBJ_TEST)/%.o,$(TEST_SRCS))

DEPS := $(APP_OBJS:.o=.d) $(TEST_OBJS:.o=.d)

.PHONY: all debug release test regression analyze format package compdb clean help

all: $(TARGET) $(TEST_BIN)

debug:
	@$(MAKE) BUILD=debug all

release:
	@$(MAKE) BUILD=release all

# ---------- 链接 ----------
$(TARGET): $(APP_OBJS) | $(BIN_DIR)
	$(CXX) $(LDFLAGS) $^ $(LDLIBS) -o $@

$(TEST_BIN): $(TEST_OBJS) $(LIB_OBJS) | $(BIN_DIR)
	$(CXX) $(LDFLAGS) $^ $(LDLIBS) -o $@

# ---------- 编译 ----------
$(OBJ_DIR)/%.o: $(SRC_DIR)/%.cpp
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -c $< -o $@

$(OBJ_TEST)/%.o: $(TEST_DIR)/%.cpp
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -I$(TEST_DIR) -c $< -o $@

# ---------- 目录 ----------
$(BIN_DIR):
	@mkdir -p $@

# ---------- 测试 ----------
test: $(TEST_BIN)
	@mkdir -p $(REP_DIR)
	$(TEST_BIN) --gtest_output=xml:$(REP_DIR)/unit.xml

regression: $(TARGET)
	@mkdir -p $(REP_DIR)
	./scripts/run_regression.sh $(TARGET) $(REP_DIR)

# ---------- 质量 ----------
analyze: compdb
	@mkdir -p $(REP_DIR)
	clang-tidy -p . $(APP_SRCS) 2>&1 | tee $(REP_DIR)/clang-tidy.txt

format:
	clang-format -i $$(find $(SRC_DIR) $(INC_DIR) $(TEST_DIR) \
	  \( -name '*.cpp' -o -name '*.h' -o -name '*.hpp' \))

compdb:
	bear -- $(MAKE) BUILD=$(BUILD) clean all

# ---------- 打包 ----------
package:
	@$(MAKE) BUILD=release all
	@mkdir -p out/release/dist
	tar -czf out/release/dist/app-$(VERSION)-linux-x86_64.tar.gz \
	  -C out/release/bin app
	cd out/release/dist && sha256sum *.tar.gz > SHA256SUMS

# ---------- 清理 ----------
clean:
	rm -rf out/

# ---------- 帮助 ----------
help:
	@echo "用法: make [BUILD=debug|release] [target]"
	@echo "  all         构建程序与单元测试（默认）"
	@echo "  debug       以 debug 类型构建"
	@echo "  release     以 release 类型构建"
	@echo "  test        运行单元测试"
	@echo "  regression  运行长回归"
	@echo "  analyze     静态分析（clang-tidy）"
	@echo "  format      clang-format 格式化"
	@echo "  package     生成发布包"
	@echo "  compdb      生成 compile_commands.json"
	@echo "  clean       清理 out/"

-include $(DEPS)
```

> 说明：`test` 目标假定单元测试使用 GoogleTest（`--gtest_output=xml:...`），以便 GitLab 解析 JUnit 报告。若使用其他框架（Catch2、自定义 runner），替换该行参数即可，报告格式保持 JUnit XML。
>
> `compdb` 会先 `clean` 再全量编译，这是 `bear` 捕获完整编译命令的必要代价，只建议在本地或 Nightly 使用，不要放进每次 MR。

### 7.5 依赖与增量构建

- `-MMD -MP` 生成 `.d` 文件，`-include $(DEPS)` 引入，保证头文件变更触发重编；
- 生成目录用 order-only 依赖（`| $(BIN_DIR)`）+ `mkdir -p`，并行安全；
- 对象文件路径由源码路径派生，避免同名文件冲突；
- **注意**：`$(shell find ...)` 在 Make 解析阶段求值，新增源文件不会自动出现在变量里，需重新执行 `make`（Make 每次启动都会重新求值，正常使用无需担心；只有长时间运行的 Make 进程需要注意）。
- NFS/网络文件系统上时间戳可能不可靠，增量构建异常时先 `make clean`；
- 大项目链接慢时，可拆分静态库、启用 `-fuse-ld=gold` 或 `lld`。

### 7.6 版本与可重现

- 版本号来自 `git describe`，通过 `-DVERSION` 注入，程序 `--version` 输出：版本、commit、构建类型、编译器版本；
- 发布包记录构建清单（编译器、依赖、构建时间、commit）；
- 可重现构建要求：不使用 `__DATE__`/`__TIME__`；路径用 `-ffile-prefix-map` 归一化；工具链固定到镜像 digest；打包时文件顺序与时间戳归一。

### 7.7 第三方依赖

- 版本锁定：在 `README` 与构建清单中写明版本（示例：GCC 10.2、Boost 1.70）；
- 头文件用 `-isystem` 引入，避免第三方告警污染；
- 动态库发布时设置 `rpath`（`-Wl,-rpath,'$$ORIGIN/../lib'`），避免现场 `LD_LIBRARY_PATH` 依赖；
- 大依赖优先使用系统包或统一缓存目录，不在仓库里放二进制；
- 依赖升级单独提交，带基准测试结果。

### 7.8 与 CI 的一致性

- CI 只执行 `make`，不额外写编译命令；
- 本地与 CI 使用同一镜像/同一编译器版本；
- CI 额外加 `CXXFLAGS="-Werror"` 作为门禁，本地默认不加，避免新编译器告警阻塞日常开发；
- 发布包由 CI 产出，本地 `make package` 的结果应可对照验证。

### 7.9 常见坑

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 改头文件不重编 | 缺少 `-MMD -MP` 或未 include `.d` | 补齐依赖生成 |
| 命令行传 `CXXFLAGS` 后优化参数丢失 | 命令行变量覆盖 Makefile 赋值 | 用独立变量（如 `SAN`）传参 |
| 并行编译随机失败 | 目录未创建、共享临时文件 | order-only 依赖 + `mkdir -p`；临时文件带 PID |
| `make` 报 `No rule to make target` | 源文件被删除但 `.d` 残留 | `make clean` |
| 本地能编 CI 不能 | 工具链版本不同、缺依赖 | 固定镜像；README 记录版本 |
| 递归 Make 并行失效 | 子 Make 之间依赖不明 | 改用单层 Makefile 或正确声明依赖 |
| recipe 中 `cd` 后路径错乱 | 每条 recipe 独立 shell | 用 `-C` 或绝对路径，避免链式 `cd` |
| 调试信息缺失 | release 去掉了 `-g` 或 strip | release 保留 `-g`，发布时再单独 strip |

---

## 8. 落地清单

按顺序执行，每步有明确验收标准：

| 步骤 | 内容 | 验收标准 |
| --- | --- | --- |
| 1 | GitLab 实例、HTTPS、SMTP、备份 | 能登录、能发测试邮件、备份可恢复 |
| 2 | 建立 Group / Subgroup 结构 | 命名符合第 2 章，权限正确 |
| 3 | 建立模板仓库 | 包含 README、`.gitignore`、`.gitlab-ci.yml`、Makefile、`.vscode/`、MR 模板、CODEOWNERS |
| 4 | 配置保护分支与审批规则 | 直推被拒，MR 必须 Pipeline 通过 |
| 5 | 注册 Runner 并打标签 | 各标签均有在线 Runner，Job 能调度 |
| 6 | 配置定时流水线 | Nightly 能手动触发并成功运行 |
| 7 | 配置通知 | 触发一次失败，验证收件人、格式、去重 |
| 8 | 试点 1–2 个仓库迁移 | 完整走通"分支 → MR → Pipeline → 合并 → 打包" |
| 9 | 两周复盘 | 收集问题，修订本文档至 v1.0 |

---

## 9. 待确认事项（初稿）

以下问题会实质影响规范细节，需在评审时确认：

1. **GitLab 部署形态**：CE 还是 EE？Docker 还是 Omnibus？运维责任人是谁？备份保留多久？
2. **分支模型**：是否确实需要 `release/x.y` 长期分支并行维护？若只有一个在研版本，可完全省略。
3. **审批强度**：核心模块是否要求 2 人批准？CODEOWNERS 归属如何划分？
4. **通知渠道**：是否引入企业微信/钉钉/飞书？邮件是否仍为主通道？谁来维护汇总脚本？
5. **回归资源**：回归耗时、仿真器/板卡数量、是否需要夜间独占？
6. **工具链版本**：是否锁定 GCC 10.2？有无升级计划？镜像如何分发？
7. **合规要求**：是否有客户审计条款影响日志留存、代码导出、制品分发？
8. **大文件策略**：是否存在测试向量、波形、固件等大文件？是否启用 Git LFS 及配额？
9. **仓库拆分粒度**：现有代码是否需要拆分为多仓库？拆分时机与迁移方式。

---

（初稿完，待评审后更新为 v1.0）
