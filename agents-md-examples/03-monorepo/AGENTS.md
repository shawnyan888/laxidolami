# AGENTS.md

<!-- 示例：Monorepo 根目录版本。
     根文件只写"全仓库通用"的内容，子项目细节下沉到各自的 AGENTS.md。 -->

pnpm + Turborepo Monorepo。Node 20，TypeScript 严格模式。

## 命令

```bash
# 安装依赖
pnpm install

# 构建全部包（按依赖拓扑序）
pnpm build

# 只构建某个包（在包目录内）
pnpm --filter @acme/web build

# 全部测试
pnpm test
```

## 全仓库约定

- TypeScript 严格模式，禁止 `any`；确需逃生舱时用 `unknown` + 类型收窄。
- 公共代码放 `packages/shared`，业务包通过 workspace 协议（`"@acme/shared": "workspace:*"`）引用，禁止相对路径跨包 import（如 `../../packages/shared/src/...`）。
- 每个包自带 `README.md`，说明包的职责和导出内容。
- commit message 遵循 Conventional Commits，scope 填包名（如 `feat(web): ...`）。

## 子项目

各子目录有独立约定，改动前先读对应目录下的 `AGENTS.md`：

- `packages/web/` — 前端应用（React）
- `packages/api/` — 后端服务（NestJS）
- `packages/shared/` — 共享库

<!--
"子项目"段是分层写法的关键：一句话告诉 AI
"处理哪个目录的文件，就去读哪个目录的 AGENTS.md"。
根文件保持短，只放跨包共享的规则——构建命令、依赖方向、commit 规范。
-->
