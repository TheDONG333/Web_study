# 实操 02.1：列表与字典专项练习

> ⏱ **预计时长**：75 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 检查有没有更简洁的推导式写法

---

## 前置要求

- ✅ 已读完理论 02.1 ~ 02.3
- ✅ 已完成模块 1，会运行 Python 脚本
- ✅ 不需要安装依赖，标准库即可

## 使用方式

在 `python-demo`（或任意目录）新建 `containers_drill.py`，按任务顺序写代码，运行：

```bash
python containers_drill.py
```

> 每完成一个任务就运行一次，对照预期输出。不要一次全写完再跑。

---

## 任务 1：列表增删改查

```python
nums = [3, 1, 2]
```

依次执行并打印：

1. `append(4)`
2. `extend([5, 6])`
3. `insert(0, 0)`
4. `pop()`（打印返回值）
5. `remove(2)`
6. `index(3)`
7. `sort()`
8. `reverse()`

**预期输出**：

```
6
[0, 3, 1, 4, 5]
1
[5, 4, 3, 1, 0]
```

> 中间过程自己对照：`[0,3,1,2,4,5,6] → pop 得 6 → 去掉 2 → ...`

---

## 任务 2：切片

```python
nums = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

打印：

| 表达式 | 预期 |
|--------|------|
| `nums[2:5]` | `[2, 3, 4]` |
| `nums[:3]` | `[0, 1, 2]` |
| `nums[-3:]` | `[7, 8, 9]` |
| `nums[::2]` | `[0, 2, 4, 6, 8]` |
| `nums[::-1]` | `[9, 8, 7, 6, 5, 4, 3, 2, 1, 0]` |

---

## 任务 3：遍历

```python
fruits = ["苹果", "香蕉", "橙子"]
```

用 `enumerate` 输出：

```
0 苹果
1 香蕉
2 橙子
```

---

## 任务 4：列表推导式（重点）

### 4.1 平方

```python
nums = [1, 2, 3, 4, 5]
```

生成平方列表，预期 `[1, 4, 9, 16, 25]`。

### 4.2 过滤偶数

生成 1~10 中的偶数，预期 `[2, 4, 6, 8, 10]`。

### 4.3 提取字段

```python
products = [
    {"title": "iPhone 15", "category": "手机", "price": 7999},
    {"title": "小米 14", "category": "手机", "price": 5299},
    {"title": "机械键盘", "category": "外设", "price": 299}
]
```

- 提取所有标题，预期 `['iPhone 15', '小米 14', '机械键盘']`
- 提取所有手机分类的标题，预期 `['iPhone 15', '小米 14']`
- 提取价格低于 1000 的标题，预期 `['机械键盘']`

### 4.4 处理对话历史

```python
history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"},
    {"role": "user", "content": "介绍一下 Python"}
]
```

- 提取所有用户消息的 content，预期 `['你好', '介绍一下 Python']`

---

## 任务 5：嵌套列表

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]
```

完成：

1. 打印 `matrix[1][2]`，预期 `6`
2. 打印每一行的和，预期 `6 15 24`
3. 用嵌套循环打印所有元素
4. 把 `matrix[0][0]` 改成 `99`，再打印整个 matrix
5. （了解）用推导式展平成一维列表，预期 `[99, 2, 3, 4, 5, 6, 7, 8, 9]`

---

## 任务 6：字典增删改查

```python
user = {"name": "Tom", "age": 25}
```

依次完成：

1. 新增 `email`：`tom@demo.com`
2. 修改 `age` 为 26
3. 打印 `user["name"]`
4. 用 `get` 读取不存在的 `phone`，默认值 `"未填写"`
5. `pop("email")` 并打印返回值
6. 打印 `keys()`、`values()`、`items()`
7. `update({"age": 27, "city": "北京"})` 后打印整个字典

**预期最后一行**：

```
{'name': 'Tom', 'age': 27, 'city': '北京'}
```

---

## 任务 7：遍历字典

用 `for key, value in user.items()` 输出：

```
name = Tom
age = 27
city = 北京
```

---

## 任务 8：字典推导式

```python
prices = {"apple": 5, "banana": 3, "orange": 8}
```

1. 所有价格打 8 折，预期 `{'apple': 4.0, 'banana': 2.4, 'orange': 6.4}`
2. 只保留价格 ≥ 5 的，预期 `{'apple': 5, 'orange': 8}`

---

## 任务 9：嵌套字典

```python
agent_state = {
    "session_id": "s001",
    "config": {"model": "gpt-4o-mini", "temperature": 0.7},
    "user": {"id": 1001, "name": "Tom", "tags": ["vip", "new"]},
    "tools": {
        "search": {"enabled": True, "timeout": 30},
        "calculator": {"enabled": False, "timeout": 5}
    }
}
```

完成：

1. 打印 model，预期 `gpt-4o-mini`
2. 打印用户的第一个标签，预期 `vip`
3. 把 temperature 改成 `0.3`
4. 把 search 的 timeout 改成 `60`
5. 用 get 链安全读取 `agent_state["config"]["top_p"]`，默认 `1.0`
6. 用 get 链安全读取不存在的 `agent_state["memory"]["max_turns"]`，默认 `10`

