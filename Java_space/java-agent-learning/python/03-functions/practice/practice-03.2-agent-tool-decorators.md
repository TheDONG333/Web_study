# 实操 03.2：Agent 工具装饰器（日志 / 计时 / 重试 / 权限）

> ⏱ **预计时长**：75 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：装饰器写错时，让 AI 按"三层结构"帮你逐层检查，不要直接要答案

---

## 前置要求

- ✅ 已读完理论 03.4（装饰器）
- ✅ 已完成实操 [03.1](practice-03.1-function-drills.md)
- ✅ 标准库即可（`time`、`functools`）

## 实践目标

写一个迷你工具系统，模拟 Agent 调用工具的真实流程：

1. 用字典做**工具注册表**
2. 给工具加**日志**装饰器
3. 给工具加**计时**装饰器
4. 给工具加**失败重试**装饰器（带参数）
5. 给敏感工具加**权限校验**装饰器
6. 用 `**kwargs` 模拟大模型 function calling 的参数分发

最终你会得到一个可运行的"工具执行器"雏形。

---

## Step 1：工具注册表 + 两个基础工具

新建 `tool_decorators.py`：

```python
import time
from functools import wraps

TOOLS = {}

def register_tool(func):
    TOOLS[func.__name__] = func
    return func

@register_tool
def search(query, top_k=5):
    time.sleep(0.05)        # 模拟网络耗时
    return f"搜索「{query}」返回 {top_k} 条结果"

@register_tool
def calculator(a, b, op="add"):
    if op == "add":
        return a + b
    if op == "multiply":
        return a * b
    raise ValueError(f"不支持的操作: {op}")

print("已注册工具:", list(TOOLS.keys()))
```

**运行预期**：

```
已注册工具: ['search', 'calculator']
```

> `register_tool` 是最外层装饰器：函数定义完就自动登记到 `TOOLS` 字典。

---

## Step 2：日志装饰器

在 `TOOLS` 定义之后、工具函数之前加入：

```python
def log_tool(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[LOG] 调用工具 {func.__name__} | args={args} kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"[LOG] {func.__name__} 返回: {result}")
        return result
    return wrapper
```

给 `search` 和 `calculator` 加上 `@log_tool`（放在 `@register_tool` 下面）：

```python
@register_tool
@log_tool
def search(query, top_k=5):
    ...
```

> 装饰器顺序：`@register_tool` 在最上面，意味着它最后执行、登记的是包装后的函数。**注册器放最外层**。

---

## Step 3：计时装饰器

```python
def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        cost = (time.time() - start) * 1000
        print(f"[TIMER] {func.__name__} 耗时 {cost:.1f}ms")
        return result
    return wrapper
```

给 `search` 再加一层（放在 `@log_tool` 下面）：

```python
@register_tool
@log_tool
@timer
def search(query, top_k=5):
    ...
```

> 应用顺序是从下往上：`search` 先被 `timer` 包，再被 `log_tool` 包，最后被 `register_tool` 登记。

---

## Step 4：带参数的重试装饰器

```python
def retry(times=3, delay=0.1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    print(f"[RETRY] {func.__name__} 第 {attempt} 次失败: {e}")
                    if attempt < times:
                        time.sleep(delay)
            raise last_error
        return wrapper
    return decorator
```

加一个"前两次必失败"的模拟工具：

```python
_flaky_count = {"n": 0}

@register_tool
@log_tool
@retry(times=3, delay=0.05)
def flaky_tool():
    _flaky_count["n"] += 1
    if _flaky_count["n"] < 3:
        raise RuntimeError("临时故障")
    return "第三次终于成功"
```

> 三层结构：`retry(times=3, delay=0.05)` 先返回 `decorator`，再由它装饰函数。

---

## Step 5：权限装饰器

