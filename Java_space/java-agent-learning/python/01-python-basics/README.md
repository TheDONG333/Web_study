# 模块 1：Python 基础语法与工程环境

> **学习目标**：搭好 Python 工程环境，掌握 Agent 开发必需的基础语法（字符串、流程控制、异常、JSON），不碰工作用不到的内容。

---

## 学习顺序

| 顺序 | 文档 | 预计时间 | 核心内容 |
|------|------|---------|---------|
| 1 | [01.1 环境与工程基础](theory/01.1-environment-and-project-basics.md) | 2 小时 | Python 安装、pip、venv、调试、模块导入 |
| 2 | [01.2 变量与字符串](theory/01.2-variables-and-strings.md) | 2.5 小时 | 动态类型、基础类型、字符串切片/f-string/常用方法 |
| 3 | [01.3 运算符与流程控制](theory/01.3-operators-and-control-flow.md) | 2 小时 | 运算符、if/elif/else、for/while、break/continue |
| 4 | [01.4 异常处理](theory/01.4-exception-handling.md) | 1.5 小时 | try/except/else/finally、常见异常、raise、Agent 兜底 |
| 5 | [01.5 文件与 JSON](theory/01.5-files-and-json.md) | 2 小时 | with、文本读写、json.dumps/loads |

---

## 实操

| 实操 | 内容 |
|------|------|
| [实操 01.1 环境搭建 + 基础语法练习](practice/practice-01.1-environment-and-basic-syntax.md) | 建 venv、装包、跑第一个脚本、字符串/流程/异常小练习 |
| [实操 01.2 文件 + JSON：Agent 配置与对话记录](practice/practice-01.2-json-and-file-agent-demo.md) | 用 with 读写文件，用 JSON 存 Agent 配置和对话历史 |

---

## 建议安排（2~3 天）

```
第 1 天：01.1 + 实操 01.1 的环境部分
  - 装好 Python，跑通 venv 和第一个脚本

第 2 天：01.2 + 01.3 + 实操 01.1 的练习部分
  - 字符串和流程控制必须动手敲

第 3 天：01.4 + 01.5 + 实操 01.2
  - 异常处理和 JSON 是 Agent 项目的地基
```

---

## 自检清单

- [ ] 能独立创建并激活 venv，知道为什么不能把包装到全局
- [ ] 会用 `requirements.txt` 固化依赖
- [ ] 知道 `import` / `from ... import ...` / `as` 三种导入写法
- [ ] 能说出 Python 和 Java 在变量声明上的核心差异
- [ ] 字符串索引、切片、f-string、split/join/strip/replace/find 都能熟练用
- [ ] 会用 `for` 遍历列表、字典，会写 `if/elif/else` 和三元表达式
- [ ] 能完整写出 `try/except/else/finally`
- [ ] 知道 ValueError / KeyError / IndexError / TypeError 分别在什么场景出现
- [ ] 会用 `json.dumps` / `json.loads`，知道中文为什么要 `ensure_ascii=False`
- [ ] 能解释 `with open(...)` 相比手动 close 的好处

---

## 本模块不学

- Python 2.x 语法
- turtle 绘图、GUI 编程、系统底层 API

## 了解即可（不深挖）

- `input()` 控制台输入（Agent 基本不用）
- `%` 格式化和 `format()`（f-string 足够）
- 二进制文件、编码进阶
- 正则表达式（后续文本处理再补）

---

*返回：[Python 学习总览](../README.md)*
