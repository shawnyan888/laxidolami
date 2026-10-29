# AGENTS.md

<!-- 示例：Monorepo 子目录版本。只写前端特有的规则，
     通用的（TypeScript 规范、commit 规范）已在根目录，这里不重复。 -->

React 18 + Vite 前端应用。

## 命令

```bash
# 开发服务器
pnpm dev

# 构建（含类型检查）
pnpm build

# 组件测试（Vitest + Testing Library）
pnpm test
```

## 前端约定

- 组件放 `src/components/`，一个组件一个目录：`ComponentName/index.tsx` + `ComponentName.module.css`。
- 组件用函数式写法 + hooks，状态管理用 Zustand，不引入 Redux。
- 样式用 CSS Modules，全局样式只放 `src/styles/global.css`。
- 接口调用统一走 `src/api/client.ts`，组件内禁止直接 `fetch`。
- 用户可见文案放在 `src/i18n/zh.json`，组件内不写死中文字符串。
- 提交前跑 `pnpm build`，类型错误不进 commit。
