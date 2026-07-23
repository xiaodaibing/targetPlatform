---
kind: frontend_style
name: 前端样式体系：基于 Vben Admin 的内部框架约束
category: frontend_style
scope:
    - '**'
source_files:
    - CLAUDE.md
---

本仓库为后端主导的单体应用（Spring Cloud Netflix/Hulk + MyBatis-Plus），前端工程并未随代码一同提交，仅通过根级 CLAUDE.md 对一期前端风格作出约定。当前仓库中不存在任何 .css / .scss / tailwind.config.* / 主题或设计 token 文件，因此无法从源码层面分析具体样式实现，只能依据工作手册中的规范进行归纳。

### 1. 使用的系统/方法
- UI 框架：Vben Admin（Vue3 + Vite + TypeScript + Ant Design Vue）
- 包管理：pnpm monorepo + turbo
- 组件与交互：强制使用 Vben Admin 二次封装的请求、组件与 hooks，禁止直接使用原生 Ant Design Vue 组件或裸 fetch
- 状态/路由：Pinia + Vue Router，动态菜单与权限按框架约定配置

### 2. 关键文件
- CLAUDE.md 第 3.2 节「前端(Vben Admin 内部框架)」——唯一的前端样式与开发规范来源

### 3. 架构与约定
- 一期仅 PC 端；移动端沿用 DCR APP（React Native），本平台只出接口与交互设计
- 样式层遵循 Vben Admin 默认主题与 Ant Design Vue 设计规范，不自行引入第三方 UI/工具库
- 所有接口地址走环境变量，按钮级权限使用框架权限指令/组件，禁止绕过权限直接渲染敏感操作

### 4. 开发者应遵守的规则
- 一律使用 <script setup> + Composition API + TypeScript
- 请求统一用框架封装的 useRequest（基于 axios），禁止裸 fetch
- 组件/请求/权限全部复用框架二次封装，禁止引入与框架重复的 UI/工具库
- 本地命令：pnpm install / pnpm dev / pnpm build / pnpm lint(ESLint/Prettier/Stylelint) / pnpm type:check / pnpm commit（husky 自动校验）

由于前端工程未包含在仓库中，以上结论完全来自 CLAUDE.md 的显式约定，置信度中等。