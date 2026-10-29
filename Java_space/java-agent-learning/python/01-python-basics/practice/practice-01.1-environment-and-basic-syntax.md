# 实操 01.1：环境搭建 + 基础语法练习

> ⏱ **预计时长**：90 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：环境报错把完整命令行输出发给 AI，让它按你的系统给出修复步骤

---

## 前置要求

- ✅ 已读完理论 01.1 ~ 01.4
- ✅ Windows 系统，有 PowerShell 或 Git Bash
- ✅ 已安装 VS Code（推荐）或 PyCharm

## 实践目标

1. 装好 Python 3.11/3.12，创建并激活 venv
2. 跑通第一个脚本，学会断点调试
3. 完成字符串、流程控制、异常处理三组小练习

---

## Step 1：安装并验证 Python

1. 到 [python.org/downloads](https://www.python.org/downloads/) 下载 **3.12.x**
2. 安装时勾选 **Add python.exe to PATH**
3. 打开终端验证：

```bash
python --version
pip --version
```

预期：

```
Python 3.12.x
pip 24.x ...
```

> 如果 `python` 打不开，试试 `py -3.12 --version`。

---

## Step 2：创建项目 + venv（重点）

```bash
mkdir python-demo
cd python-demo

python -m venv .venv
```

**激活**（按你的终端选一条）：

```bash
# PowerShell
.venv\Scripts\Activate.ps1

# Git Bash
source .venv/Scripts/activate
```

成功标志：命令行前面出现 `(.venv)`。

验证装包：

```bash
pip install requests
pip freeze > requirements.txt
```

看 `requirements.txt` 里是否有 `requests==...`。

> **验收**：`pip list` 能看到 requests；关闭终端再打开，需要重新激活 venv（这是正常的）。

---

## Step 3：第一个脚本 + 断点调试

新建 `main.py`：

```python
import json

def build_greeting(name, age):
    return f"你好，{name}，你今年 {age} 岁。"

message = build_greeting("Tom", 25)
print(message)

data = {"name": "Tom", "age": 25}
print(json.dumps(data, ensure_ascii=False))
```

运行：

```bash
python main.py
```

预期：

```
你好，Tom，你今年 25 岁。
{"name": "Tom", "age": 25}
```

**调试练习**：

1. 在第 8 行（`message = ...`）行号左边点一下，出现红点
2. 按 **F5**，选择 Python 调试器
3. 程序停在红点处，左侧 Variables 面板能看到 `name`、`age`
4. 按 **F10** 单步执行，观察 `message` 出现

> 不会调试就让 AI 按 VS Code 版本给你截图级步骤。

---

## Step 4：模块导入练习

新建 `imports_demo.py`：

```python
import math
from datetime import datetime
import json as j

print(math.sqrt(16))                 # 4.0
print(datetime.now().year)           # 当前年份
print(j.dumps({"ok": True}))         # {"ok": true}
```

运行 `python imports_demo.py`，确认三行输出。

---

## 任务 1：变量与类型

新建 `task_types.py`，定义以下变量并打印：

```python
name = "Tom"
age = 25
height = 1.75
is_student = False
nothing = None
```

要求输出：

```
Tom <class 'str'>
25 <class 'int'>
1.75 <class 'float'>
False <class 'bool'>
None <class 'NoneType'>
```

提示：用 f-string + `type()`。

---

## 任务 2：字符串操作（重点）

```python
raw = "  请帮我总结这段文字，并提取关键词。  "
```

按顺序完成并打印：

1. 去掉首尾空格 → `clean`
2. `clean` 的长度
3. 查找 "总结" 的下标
4. 判断是否以 "请" 开头
5. 把 "文字" 替换成 "文本"
6. 按 "，" 切分成列表
7. 用 `" | "` 把切分结果重新连接
8. 取 `clean` 的前 5 个字符
9. 用 f-string 拼出 Prompt：

```
请完成以下任务：请帮我总结这段文本，并提取关键词。
要求：不超过 100 字。
```

**预期前几项**：

```
17
3
True
```

> 提示：`len(clean)` 需要先算；下标从 0 开始。

---

## 任务 3：流程控制

新建 `task_flow.py`，完成：

### 3.1 成绩分级

```python
score = 85
```

用 if/elif/else 输出：≥90 优秀，≥60 及格，否则不及格。

### 3.2 三元表达式

用一行代码给 `age = 20` 生成 `"成年"` / `"未成年"`。

### 3.3 遍历对话历史

```python
history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"},
    {"role": "user", "content": "介绍一下 Python"}
]
```

用 `for` + `enumerate` 输出：

```
1. [user] 你好
2. [assistant] 你好，有什么可以帮你？
3. [user] 介绍一下 Python
```

### 3.4 while + break/continue

- 用 while 计算 1 到 100 的和
- 用 for 遍历 `range(10)`，跳过 3，遇到 6 时停止，打印其余数字

预期输出：

```
5050
0 1 2 4 5
```

---

## 任务 4：异常处理

新建 `task_exception.py`，依次完成：

### 4.1 捕获 ValueError

```python
try:
    number = int("abc")
except ValueError as e:
    print("转换失败:", e)
```

### 4.2 捕获 KeyError

```python
user = {"name": "Tom"}
# 访问 user["age"]，捕获 KeyError
```

### 4.3 捕获 IndexError

```python
nums = [1, 2, 3]
# 访问 nums[10]，捕获 IndexError
```

### 4.4 捕获 TypeError

```python
# "a" + 1，捕获 TypeError
```

### 4.5 raise 主动抛异常

写一个函数 `validate_age(age)`：

- `age < 0` → `raise ValueError("年龄不能为负数")`
- 正常返回 age

调用 `validate_age(-1)`，用 try/except 捕获并打印错误消息。

**预期输出**（示例）：

```
转换失败: invalid literal for int() with base 10: 'abc'
缺少 age 字段
下标越界
类型错误
参数校验失败: 年龄不能为负数
```

> 具体的异常消息文字可能略有不同，捕获到正确类型即可。

---

## 验证标准

- [ ] `python --version` 显示 3.11/3.12
- [ ] venv 创建并激活成功，命令行有 `(.venv)`
- [ ] `requirements.txt` 里有 requests
- [ ] `main.py` 正常输出两行
- [ ] 会打断点、F5 调试、F10 单步
- [ ] 三种 import 写法都能跑
- [ ] 任务 1 五个类型全部正确
- [ ] 任务 2 字符串九步全部正确
- [ ] 任务 3 分级、三元、遍历、循环全部正确
- [ ] 任务 4 五类异常都能捕获，raise 生效

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `'python' 不是内部或外部命令` | 安装没勾 PATH | 重装勾选 Add to PATH，或用 `py -3.12` |
| PowerShell 提示禁止运行 Activate.ps1 | 执行策略限制 | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`，再激活 |
| `ModuleNotFoundError: No module named 'requests'` | venv 没激活，装到全局了 | 先激活（出现 `(.venv)`）再 `pip install` |
| pip 安装很慢 | 没配镜像 | `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple` |
| 中文输出乱码 | 终端编码问题 | 用 Windows Terminal / PowerShell；或设 `PYTHONUTF8=1` |
| `IndentationError` | 缩进不一致 | 统一用 4 个空格，别混 Tab |
| `FileNotFoundError` | 路径不对 | 确认脚本在正确的目录下运行 |

---

## 参考答案（先自己做）

<details>
<summary>点开查看</summary>

```python
# 任务 1
print(f"{name} {type(name)}")
print(f"{age} {type(age)}")
print(f"{height} {type(height)}")
print(f"{is_student} {type(is_student)}")
print(f"{nothing} {type(nothing)}")

# 任务 2
raw = "  请帮我总结这段文字，并提取关键词。  "
clean = raw.strip()
print(len(clean))                       # 17
print(clean.find("总结"))                # 3
print(clean.startswith("请"))            # True
print(clean.replace("文字", "文本"))
parts = clean.split("，")
print(parts)
print(" | ".join(parts))
print(clean[:5])
prompt = f"""请完成以下任务：{clean.replace('文字', '文本')}
要求：不超过 100 字。"""
print(prompt)

# 任务 3
score = 85
if score >= 90:
    print("优秀")
elif score >= 60:
    print("及格")
else:
    print("不及格")

age = 20
print("成年" if age >= 18 else "未成年")

history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"},
    {"role": "user", "content": "介绍一下 Python"}
]
for i, msg in enumerate(history, start=1):
    print(f"{i}. [{msg['role']}] {msg['content']}")

total = 0
n = 1
while n <= 100:
    total += n
    n += 1
print(total)

for i in range(10):
    if i == 3:
        continue
    if i == 6:
        break
    print(i, end=" ")
print()

# 任务 4
try:
    int("abc")
except ValueError as e:
    print("转换失败:", e)

try:
    {"name": "Tom"}["age"]
except KeyError:
    print("缺少 age 字段")

try:
    [1, 2, 3][10]
except IndexError:
    print("下标越界")

try:
    "a" + 1
except TypeError:
    print("类型错误")

def validate_age(age):
    if age < 0:
        raise ValueError("年龄不能为负数")
    return age

try:
    validate_age(-1)
except ValueError as e:
    print("参数校验失败:", e)
```

</details>

---

*上一节理论：[01.4 异常处理](../theory/01.4-exception-handling.md) | 下一步：[实操 01.2 文件 + JSON](practice-01.2-json-and-file-agent-demo.md)*
