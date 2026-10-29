# 实操 04.1：Agent 类与封装

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 检查你的私有属性和 @property 是否有遗漏的保护

---

## 前置要求

- ✅ 已读完理论 04.1 ~ 04.2
- ✅ 会运行 Python 脚本
- ✅ 标准库即可

## 实践目标

写一个 `Agent` 类，把内部状态封装起来：

- 公有：`name`
- 内部：`_model`、`_history`
- 私有：`__api_key`
- 对外：`chat()`、只读的 `history`、可校验的 `model`、计算属性 `message_count`

---

## Step 1：定义基础类

新建 `agent_class.py`：

```python
class Agent:
    def __init__(self, name, model="gpt-4o-mini"):
        self.name = name                 # 公有
        self._model = model              # 内部
        self._history = []               # 内部
        self.__api_key = "sk-demo"       # 私有（会改名）

    def _add_message(self, role, content):   # 内部方法
        self._history.append({"role": role, "content": content})

    def chat(self, user_input):              # 对外方法
        self._add_message("user", user_input)
        reply = f"[{self._model}] 收到：{user_input}"
        self._add_message("assistant", reply)
        return reply
```

测试：

```python
agent = Agent("小助手")
print(agent.chat("你好"))
print(agent._history)        # 能访问，但约定不要碰
```

**预期**：

```
[gpt-4o-mini] 收到：你好
[{'role': 'user', 'content': '你好'}, {'role': 'assistant', 'content': '[gpt-4o-mini] 收到：你好'}]
```

---

## Step 2：加 @property（只读 history）

在类里加：

```python
    @property
    def history(self):
        return list(self._history)      # 返回副本
```

测试：

```python
print("历史消息数:", len(agent.history))

agent.history.append({"role": "user", "content": "外部伪造"})
print("外部 append 后内部消息数:", len(agent.history))   # 仍然是 2
```

**预期**：

```
历史消息数: 2
外部 append 后内部消息数: 2
```

> 因为 `history` 返回的是副本，外部改副本不影响内部。

再试：

```python
try:
    agent.history = []
except AttributeError as e:
    print("无法直接赋值:", e)
```

**预期**：输出类似 `无法直接赋值: property 'history' of 'Agent' object has no setter`。

---

## Step 3：加 model 的 getter / setter

在类里加：

```python
    @property
    def model(self):
        return self._model

    @model.setter
    def model(self, value):
        if not value:
            raise ValueError("模型名不能为空")
        self._model = value
```

测试：

```python
print("模型:", agent.model)

agent.model = "gpt-4o"
print("修改后:", agent.model)

try:
    agent.model = ""
except ValueError as e:
    print("拦截空模型:", e)
```

**预期**：

```
模型: gpt-4o-mini
修改后: gpt-4o
拦截空模型: 模型名不能为空
```

---

## Step 4：私有属性实验

```python
try:
    print(agent.__api_key)
except AttributeError:
    print("直接访问 __api_key: AttributeError")

print("改名访问:", agent._Agent__api_key)
```

**预期**：

```
直接访问 __api_key: AttributeError
改名访问: sk-demo
```

> 双下划线不是真正的安全，只是改名。**不要把密钥硬编码在类里**，真实项目用环境变量/配置中心。

---

## Step 5：加一个计算属性

在类里加：

```python
    @property
    def message_count(self):
        return len(self._history)
```

测试：

```python
print("消息条数:", agent.message_count)
```

**预期**：`消息条数: 2`

---

## Step 6：加 `__str__`

```python
    def __str__(self):
        return f"Agent({self.name})"
```

测试：

```python
print(agent)      # Agent(小助手)
```

---

## 完整代码

<details>
<summary>点开查看完整脚本</summary>

```python
class Agent:
    def __init__(self, name, model="gpt-4o-mini"):
        self.name = name
        self._model = model
        self._history = []
        self.__api_key = "sk-demo"

    @property
    def history(self):
        return list(self._history)

    @property
    def model(self):
        return self._model

    @model.setter
    def model(self, value):
        if not value:
            raise ValueError("模型名不能为空")
        self._model = value

    @property
    def message_count(self):
        return len(self._history)

    def _add_message(self, role, content):
        self._history.append({"role": role, "content": content})

    def chat(self, user_input):
        self._add_message("user", user_input)
        reply = f"[{self._model}] 收到：{user_input}"
        self._add_message("assistant", reply)
        return reply

    def __str__(self):
        return f"Agent({self.name})"


agent = Agent("小助手")
print(agent.chat("你好"))
print("历史消息数:", len(agent.history))

agent.history.append({"role": "user", "content": "外部伪造"})
print("外部 append 后内部消息数:", len(agent.history))

try:
    agent.history = []
except AttributeError as e:
    print("无法直接赋值:", e)

print("模型:", agent.model)
agent.model = "gpt-4o"
print("修改后:", agent.model)

try:
    agent.model = ""
except ValueError as e:
    print("拦截空模型:", e)

try:
    print(agent.__api_key)
except AttributeError:
    print("直接访问 __api_key: AttributeError")

print("改名访问:", agent._Agent__api_key)
print("消息条数:", agent.message_count)
print(agent)
```

</details>

---

## 验证标准

- [ ] `chat()` 能返回回复并记录两条历史
- [ ] `history` 外部 append 不影响内部
- [ ] `agent.history = []` 报 AttributeError（没有 setter）
- [ ] `model` 能正常读写
- [ ] 空模型名被 ValueError 拦截
- [ ] `__api_key` 直接访问失败，改名后能访问
- [ ] `message_count` 返回 2
- [ ] `print(agent)` 输出 `Agent(小助手)`
- [ ] 能说出 `_` 和 `__` 的区别
- [ ] 能说出返回 `list(self._history)` 的原因

---

## 练习任务

1. **加 `clear_history()` 方法**：只能通过方法清空历史，外部不能直接赋值。
2. **加 `set_api_key()`**：校验长度至少 8 位，否则抛 ValueError。
3. **加只读属性 `api_key_masked`**：返回 `sk-****` 这种脱敏形式，不暴露真实 Key。
4. **加类属性** `DEFAULT_MODEL = "gpt-4o-mini"`，在 `__init__` 里使用它。
5. **思考**：为什么 `history` 返回副本，而不是直接返回 `self._history`？

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `AttributeError: can't set attribute` | @property 没写 setter | 加 `@model.setter` |
| setter 没生效 | setter 名字前面的属性名写错 | `@model.setter` 必须和 getter 同名 |
| `__api_key` 访问不到 | 双下划线改名 | 正常现象；不要依赖外部访问 |
| `history` 被外部改了 | 直接返回了 `self._history` | 改成 `return list(self._history)` |
| 调用 property 时加了括号 | `agent.model()` | @property 按属性访问，不加括号 |

---

*上一节理论：[04.4 魔术方法与类方法](../theory/04.4-magic-methods-and-class-methods.md) | 下一步：[实操 04.2 Agent 继承体系](practice-04.2-agent-inheritance.md)*
