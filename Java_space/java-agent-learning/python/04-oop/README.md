# 模块 4：面向对象 OOP

> **学习目标**：会定义类和对象，掌握封装、继承、多态，能写出"基类 Agent + 不同业务子类"的结构，并让 Agent 实例可以像函数一样调用。

---

## 学习顺序

| 顺序 | 文档 | 预计时间 | 核心内容 |
|------|------|---------|---------|
| 1 | [04.1 类与对象基础](theory/04.1-class-basics.md) | 2 小时 | class、`__init__`、self、实例属性/方法、实例化 |
| 2 | [04.2 封装](theory/04.2-encapsulation.md) | 2.5 小时 | 公有/私有、`_` 与 `__`、@property get/set |
| 3 | [04.3 继承与多态](theory/04.3-inheritance-and-polymorphism.md) | 2.5 小时 | 单继承、super()、方法重写、鸭子类型、Agent 基类 |
| 4 | [04.4 魔术方法与类方法](theory/04.4-magic-methods-and-class-methods.md) | 2 小时 | `__str__`/`__repr__`、`__call__`、@classmethod、@staticmethod |

---

## 实操

| 实操 | 内容 |
|------|------|
| [实操 04.1 Agent 类与封装](practice/practice-04.1-agent-class.md) | 定义 Agent 类，用私有属性 + @property 保护对话历史和配置 |
| [实操 04.2 Agent 继承体系](practice/practice-04.2-agent-inheritance.md) | 基类 Agent + 对话/工具/数据三个子类，多态调用，`agent("你好")` |

---

## 建议安排（2~3 天）

```
第 1 天：04.1 + 04.2 + 实操 04.1
  - 类和对象、self、封装、@property 必须亲手写

第 2 天：04.3 + 04.4
  - 基类与子类、super()、方法重写
  - 魔术方法和类方法快速过

第 3 天：实操 04.2
  - 搭一个完整的 Agent 继承体系，体会多态
```

---

## 自检清单

- [ ] 会定义类、写 `__init__`、创建对象
- [ ] 理解 self 就是"当前对象"
- [ ] 能分清实例属性和类属性
- [ ] 知道 `_name` 是约定、`__name` 会改名，都不是真正的私有
- [ ] 会用 @property 写 getter，用 `@x.setter` 写 setter
- [ ] 能说出封装对 Agent 状态管理的意义
- [ ] 会写单继承，会用 super() 调用父类
- [ ] 会重写父类方法
- [ ] 理解多态和鸭子类型
- [ ] 会写 `__str__` / `__repr__`
- [ ] 会写 `__call__`，让 `agent("你好")` 可用
- [ ] 能分清 @classmethod 和 @staticmethod

---

## 了解即可（不深挖）

- 多继承、MRO 方法解析顺序
- 元类、反射、魔术方法大全
- 全套设计模式（写多了自然会接触）

## 完全不用学

- 过度复杂的元编程
- 猴子补丁
- 纯理论性的设计模式堆砌

---

*返回：[Python 学习总览](../README.md)*
