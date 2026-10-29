# 实操 02.2：Agent 数据模型综合（嵌套字典 + 列表 + JSON）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：先自己写，卡住再让 AI 解释某一步，不要直接要完整答案

---

## 前置要求

- ✅ 已读完理论 02.1 ~ 02.4
- ✅ 已完成实操 [02.1](practice-02.1-list-dict-drills.md)
- ✅ 已完成模块 1 的 [JSON 实操](../../01-python-basics/practice/practice-01.2-json-and-file-agent-demo.md)
- ✅ 标准库即可，无需安装依赖

## 实践目标

用列表 + 字典搭出一个**迷你 Agent 状态对象**，完成：

1. 安全读取嵌套配置
2. 更新配置
3. 追加对话、过滤消息、统计角色
4. 管理工具配置
5. 用集合做标签匹配
6. 用元组接收函数多返回值
7. 保存成 JSON 并重新加载

> 这不是"玩具练习"：真实的 Agent 项目里，状态对象、配置、对话历史、工具参数就是这种结构。

---

## Step 1：定义状态对象

新建 `agent_state_demo.py`：

```python
import json

agent_state = {
    "session_id": "s001",
    "config": {
        "model": "gpt-4o-mini",
        "temperature": 0.7,
        "max_tokens": 1024
    },
    "user": {
        "id": 1001,
        "name": "Tom",
        "tags": ["vip", "new", "vip"]
    },
    "history": [
        {"role": "user", "content": "你好"},
        {"role": "assistant", "content": "你好，有什么可以帮你？"}
    ],
    "tools": {
        "search": {"enabled": True, "timeout": 30},
        "calculator": {"enabled": False, "timeout": 5}
    }
}
```

> 结构拆解：**字典套字典**（config/user/tools）、**列表套字典**（history）、**列表**（tags）。

---

## Step 2：安全读取配置

```python
print("=== 1. 安全读取配置 ===")

config = agent_state.get("config", {})
print("模型:", config.get("model", "unknown"))
print("温度:", config.get("temperature", 0.7))
print("top_p:", config.get("top_p", 1.0))
```

**预期**：

```
=== 1. 安全读取配置 ===
模型: gpt-4o-mini
温度: 0.7
top_p: 1.0
```

> 为什么用 get 链？如果某个字段缺失，程序不会崩，而是返回默认值。

---

## Step 3：更新配置

```python
print("\n=== 2. 更新配置 ===")

agent_state["config"]["temperature"] = 0.3
agent_state["config"].update({"top_p": 0.9, "max_tokens": 2048})

print(agent_state["config"])
```

**预期**：

```
=== 2. 更新配置 ===
{'model': 'gpt-4o-mini', 'temperature': 0.3, 'max_tokens': 2048, 'top_p': 0.9}
```

> `update` 一次改多个字段，已存在的覆盖，不存在的新增。

---

## Step 4：对话历史操作

```python
print("\n=== 3. 对话历史 ===")

agent_state["history"].append({"role": "user", "content": "介绍一下 Python"})

user_messages = [
    msg["content"]
    for msg in agent_state["history"]
    if msg["role"] == "user"
]
print("用户消息:", user_messages)

role_counts = {}
for msg in agent_state["history"]:
    role = msg["role"]
    role_counts[role] = role_counts.get(role, 0) + 1
print("角色统计:", role_counts)
```

**预期**：

```
=== 3. 对话历史 ===
用户消息: ['你好', '介绍一下 Python']
角色统计: {'user': 2, 'assistant': 1}
```

> 这里同时用到了：列表 append、列表推导式、字典计数、get 默认值。

---

## Step 5：工具配置

```python
print("\n=== 4. 工具配置 ===")

enabled_tools = {
    name: cfg
    for name, cfg in agent_state["tools"].items()
    if cfg.get("enabled", False)
}
print("启用的工具:", enabled_tools)

agent_state["tools"]["weather"] = {"enabled": True, "timeout": 10}
print("天气工具超时:", agent_state["tools"]["weather"]["timeout"])
```