```python
CURRENT_USER = {"name": "Tom", "role": "user"}

def require_role(role):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if CURRENT_USER["role"] != role:
                raise PermissionError(
                    f"需要 {role} 权限，当前角色: {CURRENT_USER['role']}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@register_tool
@log_tool
@require_role("admin")
def delete_user(user_id):
    return f"已删除用户 {user_id}"
```

> 权限检查放在 wrapper 最前面，**先校验，再执行**。

---

## Step 6：工具分发器（模拟 function calling）

```python
def call_tool(name, arguments):
    if name not in TOOLS:
        raise ValueError(f"未知工具: {name}")
    return TOOLS[name](**arguments)
```

模拟大模型返回的调用请求：

```python
print("=== 1. 模拟大模型 function calling ===")
tool_call = {
    "name": "search",
    "arguments": {"query": "Python 装饰器", "top_k": 2}
}
print("结果:", call_tool(tool_call["name"], tool_call["arguments"]))
```

> `**arguments` 把字典拆成关键字参数——这就是 03.2 讲的参数解包。

---

## Step 7：运行完整流程

在文件末尾加上：

```python
print("\n=== 2. 普通工具 ===")
print(call_tool("calculator", {"a": 2, "b": 3, "op": "multiply"}))

print("\n=== 3. 失败重试 ===")
print(call_tool("flaky_tool", {}))

print("\n=== 4. 权限校验 ===")
try:
    call_tool("delete_user", {"user_id": 1001})
except PermissionError as e:
    print("权限拦截:", e)

CURRENT_USER["role"] = "admin"
print("切换管理员后:", call_tool("delete_user", {"user_id": 1001}))
```

### 预期输出（耗时数字会不同）

```
=== 1. 模拟大模型 function calling ===
[LOG] 调用工具 search | args=() kwargs={'query': 'Python 装饰器', 'top_k': 2}
[TIMER] search 耗时 50.x ms
[LOG] search 返回: 搜索「Python 装饰器」返回 2 条结果
结果: 搜索「Python 装饰器」返回 2 条结果

=== 2. 普通工具 ===
[LOG] 调用工具 calculator | args=() kwargs={'a': 2, 'b': 3, 'op': 'multiply'}
[LOG] calculator 返回: 6
6

=== 3. 失败重试 ===
[LOG] 调用工具 flaky_tool | args=() kwargs={}
[RETRY] flaky_tool 第 1 次失败: 临时故障
[RETRY] flaky_tool 第 2 次失败: 临时故障
[LOG] flaky_tool 返回: 第三次终于成功
第三次终于成功

=== 4. 权限校验 ===
[LOG] 调用工具 delete_user | args=() kwargs={'user_id': 1001}
权限拦截: 需要 admin 权限，当前角色: user
切换管理员后:
[LOG] 调用工具 delete_user | args=() kwargs={'user_id': 1001}
[LOG] delete_user 返回: 已删除用户 1001
已删除用户 1001
```

---

## 完整代码（对照检查）

<details>
<summary>点开查看完整脚本</summary>

