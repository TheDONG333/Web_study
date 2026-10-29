# 实操 04.2：Agent 继承体系（基类 + 三个子类 + 多态）

> ⏱ **预计时长**：75 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：让 AI 扮演代码审查员，检查你的继承结构有没有重复代码

---

## 前置要求

- ✅ 已读完理论 04.3 ~ 04.4
- ✅ 已完成实操 [04.1](practice-04.1-agent-class.md)
- ✅ 标准库即可

## 实践目标

搭一个 Agent 继承体系：

```
BaseAgent（基类：名字、模型、历史、chat 流程、__call__）
├── ChatAgent（对话）
├── ToolAgent（工具调用）
└── DataAgent（数据分析）
```

要求：

1. 基类负责公共流程，子类只实现 `respond()`
2. 子类用 `super().__init__()` 初始化父类
3. 用列表 + 循环体现多态
4. 实现 `__str__` / `__repr__` / `__call__`
5. 用 @classmethod 做工厂、@staticmethod 做角色校验

---

## Step 1：定义基类 BaseAgent

新建 `agent_inheritance.py`：

```python
class BaseAgent:
    VALID_ROLES = {"system", "user", "assistant"}

    def __init__(self, name, model="gpt-4o-mini"):
        self.name = name
        self.model = model
        self._history = []

    @property
    def history(self):
        return list(self._history)

    def _add_message(self, role, content):
        if not self.is_valid_role(role):
            raise ValueError(f"非法角色: {role}")
        self._history.append({"role": role, "content": content})

    def respond(self, user_input):
        raise NotImplementedError("子类必须实现 respond")

    def chat(self, user_input):
        self._add_message("user", user_input)
        reply = self.respond(user_input)      # 调用子类实现
        self._add_message("assistant", reply)
        return reply

    def __call__(self, user_input):
        return self.chat(user_input)

    def __str__(self):
        return f"Agent({self.name})"

    def __repr__(self):
        return f"{type(self).__name__}(name={self.name!r}, model={self.model!r})"

    @staticmethod
    def is_valid_role(role):
        return role in BaseAgent.VALID_ROLES
```

> 注意：`chat()` 是**模板方法**——流程固定（记用户消息 → 调 respond → 记回复），子类只改 `respond()` 的细节。

---

## Step 2：定义三个子类

```python
class ChatAgent(BaseAgent):
    def respond(self, user_input):
        return f"[对话] {user_input}"


class ToolAgent(BaseAgent):
    def __init__(self, name, tools, model="gpt-4o-mini"):
        super().__init__(name, model)      # 先初始化父类
        self.tools = tools                 # 再加自己的属性

    def respond(self, user_input):
        return f"[工具] 可用工具 {self.tools}，输入：{user_input}"


class DataAgent(BaseAgent):
    def respond(self, user_input):
        return f"[数据] 正在分析：{user_input}"
```

> `ChatAgent` / `DataAgent` 没有自己的 `__init__`，直接继承父类的；`ToolAgent` 多了 `tools`，所以要写 `__init__` 并调用 `super()`。

---

## Step 3：多态调用

```python
print("=== 1. 多态调用 ===")

agents = [
    ChatAgent("对话助手"),
    ToolAgent("工具助手", ["search", "calculator"]),
    DataAgent("数据助手")
]

for agent in agents:
    print(agent("你好"))
```

**预期**：

```
=== 1. 多态调用 ===
[对话] 你好
[工具] 可用工具 ['search', 'calculator']，输入：你好
[数据] 正在分析：你好
```

> 循环里只写了 `agent("你好")`，没有 if-else——具体行为由对象类型决定，这就是多态。

---

## Step 4：对象打印

```python
print("\n=== 2. 对象打印 ===")

print(agents[0])           # __str__
print(repr(agents[0]))     # __repr__
print(agents)              # 列表里的元素用 __repr__
```

**预期**：

```
=== 2. 对象打印 ===
Agent(对话助手)
ChatAgent(name='对话助手', model='gpt-4o-mini')
[ChatAgent(name='对话助手', model='gpt-4o-mini'), ToolAgent(name='工具助手', model='gpt-4o-mini'), DataAgent(name='数据助手', model='gpt-4o-mini')]
```

> 列表打印时用的是每个对象的 `__repr__`，所以 `__repr__` 要写全关键字段。

---

## Step 5：工厂方法 @classmethod

给 `ChatAgent` 加：

```python
    @classmethod
    def from_config(cls, config):
        return cls(config["name"], config.get("model", "gpt-4o-mini"))
```

使用：

```python
print("\n=== 3. 工厂方法 ===")
agent = ChatAgent.from_config({"name": "小助手", "model": "gpt-4o"})
print("from_config:", agent.name, agent.model)
```

**预期**：

```
=== 3. 工厂方法 ===
from_config: 小助手 gpt-4o
```

> 类方法适合做"从配置创建对象"的入口，`cls(...)` 会自动创建对应子类的实例。

---

## Step 6：静态方法角色校验

```python
print("\n=== 4. 静态方法 ===")
print("user 合法:", BaseAgent.is_valid_role("user"))
print("boss 合法:", BaseAgent.is_valid_role("boss"))
print("历史条数:", len(agents[0].history))
```

**预期**：

```
=== 4. 静态方法 ===
user 合法: True
boss 合法: False
历史条数: 2
```

