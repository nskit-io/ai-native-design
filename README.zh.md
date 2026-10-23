[🇬🇧 English](README.md) · [🇰🇷 한국어](README.ko.md) · [🇯🇵 日本語](README.ja.md)

# AI-Native Design

**让 AI 能理解、修改并交付代码的结构化设计原则。**

> 如果我们不按人类的阅读习惯优化代码结构,而是按 AI 实际处理和修改代码的方式来优化,会怎样?

> 背景: 这些原则支撑着 [**NSKit**](https://github.com/nskit-io/nskit-io) — *有结构,才有无限组合*。一个生产级 Web 应用平台,把这些原则落成真正运行的服务。AI 视觉生成工具(Claude Design、v0、Lovable)位于另一层级 — 它们生成模型,而 NSKit 把 AI-Native Design 应用到 **真正运行的代码** 上。

---

## 为什么需要它

React、Vue、Angular 等现代框架是为 **Human Developer Experience** 精心设计的 — 模块化、类型安全、可复用、关注点分离。当人类是主要代码作者时,这些是正确的目标。

但当 AI 成为主要协作者 — 读代码、理解上下文、做修改、交付功能 — 优化目标就不同了。**AI-Native Design** 是从「什么样的结构能让 AI 工作得最准确、最高效?」出发的一组架构原则。

它不是一个框架或库,而是 **由生产系统基准实测支撑的设计哲学**。

---

## 核心对立

| 关注点 | Human DX 优化 | AI DX 优化 |
|---|---|---|
| **文件结构** | 带清晰 import 的小模块 | 一切内联的 Self-Contained 文件 |
| **依赖关系** | 抽象、复用、共享 | 最小化。每个 import 都是 AI 必须追踪的上下文 |
| **类型系统** | 丰富的类型用于安全和文档 | 约定优先于类型,模式优先于契约 |
| **文件大小** | 越小越好(SRP) | 500~800 行最佳(上下文窗口效率) |
| **构建流程** | Webpack/Vite/esbuild 用于优化 | 无。消除整类错误 |
| **状态管理** | 集中式存储(Redux、Pinia) | 文件作用域;无共享可变状态 |

两种方法无所谓「更好」,而是 **解决不同的约束**。当 AI 写了 80% 以上的代码时,最重要的约束是: **AI 能多准确地理解并修改这一个文件**。

---

## 六条原则

1. **Self-Contained Files** — HTML + CSS + JS 合成一个文件。不用追踪 import。
2. **500~800 行的最佳区间** — 针对上下文窗口优化。NSKit 框架实测:22 个 JS 文件、共 11,850 行、平均 539 行。
3. **No Build Process** — 从类别上消除打包与转译引发的错误。
4. **Convention over Configuration** — 可预测的模式比配置更帮助 AI。
5. **把 Token 效率当作设计目标** — AI 读的每一行都要消耗 token。
6. **Parallel AI Sessions** — 无共享可变状态。并行 AI 会话零冲突。

各原则的代码示例与论据见 [英文版 README](README.md)。

---

## Token 效率(估算)

修改卡片组件中一个点击处理函数时的估算:

| 对比 | 每次变更 token |
|---|---|
| React(模块化) | ~25,000(中位估算) |
| **AI-Native(Self-Contained)** | **~800** |

**约 31 倍效率**。估算方法与前提见 [benchmarks/token-comparison.md](benchmarks/token-comparison.md)。实际数值因项目而异。

---

## 与 NSKit 的关系

| 层 | 关系 |
|---|---|
| **AI-Native Design** | 设计原则(本仓库) |
| **NSKit** | 应用这些原则的生产框架 |
| **NSKit Template Mixing** | 因为原则导致的统一结构,模板才能无限组合 |

原则是通用的 — 你可以应用到自己的项目上。NSKit 是其参考实现之一。

---

## 详细内容与示例

完整技术内容、代码示例、FAQ、生产结果请查看英文版: **[README (English)](README.md)**

---

<div align="center">

**AI-Native Design** · Part of the **[NSKit](https://github.com/nskit-io/nskit-io)** ecosystem

© 2026 Neoulsoft Inc.

</div>
