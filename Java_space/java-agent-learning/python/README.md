# Python 学习（AI Agent 方向）

> 本目录独立于 `chapters/`，专门存放 Python 相关文档。后续所有 Python 内容都放在这里。

---

## 技术选型

| 项 | 选择 | 说明 |
|----|------|------|
| Python | **3.11 / 3.12**（最低 3.10） | 新版语法和性能更适合 AI 开发 |
| 包管理 | pip + **venv** | 每个项目独立虚拟环境，依赖隔离 |
| 编辑器 | VS Code（推荐）或 PyCharm | 都要会打断点调试 |
| 依赖清单 | `requirements.txt` | 项目依赖固化，方便复现 |

---

## 目录约定

```
python/
├── README.md
└── 01-python-basics/          ← 每个模块一个文件夹
    ├── README.md              ← 模块导读、学习安排、自检
    ├── theory/                ← 理论文档
    └── practice/              ← 实操文档
```

> 后续模块（如 LangChain、RAG、Agent 开发）按学习大纲逐步添加，不提前占位。

---

## 模块索引

| 模块 | 主题 | 文档 |
|------|------|------|
| 模块 1 | Python 基础语法与工程环境 | [01-python-basics/README.md](01-python-basics/README.md) |
| 模块 2 | Python 容器（列表/字典/元组/集合） | [02-containers/README.md](02-containers/README.md) |
| 模块 3 | 函数与装饰器 | [03-functions/README.md](03-functions/README.md) |
| 模块 4 | 面向对象 OOP | [04-oop/README.md](04-oop/README.md) |
| 模块 5 | FastAPI 接口开发 | [05-fastapi/README.md](05-fastapi/README.md) |
| 模块 6 | Pydantic v2 与结构化接口 | [06-pydantic/README.md](06-pydantic/README.md) |

---

## 明确不学

| 内容 | 原因 |
|------|------|
| Python 2.x 语法 | 已淘汰 |
| turtle 绘图 | 教学玩具，工作用不到 |
| GUI 编程（tkinter/PyQt） | Agent 开发不涉及 |
| 系统底层 API | 入门阶段不需要 |

> 另有一些"了解即可"的内容（`input()`、`%` 格式化、`format()`、二进制文件、正则），在模块 1 文档里只做一句话说明，不展开。

---

## 学习原则

1. **环境先跑通**：venv + pip 是后面所有 Agent 项目的基础，第一天就要会。
2. **字符串是重点**：Agent 全程在处理文本、拼接 Prompt，字符串操作必须熟。
3. **异常处理是工程化底线**：调用大模型 API 一定会失败，必须会兜底。
4. **JSON 是数据交换格式**：大模型请求/响应、配置、对话历史，全是 JSON。
