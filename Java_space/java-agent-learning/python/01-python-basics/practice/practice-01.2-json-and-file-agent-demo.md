# 实操 01.2：文件 + JSON —— Agent 配置与对话记录

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：JSON 报错把原始字符串和报错一起发给 AI，让它指出格式问题

---

## 前置要求

- ✅ 已读完理论 01.5
- ✅ 已完成实操 [01.1](practice-01.1-environment-and-basic-syntax.md)，`python-demo` 项目和 venv 能正常用
- ✅ 不需要安装任何第三方库，只用标准库 `json`

## 实践目标

写一个 `agent_demo.py`，完成 Agent 项目里最常见的三件事：

1. 把 **Agent 配置** 写成 JSON 文件，再读回来
2. 把 **对话历史**（列表套字典）保存成 JSON，再加载并遍历
3. 用 `with` 追加写日志，并处理"文件不存在 / JSON 格式错误"两种异常

---

## 项目结构

```
python-demo/
├── .venv/
├── requirements.txt
├── agent_demo.py          ← 本次要写的脚本
├── agent_config.json      ← 脚本自动生成
├── chat_history.json      ← 脚本自动生成
└── agent.log              ← 脚本自动生成
```

> 在 `python-demo` 目录下新建 `agent_demo.py`，激活 venv 后运行。

---

## Step 1：写配置 + 读配置

```python
import json

config = {
    "model": "gpt-4o-mini",
    "temperature": 0.7,
    "max_tokens": 1024,
    "system_prompt": "你是一个简洁的中文助手"
}

# 写：对象 → JSON 文件
with open("agent_config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, ensure_ascii=False, indent=2)

print("配置已写入 agent_config.json")

# 读：JSON 文件 → 对象
with open("agent_config.json", "r", encoding="utf-8") as f:
    loaded_config = json.load(f)

print("当前模型:", loaded_config["model"])
print("温度:", loaded_config["temperature"])
```

**运行后打开 `agent_config.json` 看一眼**，应该是可读的中文 JSON（不是 `\uXXXX`）。

> 关键参数：`ensure_ascii=False` 让中文正常显示，`indent=2` 让它换行缩进。

---

## Step 2：保存并加载对话历史

继续在 `agent_demo.py` 里追加：

```python
history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"}
]

# 模拟新增一条用户消息
history.append({"role": "user", "content": "介绍一下 Python"})

with open("chat_history.json", "w", encoding="utf-8") as f:
    json.dump(history, f, ensure_ascii=False, indent=2)

print(f"\n对话已保存，共 {len(history)} 条")

# 重新加载，模拟"下次启动时恢复对话"
with open("chat_history.json", "r", encoding="utf-8") as f:
    loaded_history = json.load(f)

print("--- 对话记录 ---")
for msg in loaded_history:
    print(f"{msg['role']}: {msg['content']}")
```

**预期输出**：

```
对话已保存，共 3 条
--- 对话记录 ---
user: 你好
assistant: 你好，有什么可以帮你？
user: 介绍一下 Python
```

---

## Step 3：追加写日志 + 读取日志

```python
# "a" 追加模式：不会清空原内容
with open("agent.log", "a", encoding="utf-8") as f:
    f.write("INFO 加载配置成功\n")
    f.write("INFO 保存对话成功\n")

print("\n--- 日志 ---")
with open("agent.log", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())
```

**预期输出**：

```
--- 日志 ---
INFO 加载配置成功
INFO 保存对话成功
```

> 运行两次，日志会变成 4 行——这就是追加模式的效果。想清空就手动删掉 `agent.log`，或改成 `"w"`。

---

## Step 4：异常处理（工程化必备）

继续追加：

```python
print("\n--- 异常演示 ---")

# ① 文件不存在
try:
    with open("not_exist.json", "r", encoding="utf-8") as f:
        json.load(f)
except FileNotFoundError:
    print("配置文件不存在，使用默认配置")

# ② JSON 格式错误（不是合法 JSON）
bad_json = "{name: 'tom', age: 18}"     # 键没引号、用了单引号
try:
    json.loads(bad_json)
except json.JSONDecodeError as e:
    print("JSON 格式错误:", e)

# ③ 缺字段时用 get 兜底，避免 KeyError
api_key = loaded_config.get("api_key", "未配置")
print("api_key:", api_key)
```

**预期输出（异常消息的具体文字可能不同）**：

```
--- 异常演示 ---
配置文件不存在，使用默认配置
JSON 格式错误: Expecting property name enclosed in double quotes: ...
api_key: 未配置
```

