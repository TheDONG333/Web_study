# 实操 03.1：函数专项练习

> ⏱ **预计时长**：70 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 检查参数设计是否合理（命名、默认值、是否该用 *args）

---

## 前置要求

- ✅ 已读完理论 03.1 ~ 03.3
- ✅ 会运行 Python 脚本
- ✅ 标准库即可，无需安装依赖

## 使用方式

新建 `functions_drill.py`，按任务顺序写，每完成一题运行一次：

```bash
python functions_drill.py
```

---

## 任务 1：基础函数 + 默认参数

写函数：

```python
def greet(name, greeting="你好"):
    # TODO：返回 "问候语，名字"
```

调用并验证：

```python
print(greet("Tom"))                      # 你好，Tom
print(greet("Tom", "早上好"))             # 早上好，Tom
print(greet(name="Tom", greeting="晚上好"))  # 晚上好，Tom
```

---

## 任务 2：位置参数 + 关键字参数

写函数：

```python
def create_user(name, age, city="北京"):
    # TODO：返回 "name-age-city"
```

验证：

```python
print(create_user("Tom", 25))                    # Tom-25-北京
print(create_user("Tom", 25, "上海"))             # Tom-25-上海
print(create_user(age=25, name="Tom", city="深圳"))  # Tom-25-深圳
```

> 观察：关键字参数可以打乱顺序，但必须放在位置参数后面。

---

## 任务 3：多返回值解包（复习）

写函数 `min_max(nums)`，返回最小值和最大值；调用时解包：

```python
low, high = min_max([3, 1, 5, 2])
print(low, high)      # 1 5
```

---

## 任务 4：默认参数的可变对象坑

先运行错误版本：

```python
def add_item_bad(item, items=[]):
    items.append(item)
    return items

print(add_item_bad("a"))    # ['a']
print(add_item_bad("b"))    # ['a', 'b'] ← 为什么不是 ['b']？
```

再写正确版本 `add_item_ok(item, items=None)`，验证两次调用互不影响。

**预期**：

```
['a']
['b']
```

---

## 任务 5：*args 求和

```python
def sum_all(*nums):
    # TODO：返回所有数字之和
```

验证：

```python
print(sum_all(1, 2, 3))          # 6
print(sum_all(1, 2, 3, 4, 5))    # 15
print(sum_all())                 # 0
```

> 提示：函数内 `nums` 是元组，可以直接 `sum(nums)`。

---

## 任务 6：**kwargs 构建 Prompt

```python
def build_prompt(**kwargs):
    # 从 kwargs 里取 role（默认 "user"）和 content（默认 ""）
    # 返回 "角色：xxx，内容：xxx"
```

验证：

```python
print(build_prompt(role="system", content="你是一个简洁的助手"))
# 角色：system，内容：你是一个简洁的助手

print(build_prompt(content="介绍一下 Python"))
# 角色：user，内容：介绍一下 Python
```

---

## 任务 7：参数解包

### 7.1 列表 → 位置参数

```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))      # 6
```

### 7.2 字典 → 关键字参数

```python
params = {"name": "Tom", "age": 25, "city": "上海"}
print(create_user(**params))    # Tom-25-上海
```

> 试试去掉 `**` 直接写 `create_user(params)`，观察报错——理解为什么要解包。

---

## 任务 8：作用域

```python
name = "全局"

def outer():
    name = "局部"
    print(name)

outer()
print(name)
```

**预期**：`局部` → `全局`。

再用 `global` 演示一次（了解即可）：

```python
count = 0

def increase():
    global count
    count += 1

increase()
print(count)      # 1
```

> 实际项目里尽量用参数和返回值，不要用 global。

---

## 任务 9：lambda + 排序 / 转换 / 过滤

```python
products = [
    {"title": "iPhone", "price": 7999},
    {"title": "键盘", "price": 299},
    {"title": "显示器", "price": 1500}
]
```

完成：

1. 按价格升序排序，打印标题顺序
2. 提取所有标题（map + lambda，记得 list()）
3. 过滤出价格低于 1000 的商品（filter + lambda）
4. 用 `max` + lambda 找出最贵的商品标题

**预期**：

```
['键盘', '显示器', 'iPhone']
['iPhone', '键盘', '显示器']
[{'title': '键盘', 'price': 299}]
iPhone
```

---

## 任务 10：常用内置函数

依次打印结果并核对：

```python
print(len([1, 2, 3]))                     # 3
print(type(3.14))                         # <class 'float'>
print(isinstance(True, int))              # True（bool 是 int 的子类）
print(list(range(1, 5)))                  # [1, 2, 3, 4]
print(list(enumerate(["a", "b"], start=1)))  # [(1, 'a'), (2, 'b')]
print(list(zip([1, 2], ["a", "b"])))      # [(1, 'a'), (2, 'b')]
print(sorted([3, 1, 2]))                  # [1, 2, 3]
print(max([3, 1, 5]), min([3, 1, 5]))     # 5 1
print(max([], default=0))                 # 0
```

---

## 验证标准

- [ ] 任务 1~3 输出与预期一致
- [ ] 任务 4 能解释为什么默认列表会"记住"上次的值，并写出正确版本
- [ ] 任务 5 空参数返回 0
- [ ] 任务 6 的 kwargs 默认值正确
- [ ] 任务 7 列表/字典解包都能传参
- [ ] 任务 8 能说出局部和全局的区别
- [ ] 任务 9 四个结果全部正确
- [ ] 任务 10 十个内置函数结果全部正确
- [ ] 能说出 map/filter 为什么需要 list()

---

## 参考答案（先自己做）

<details>
<summary>点开查看</summary>

```python
# 任务 1
def greet(name, greeting="你好"):
    return f"{greeting}，{name}"

# 任务 2
def create_user(name, age, city="北京"):
    return f"{name}-{age}-{city}"

# 任务 3
def min_max(nums):
    return min(nums), max(nums)

low, high = min_max([3, 1, 5, 2])
print(low, high)

# 任务 4
def add_item_ok(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

print(add_item_ok("a"))    # ['a']
print(add_item_ok("b"))    # ['b']

# 任务 5
def sum_all(*nums):
    return sum(nums)

# 任务 6
def build_prompt(**kwargs):
    role = kwargs.get("role", "user")
    content = kwargs.get("content", "")
    return f"角色：{role}，内容：{content}"

# 任务 7
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))

params = {"name": "Tom", "age": 25, "city": "上海"}
print(create_user(**params))

# 任务 8
name = "全局"
def outer():
    name = "局部"
    print(name)
outer()
print(name)

count = 0
def increase():
    global count
    count += 1
increase()
print(count)

# 任务 9
products = [
    {"title": "iPhone", "price": 7999},
    {"title": "键盘", "price": 299},
    {"title": "显示器", "price": 1500}
]
by_price = sorted(products, key=lambda p: p["price"])
print([p["title"] for p in by_price])
print(list(map(lambda p: p["title"], products)))
print(list(filter(lambda p: p["price"] < 1000, products)))
print(max(products, key=lambda p: p["price"])["title"])

# 任务 10
print(len([1, 2, 3]))
print(type(3.14))
print(isinstance(True, int))
print(list(range(1, 5)))
print(list(enumerate(["a", "b"], start=1)))
print(list(zip([1, 2], ["a", "b"])))
print(sorted([3, 1, 2]))
print(max([3, 1, 5]), min([3, 1, 5]))
print(max([], default=0))
```

</details>

---

*上一节理论：[03.4 装饰器](../theory/03.4-decorators.md) | 下一步：[实操 03.2 Agent 工具装饰器](practice-03.2-agent-tool-decorators.md)*