**预期**：

```
=== 4. 工具配置 ===
启用的工具: {'search': {'enabled': True, 'timeout': 30}}
天气工具超时: 10
```

> 字典推导式筛选出启用的工具；嵌套赋值新增一个天气工具。

---

## Step 6：集合做标签匹配

```python
print("\n=== 5. 标签集合 ===")

user_tags = set(agent_state["user"]["tags"])
product_tags = {"vip", "electronics", "sale"}

print("用户标签:", user_tags)
print("共同标签:", user_tags & product_tags)
print("全部标签:", user_tags | product_tags)
print("用户独有:", user_tags - product_tags)
```

**预期**（集合顺序可能不同）：

```
=== 5. 标签集合 ===
用户标签: {'vip', 'new'}
共同标签: {'vip'}
全部标签: {'vip', 'new', 'electronics', 'sale'}
用户独有: {'new'}
```

> `tags` 里 vip 写了两次，转成 set 后自动去重。

---

## Step 7：元组接收多返回值

```python
print("\n=== 6. 元组返回值 ===")

def execute_tool(name, success=True):
    if success:
        return True, f"{name} 执行成功"
    return False, f"{name} 执行失败"

ok, message = execute_tool("search")
print(ok, message)

ok2, message2 = execute_tool("calculator", success=False)
print(ok2, message2)
```

**预期**：

```
=== 6. 元组返回值 ===
True search 执行成功
False calculator 执行失败
```

> 函数返回的 `True, "..."` 本质是一个元组，接收时直接解包。

---

## Step 8：保存为 JSON 并重新加载

```python
print("\n=== 7. 保存为 JSON ===")

with open("agent_state.json", "w", encoding="utf-8") as f:
    json.dump(agent_state, f, ensure_ascii=False, indent=2)

with open("agent_state.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print("会话:", loaded["session_id"])
print("消息数:", len(loaded["history"]))
print("模型:", loaded["config"]["model"])
print("已保存到 agent_state.json")
```

**预期**：

```
=== 7. 保存为 JSON ===
会话: s001
消息数: 3
模型: gpt-4o-mini
已保存到 agent_state.json
```

> 打开 `agent_state.json` 看一眼：中文可读、缩进清晰、结构和你定义的完全一致。

---

## 完整脚本

<details>
<summary>点开查看完整代码</summary>