> 这三段就是 Agent 工程的日常：**文件可能不在、模型可能返回伪 JSON、配置项可能缺**。

---

## 完整脚本（对照检查）

<details>
<summary>点开查看完整代码</summary>

```python
import json

# ---------- 配置 ----------
config = {
    "model": "gpt-4o-mini",
    "temperature": 0.7,
    "max_tokens": 1024,
    "system_prompt": "你是一个简洁的中文助手"
}

with open("agent_config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, ensure_ascii=False, indent=2)

print("配置已写入 agent_config.json")

with open("agent_config.json", "r", encoding="utf-8") as f:
    loaded_config = json.load(f)

print("当前模型:", loaded_config["model"])
print("温度:", loaded_config["temperature"])

# ---------- 对话历史 ----------
history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"}
]
history.append({"role": "user", "content": "介绍一下 Python"})

with open("chat_history.json", "w", encoding="utf-8") as f:
    json.dump(history, f, ensure_ascii=False, indent=2)

print(f"\n对话已保存，共 {len(history)} 条")

with open("chat_history.json", "r", encoding="utf-8") as f:
    loaded_history = json.load(f)

print("--- 对话记录 ---")
for msg in loaded_history:
    print(f"{msg['role']}: {msg['content']}")

# ---------- 日志 ----------
with open("agent.log", "a", encoding="utf-8") as f:
    f.write("INFO 加载配置成功\n")
    f.write("INFO 保存对话成功\n")

print("\n--- 日志 ---")
with open("agent.log", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())

# ---------- 异常演示 ----------
print("\n--- 异常演示 ---")

try:
    with open("not_exist.json", "r", encoding="utf-8") as f:
        json.load(f)
except FileNotFoundError:
    print("配置文件不存在，使用默认配置")

bad_json = "{name: 'tom', age: 18}"
try:
    json.loads(bad_json)
except json.JSONDecodeError as e:
    print("JSON 格式错误:", e)

api_key = loaded_config.get("api_key", "未配置")
print("api_key:", api_key)
```

</details>

---

## 验证标准

- [ ] `agent_config.json` 打开后中文可读、有缩进
- [ ] 读回的 model 和 temperature 与写入一致
- [ ] `chat_history.json` 是 3 条消息的数组
- [ ] 遍历输出格式为 `role: content`
- [ ] `agent.log` 每次运行追加 2 行
- [ ] 文件不存在时输出"配置文件不存在，使用默认配置"，程序不崩
- [ ] 非法 JSON 被 `JSONDecodeError` 捕获
- [ ] 能用 `get` 避免 KeyError
- [ ] 能说出 `dumps` 和 `dump` 的区别

---

## 练习任务

1. **扩展配置**：给 config 增加 `"tools": ["search", "calculator"]` 和 `"timeout": 30`，重新写入并读回打印。
2. **多轮对话**：再追加一条 assistant 消息，重新保存，确认文件里变成 4 条。
3. **缺配置兜底**：写一个函数 `load_config(path)`，文件不存在时返回默认配置（用 try/except FileNotFoundError 实现）。
4. **按角色筛选**：从 `loaded_history` 里用列表推导或循环找出所有 `role == "user"` 的消息并打印。
5. **（选做）文本日志**：把每条对话写成一行 `时间 | role | content` 追加到 `chat.log`。

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `FileNotFoundError` | 运行目录不对 | `cd` 到 `python-demo` 再运行，文件会生成在当前目录 |
| 中文变成 `\\u4f60\\u597d` 这类转义 | 没加 `ensure_ascii=False` | 加上该参数 |
| `JSONDecodeError` | 用了单引号、键没引号、多了逗号 | JSON 必须双引号、不能有尾随逗号 |
| `Object of type set is not JSON serializable` | 对象里有 set / datetime 等 JSON 不支持的类型 | 先转成 list / 字符串 |
| `PermissionError` | 文件被其他程序占用 | 关掉打开该文件的编辑器/Excel |
| `import json` 报错或行为怪异 | 你的脚本命名成了 `json.py` | 改名，别和标准库重名 |
| 日志越写越多 | `"a"` 是追加模式 | 想清空就删除文件，或临时改 `"w"` |

---

*上一节实操：[01.1 环境搭建 + 基础语法练习](practice-01.1-environment-and-basic-syntax.md) | 下一模块：[02.1 列表 list](../../02-containers/theory/02.1-list.md)*