```python
import time
from functools import wraps

TOOLS = {}
CURRENT_USER = {"name": "Tom", "role": "user"}

def register_tool(func):
    TOOLS[func.__name__] = func
    return func

def log_tool(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[LOG] 调用工具 {func.__name__} | args={args} kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"[LOG] {func.__name__} 返回: {result}")
        return result
    return wrapper

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        cost = (time.time() - start) * 1000
        print(f"[TIMER] {func.__name__} 耗时 {cost:.1f}ms")
        return result
    return wrapper

def retry(times=3, delay=0.1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    print(f"[RETRY] {func.__name__} 第 {attempt} 次失败: {e}")
                    if attempt < times:
                        time.sleep(delay)
            raise last_error
        return wrapper
    return decorator

def require_role(role):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if CURRENT_USER["role"] != role:
                raise PermissionError(
                    f"需要 {role} 权限，当前角色: {CURRENT_USER['role']}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@register_tool
@log_tool
@timer
def search(query, top_k=5):
    time.sleep(0.05)
    return f"搜索「{query}」返回 {top_k} 条结果"

@register_tool
@log_tool
def calculator(a, b, op="add"):
    if op == "add":
        return a + b
    if op == "multiply":
        return a * b
    raise ValueError(f"不支持的操作: {op}")

_flaky_count = {"n": 0}

@register_tool
@log_tool
@retry(times=3, delay=0.05)
def flaky_tool():
    _flaky_count["n"] += 1
    if _flaky_count["n"] < 3:
        raise RuntimeError("临时故障")
    return "第三次终于成功"

@register_tool
@log_tool
@require_role("admin")
def delete_user(user_id):
    return f"已删除用户 {user_id}"

def call_tool(name, arguments):
    if name not in TOOLS:
        raise ValueError(f"未知工具: {name}")
    return TOOLS[name](**arguments)

print("=== 1. 模拟大模型 function calling ===")
tool_call = {"name": "search", "arguments": {"query": "Python 装饰器", "top_k": 2}}
print("结果:", call_tool(tool_call["name"], tool_call["arguments"]))

print("\n=== 2. 普通工具 ===")
print(call_tool("calculator", {"a": 2, "b": 3, "op": "multiply"}))

print("\n=== 3. 失败重试 ===")
print(call_tool("flaky_tool", {}))

print("\n=== 4. 权限校验 ===")
try:
    call_tool("delete_user", {"user_id": 1001})
except PermissionError as e:
    print("权限拦截:", e)

CURRENT_USER["role"] = "admin"
print("切换管理员后:", call_tool("delete_user", {"user_id": 1001}))
```

</details>

---

## 验证标准

- [ ] 工具能自动注册到 `TOOLS`
- [ ] 调用 search 时能看到调用日志、参数、返回值和耗时
- [ ] calculator 通过 `**arguments` 正确传参
- [ ] flaky_tool 前两次失败、第三次成功
- [ ] 普通用户调用 delete_user 被 PermissionError 拦截
- [ ] 切换 admin 后 delete_user 成功
- [ ] 能说出四个装饰器各自负责什么
- [ ] 能解释 `@wraps` 的作用
- [ ] 能解释 `@register_tool` 为什么放最外层
- [ ] 能说出带参数装饰器的三层结构

---

## 练习任务

1. **加一个天气工具**：`weather(city, unit="c")`，通过 `call_tool("weather", {"city": "北京"})` 调用。
2. **加重试次数**：把 flaky_tool 改成前 4 次失败，`retry(times=5, delay=0.02)`，观察日志。
3. **加调用计数**：写一个 `count_calls` 装饰器，记录每个工具被调用了多少次（提示：用字典和 `func.__name__`）。
4. **权限扩展**：给 `search` 也加 `@require_role("user")`，确认普通用户可以调用。
5. **思考**：
   - 为什么 `wrapper` 必须 `return result`？不返回会怎样？
   - 如果 `@register_tool` 放在最下面（离函数最近），会发生什么？
   - 多个装饰器的执行顺序是怎样的？

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `TypeError: 'NoneType' object is not callable` | 装饰器忘了 `return wrapper` | 检查每层 return |
| 调用工具后返回 None | wrapper 忘了 `return result` | 把原函数结果返回 |
| `TOOLS` 里是原始函数，日志不生效 | 注册器放太里面 | `@register_tool` 放最外层 |
| 权限没生效 | `require_role` 放到了装饰器最里面且被其他逻辑跳过 | 权限校验放 wrapper 最前 |
| `KeyError: 'xxx'` | 工具没注册 / 名字写错 | 打印 `TOOLS.keys()` 检查 |
| 重试不停止 | 异常类型不匹配 / 次数写错 | 检查 `retry(times=...)` 和异常捕获 |
| 计时为 0 | 函数太快 | 加 `time.sleep` 模拟耗时 |

---

*上一节实操：[03.1 函数专项练习](practice-03.1-function-drills.md) | 下一模块：[04.1 类与对象基础](../../04-oop/theory/04.1-class-basics.md)*
