# Java + AI Agent 开发学习项目

> 从 Web 底层到 AI Agent，系统化掌握 Java 智能体开发

## 📚 项目介绍

本项目是一套**体系化的 Java + AI Agent 开发学习路径**，以"学一章、练一章"的渐进方式，带你从 Web 基础知识出发，逐步深入到 LLM 集成、Function Calling、RAG、Multi-Agent 协作等 AI Agent 核心领域。

项目包含：
- **理论文档** — 分章节的 Markdown 知识体系，每个技术点都有详细的白话解释 + 完整代码
- **实操指导** — 每章配套的动手实践指南，从复现问题到完整项目实现
- **项目代码** — 可直接编译运行的 Java 示例项目
- **综合实战** — 跨章节的大型实战项目

## 🗺️ 学习路线总览

```
┌─────────── Java 核心基础 ──────────────┐
│ ① Web底层基础 → ② 多线程与并发编程        │
│      │                                   │
│      └──── ③ Servlet 核心体系 ──────┘     │
│      （理解 Spring MVC 的底层根基）        │
└────────────────┬───────────────────────┘
                 ▼
┌─────────── AI Agent 开发 ──────────────┐
│ ④ Agent入门 → ⑤ LLM基础 → ⑥ Function Calling
│                                   │
│         ⑧ LangChain4j ← ⑦ RAG+Memory
│              │
│         ⑨ Spring AI → ⑩ 高级模式
│                             │
│          综合实战（3个项目）──┘
└────────────────────────────────────────┘
```

## 🛠️ 技术栈

| 维度 | 技术 |
|------|------|
| 语言 | Java 17+ |
| 构建工具 | Maven / Gradle |
| Web 基础 | Servlet / Tomcat |
| 数据库 | MySQL |
| LLM API | OpenAI / Claude / 国产大模型（可配置） |
| AI 框架 | LangChain4j、Spring AI |
| 向量数据库 | ChromaDB / Pinecone / Milvus（可切换） |

> 具体技术选择将在各章节中明确说明。

## 📖 章节速览

| # | 章节 | 类型 | 状态 |
|---|------|------|------|
| 01 | [Web 底层基础](chapters/01-web-fundamentals/theory/) | Java 基础 | ✅ 已完成 |
| 02 | [多线程与并发编程](chapters/02-concurrency-programming/theory/) | Java 基础 | ✅ 已完成 |
| 03 | [Servlet 核心体系](chapters/03-servlet-core/theory/) | Java 基础 | ✅ 已完成 |
| 04 | AI Agent 入门 | AI Agent | ⏳ 待填充 |
| 05 | LLM 基础 | AI Agent | ⏳ 待填充 |
| 06 | Function Calling | AI Agent | ⏳ 待填充 |
| 07 | RAG 基础 + Memory 系统 | AI Agent | ⏳ 待填充 |
| 08 | LangChain4j | AI Agent | ⏳ 待填充 |
| 09 | Spring AI | AI Agent | ⏳ 待填充 |
| 10 | 高级模式 | AI Agent | ⏳ 待填充 |

## 🚀 如何使用

1. **按顺序学习** — 每章标注了前置依赖，建议按编号顺序进行
2. **先读理论，再做实践** — 每章 `theory/` 目录为理论基础，`practice/` 为实操指导
3. **动手写代码** — `code/` 目录包含随章节推进的 Java 项目，边学边练
4. **完成综合项目** — 学完所有章节后，挑战 `projects/` 下的完整实战

## 📖 学习建议

- 每天投入 2-3 小时，预计 7-9 周完成全部内容
- 遇到问题时先用 AI 工具辅助理解
- 每章实操完成后，建议用自己的话总结并输出笔记
- 积极参与综合项目，将所学串联起来

---

*开始你的第一个学习章节：👉 [01-Web底层基础](chapters/01-web-fundamentals/theory/)*