```python
import json

agent_state = {
    "session_id": "s001",
    "config": {
        "model": "gpt-4o-mini",
        "temperature": 0.7,
        "max_tokens": 1024
    },
    "user": {
        "id": 1001,
        "name": "Tom",
        "tags": ["vip", "new", "vip"]
    },
    "history": [
        {"role": "user", "content": "你好"},
        {"role": "assistant", "content": "你好，有什么可以帮你？"}
    ],
    "tools": {
        "search": {"enabled": True, "timeout": 30},
        "calculator": {"enabled": False, "timeout": 5}
    }
}

print("=== 1. 安全读取配置 ===")
config = agent_state.get("config", {})
print("模型:", config.get("model", "unknown"))
print("温度:", config.get("temperature", 0.7))
print("top_p:", config.get("top_p", 1.0))

print("\n=== 2. 更新配置 ===")
agent_state["config"]["temperature"] = 0.3
agent_state["config"].update({"top_p": 0.9, "max_tokens": 2048})
print(agent_state["config"])

print("\n=== 3. 对话历史 ===")
agent_state["history"].append({"role": "user", "content": "介绍一下 Python"})
user_messages = [
    msg["content"]
    for msg in agent_state["history"]
    if msg["role"] == "user"
]
print("用户消息:", user_messages)

role_counts = {}
for msg in agent_state["history"]:
    role = msg["role"]
    role_counts[role] = role_counts.get(role, 0) + 1
print("角色统计:", role_counts)

print("\n=== 4. 工具配置 ===")
enabled_tools = {
    name: cfg
    for name, cfg in agent_state["tools"].items()
    if cfg.get("enabled", False)
}
print("启用的工具:", enabled_tools)
agent_state["tools"]["weather"] = {"enabled": True, "timeout": 10}
print("天气工具超时:", agent_state["tools"]["weather"]["timeout"])

print("\n=== 5. 标签集合 ===")
user_tags = set(agent_state["user"]["tags"])
product_tags = {"vip", "electronics", "sale"}
print("用户标签:", user_tags)
print("共同标签:", user_tags & product_tags)
print("全部标签:", user_tags | product_tags)
print("用户独有:", user_tags - product_tags)

print("\n=== 6. 元组返回值 ===")
def execute_tool(name, success=True):
    if success:
        return True, f"{name} 执行成功"
    return False, f"{name} 执行失败"

ok, message = execute_tool("search")
print(ok, message)
ok2, message2 = execute_tool("calculator", success=False)
print(ok2, message2)

print("\n=== 7. 保存为 JSON ===")
with open("agent_state.json", "w", encoding="utf-8") as f:
    json.dump(agent_state, f, ensure_ascii=False, indent=2)

with open("agent_state.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print("会话:", loaded["session_id"])
print("消息数:", len(loaded["history"]))
print("模型:", loaded["config"]["model"])
print("已保存到 agent_state.json")
```

</details>

---

## 验证标准

- [ ] 嵌套配置能安全读取，缺失字段返回默认值
- [ ] `update` 后配置包含 top_p，max_tokens 被更新
- [ ] 历史追加后共 3 条，用户消息过滤正确
- [ ] 角色统计结果为 `{'user': 2, 'assistant': 1}`
- [ ] 启用的工具只有 search
- [ ] 天气工具新增成功，超时可读取
- [ ] 标签去重、交集、并集、差集全部正确
- [ ] 元组返回值能正确解包
- [ ] `agent_state.json` 生成且中文可读
- [ ] 重新加载后会话、消息数、模型都正确
- [ ] 能说出这份数据里哪些是 list、哪些是 dict、哪些是 set/tuple

---

## 练习任务

1. **加系统消息**：在 history 最前面插入 `{"role": "system", "content": "你是一个简洁的助手"}`，重新统计角色数量。
2. **合并用户配置**：定义 `user_config = {"temperature": 0.1, "top_p": 0.8}`，用 `update` 合并到 `agent_state["config"]`。
3. **第二个用户**：定义 `user_b_tags = {"vip", "electronics"}`，求两个用户的共同标签。
4. **只保存配置**：把 `agent_state["config"]` 单独保存成 `config.json`。
5. **（了解）用 Counter**：用 `collections.Counter` 重写角色统计，对比代码量。
6. **（选做）异常兜底**：读取不存在的 `memory` 配置时，用 try/except KeyError 返回默认字典。

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `KeyError: 'xxx'` | 用了 `d[key]` 取不存在的键 | 改用 `d.get(key, 默认值)` |
| `TypeError: Object of type set is not JSON serializable` | 把 set 直接放进了要 dump 的结构 | 保存前转成 `list(...)` |
| JSON 里中文是转义字符 | 没加 `ensure_ascii=False` | 加参数 |
| 文件生成在奇怪的位置 | 运行目录不对 | 先 `cd` 到脚本目录再运行 |
| `json.decoder.JSONDecodeError` | 手动改坏了 JSON | 检查双引号、逗号，重新生成 |
| 集合输出顺序每次不同 | set 无序 | 正常现象，比较内容不要比较顺序 |

---

*上一节实操：[02.1 列表与字典专项练习](practice-02.1-list-dict-drills.md) | 下一模块：[03.1 函数基础与作用域](../../03-functions/theory/03.1-function-basics.md)*