---

## 任务 10：元组与集合

```python
point = (3, 5)
nums = [1, 2, 2, 3, 3, 3]
```

完成：

1. 解包 point 为 x、y 并打印
2. 交换 `a, b = 1, 2` 的值
3. 写函数 `get_user()` 返回 `(1001, "Tom")`，接收时解包
4. 用 set 对 nums 去重并打印（顺序可能不同）
5. 求两个集合的交、并、差：

```python
a = {"vip", "new", "active"}
b = {"vip", "active", "churn"}
```

预期：交集 `{'vip', 'active'}`，差集 `{'new'}`，并集包含 4 个元素。

---

## 验证标准

- [ ] 任务 1 的 pop 返回 6，最终列表为 `[5,4,3,1,0]`
- [ ] 任务 2 五个切片全部正确
- [ ] 任务 3 的 enumerate 输出正确
- [ ] 任务 4 四组推导式全部正确
- [ ] 任务 5 能读写嵌套列表并展平
- [ ] 任务 6 字典增删改查、get 默认值正确
- [ ] 任务 7 用 items 遍历正确
- [ ] 任务 8 两个字典推导式正确
- [ ] 任务 9 嵌套读写和 get 链正确
- [ ] 任务 10 元组解包、set 去重和交并差正确

---

## 参考答案（先自己做）

<details>
<summary>点开查看</summary>

```python
# 任务 1
nums = [3, 1, 2]
nums.append(4)
nums.extend([5, 6])
nums.insert(0, 0)
print(nums.pop())        # 6
nums.remove(2)
print(nums)              # [0, 3, 1, 4, 5]
print(nums.index(3))     # 1
nums.sort()
nums.reverse()
print(nums)              # [5, 4, 3, 1, 0]

# 任务 2
nums = list(range(10))
print(nums[2:5], nums[:3], nums[-3:], nums[::2], nums[::-1])

# 任务 3
for i, fruit in enumerate(["苹果", "香蕉", "橙子"]):
    print(i, fruit)

# 任务 4
print([n * n for n in [1, 2, 3, 4, 5]])
print([n for n in range(1, 11) if n % 2 == 0])

products = [
    {"title": "iPhone 15", "category": "手机", "price": 7999},
    {"title": "小米 14", "category": "手机", "price": 5299},
    {"title": "机械键盘", "category": "外设", "price": 299}
]
print([p["title"] for p in products])
print([p["title"] for p in products if p["category"] == "手机"])
print([p["title"] for p in products if p["price"] < 1000])

history = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"},
    {"role": "user", "content": "介绍一下 Python"}
]
print([m["content"] for m in history if m["role"] == "user"])

# 任务 5
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
print(matrix[1][2])                      # 6
for row in matrix:
    print(sum(row), end=" ")             # 6 15 24
print()
for row in matrix:
    for num in row:
        print(num, end=" ")
print()
matrix[0][0] = 99
print(matrix)
print([num for row in matrix for num in row])

# 任务 6
user = {"name": "Tom", "age": 25}
user["email"] = "tom@demo.com"
user["age"] = 26
print(user["name"])
print(user.get("phone", "未填写"))
print(user.pop("email"))
print(user.keys(), user.values(), user.items())
user.update({"age": 27, "city": "北京"})
print(user)

# 任务 7
for key, value in user.items():
    print(f"{key} = {value}")

# 任务 8
prices = {"apple": 5, "banana": 3, "orange": 8}
print({k: v * 0.8 for k, v in prices.items()})
print({k: v for k, v in prices.items() if v >= 5})

# 任务 9
agent_state = {
    "session_id": "s001",
    "config": {"model": "gpt-4o-mini", "temperature": 0.7},
    "user": {"id": 1001, "name": "Tom", "tags": ["vip", "new"]},
    "tools": {
        "search": {"enabled": True, "timeout": 30},
        "calculator": {"enabled": False, "timeout": 5}
    }
}
print(agent_state["config"]["model"])
print(agent_state["user"]["tags"][0])
agent_state["config"]["temperature"] = 0.3
agent_state["tools"]["search"]["timeout"] = 60
print(agent_state.get("config", {}).get("top_p", 1.0))
print(agent_state.get("memory", {}).get("max_turns", 10))

# 任务 10
point = (3, 5)
x, y = point
print(x, y)
a, b = 1, 2
a, b = b, a
print(a, b)

def get_user():
    return 1001, "Tom"

user_id, name = get_user()
print(user_id, name)

nums = [1, 2, 2, 3, 3, 3]
print(set(nums))

s1 = {"vip", "new", "active"}
s2 = {"vip", "active", "churn"}
print(s1 & s2)
print(s1 - s2)
print(s1 | s2)
```

</details>

---

*上一节理论：[02.4 了解即可](../theory/02.4-collections-and-iterators.md) | 下一步：[实操 02.2 Agent 数据模型综合](practice-02.2-agent-data-model.md)*