---

## Step 7：验证基类必须被重写

```python
print("\n=== 5. 父类方法必须重写 ===")
try:
    BaseAgent("基类").chat("你好")
except NotImplementedError as e:
    print("拦截:", e)
```

**预期**：

```
=== 5. 父类方法必须重写 ===
拦截: 子类必须实现 respond
```

> 基类里的 `respond` 抛 `NotImplementedError`，强制子类实现，避免"忘了写"。

---

## 完整代码

<details>
<summary>点开查看完整脚本</summary>

```python
class BaseAgent:
    VALID_ROLES = {"system", "user", "assistant"}

    def __init__(self, name, model="gpt-4o-mini"):
        self.name = name
        self.model = model
        self._history = []

    @property
    def history(self):
        return list(self._history)

    def _add_message(self, role, content):
        if not self.is_valid_role(role):
            raise ValueError(f"非法角色: {role}")
        self._history.append({"role": role, "content": content})

    def respond(self, user_input):
        raise NotImplementedError("子类必须实现 respond")

    def chat(self, user_input):
        self._add_message("user", user_input)
        reply = self.respond(user_input)
        self._add_message("assistant", reply)
        return reply

    def __call__(self, user_input):
        return self.chat(user_input)

    def __str__(self):
        return f"Agent({self.name})"

    def __repr__(self):
        return f"{type(self).__name__}(name={self.name!r}, model={self.model!r})"

    @staticmethod
    def is_valid_role(role):
        return role in BaseAgent.VALID_ROLES


class ChatAgent(BaseAgent):
    def respond(self, user_input):
        return f"[对话] {user_input}"

    @classmethod
    def from_config(cls, config):
        return cls(config["name"], config.get("model", "gpt-4o-mini"))


class ToolAgent(BaseAgent):
    def __init__(self, name, tools, model="gpt-4o-mini"):
        super().__init__(name, model)
        self.tools = tools

    def respond(self, user_input):
        return f"[工具] 可用工具 {self.tools}，输入：{user_input}"


class DataAgent(BaseAgent):
    def respond(self, user_input):
        return f"[数据] 正在分析：{user_input}"


print("=== 1. 多态调用 ===")
agents = [
    ChatAgent("对话助手"),
    ToolAgent("工具助手", ["search", "calculator"]),
    DataAgent("数据助手")
]
for agent in agents:
    print(agent("你好"))

print("\n=== 2. 对象打印 ===")
print(agents[0])
print(repr(agents[0]))
print(agents)

print("\n=== 3. 工厂方法 ===")
agent = ChatAgent.from_config({"name": "小助手", "model": "gpt-4o"})
print("from_config:", agent.name, agent.model)

print("\n=== 4. 静态方法 ===")
print("user 合法:", BaseAgent.is_valid_role("user"))
print("boss 合法:", BaseAgent.is_valid_role("boss"))
print("历史条数:", len(agents[0].history))

print("\n=== 5. 父类方法必须重写 ===")
try:
    BaseAgent("基类").chat("你好")
except NotImplementedError as e:
    print("拦截:", e)
```

</details>

---

## 验证标准

- [ ] BaseAgent 能正常初始化名字、模型、历史
- [ ] 三个子类都能调用 `chat()`，输出各不相同
- [ ] ToolAgent 的 `tools` 通过 `super().__init__` 后正常设置
- [ ] 多态循环里没有 if-else
- [ ] `print(agent)` 输出 `Agent(名字)`
- [ ] `repr(agent)` 输出 `类名(name=..., model=...)`
- [ ] `agent("你好")` 能直接调用
- [ ] `from_config` 能创建 ChatAgent
- [ ] `is_valid_role` 能正确判断
- [ ] BaseAgent 直接调用 chat 会抛 NotImplementedError
- [ ] 能说出"模板方法"的含义

---

## 练习任务

1. **新增 WeatherAgent**：继承 BaseAgent，`respond` 返回 `[天气] 查询 {user_input}`，加入多态列表。
2. **给 ToolAgent 加 `__repr__`**：把 `tools` 也显示出来。
3. **扩展 DataAgent**：加 `__init__(name, data_source)`，用 `super().__init__(name)`，在 respond 里带上数据源。
4. **给 BaseAgent 加 `message_count` 属性**：返回历史条数。
5. **思考**：
   - 为什么 `chat()` 写在基类，`respond()` 留给子类？
   - 如果子类忘了写 `respond`，会发生什么？
   - `__call__` 和 `chat` 是什么关系？

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 子类调用时报缺少 name/history | 子类写了 `__init__` 但没调 `super().__init__` | 第一行加 `super().__init__(...)` |
| `NotImplementedError` | 基类 respond 没被子类重写 | 子类实现 `respond` |
| 多态循环报错 | 某个子类方法名拼错 | 方法名必须是 `respond` |
| `repr` 显示内存地址 | 没定义 `__repr__` | 加 `__repr__` |
| `agent("你好")` 报 not callable | 没定义 `__call__` | 加 `__call__` |
| 类方法报缺少参数 | 第一个参数写成了 self | 改成 `cls` |

---

*上一节实操：[04.1 Agent 类与封装](practice-04.1-agent-class.md) | 下一模块：[05.1 FastAPI 环境与启动](../../05-fastapi/theory/05.1-quickstart.md)*
