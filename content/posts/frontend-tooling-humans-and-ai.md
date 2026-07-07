---
title: 前端工具链的「双模态」时代：为人类和 AI 同时设计
date: 2026-07-07
tags: [前端工程化, AI, 工具链, Vite, Rspack]
author: Hermes Blog
---

过去十年，前端工具链以「开发者体验」为单一 north star 演进。2026 年的今天，这个等式多了一个变量：AI Agent 也要顺畅地读写你的构建配置、类型定义和中间产物。工具链的选型标准，从「DX 最速」变成了「DX + AI-readiness 双模兼容」。

这不是遥远的未来叙事——Cursor、Claude Code 等编码 Agent 已经把项目上下文（包括 `vite.config.ts`、`tsconfig.json`、`webpack.config.js`）喂进模型。如果你的工具链输出是高度结构化的、确定性的，Agent 就能更好地协作；如果是一团黑盒魔法，Agent 就只能瞎猜、反复重试，甚至输出dangerous的补丁。

## 为什么「AI 友好」和「人友好」以前是矛盾的？

传统工具的优化方向普遍是「让人类少写、少想」：Webpack 的 loader/plugin 链、Vite 的隐式约定、PostCSS 的魔法配置。这些对经验丰富的工程师是加速器，但对 Agent 来说有几个致命问题：

1. **隐式约定过多**。Vite 的 `define: { 'import.meta.env.MODE': string }` 对人是文档，对 Agent 是未序列化的知识缺口。
2. **输出格式不确定**。打包产物可能带 source map、可能带 hash、可能 tree-shake，取决于命令行参数和缓存状态。Agent 难以在 deterministic 预期下做后续处理（如按模块解构产物做热更新策略分析）。
3. **上下文碎片化**。一个前端项目的「真相」散落在 `vite.config.ts`、`tsconfig.json`、`package.json`、`.env`、`index.html` 五六个位置，Agent 需要串起它们才能完成「加一个路由」的操作。

## 双模态工具链的核心标准

我们提出三个可量化的选型维度：

### 1. 配置的强类型与自描述性