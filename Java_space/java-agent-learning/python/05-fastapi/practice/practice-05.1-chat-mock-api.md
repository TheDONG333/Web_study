# 实操 05.1：最简对话模拟接口（GET + /docs 调试）

> ⏱ **预计时长**：45 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：接口返回不对时，把 `/docs` 里的请求参数和响应一起发给 AI

---

## 前置要求

- ✅ 已读完理论 05.1 ~ 05.4
- ✅ 已完成模块 1 的环境搭建，会用 venv
- ✅ 不需要大模型 API，本次返回固定模拟回复

## 实践目标

1. 建一个 FastAPI 项目并启动
2. 写一个 GET `/chat` 接口，接收 `message` 查询参数
3. 返回固定格式的模拟回复
4. 在 `/docs` 里手动调用、查看返回
5. 验证空参数返回 422

---

## Step 1：创建项目与虚拟环境

```bash
mkdir fastapi-chat-demo
cd fastapi-chat-demo

python -m venv .venv
```

激活（按终端选）：

```bash
# PowerShell
.venv\Scripts\Activate.ps1

# Git Bash
source .venv/Scripts/activate
```

安装依赖：

```bash
pip install fastapi "uvicorn[standard]"
pip freeze > requirements.txt
```

---

## Step 2：写 main.py

新建 `main.py`：

```python
from fastapi import FastAPI, Query

app = FastAPI(
    title="Agent Chat API",
    description="最简对话模拟接口，用于练习 FastAPI 基础",
    version="0.1.0"
)

@app.get("/")
def root():
    return {"message": "Agent Chat API is running"}

@app.get("/chat", summary="模拟对话", tags=["chat"])
def chat(
    message: str = Query(
        ...,
        description="用户输入的消息",
        min_length=1,
        max_length=200
    )
):
    """
    接收用户消息，返回固定的模拟回复。
    """
    reply = f"你说的是：{message}。这是模拟回复，后续会接入大模型。"
    return {
        "code": 200,
        "message": "ok",
        "data": {"reply": reply}
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

> 三个要点：`Query(...)` 表示必填、`description` 会显示在文档里、返回 `dict` 自动转 JSON。

---

## Step 3：启动服务

在 `main.py` 所在目录执行：

```bash
uvicorn main:app --reload
```

看到：

```
Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

> 也可以直接 `python main.py`，效果一样。

---

## Step 4：浏览器直接调用

访问：

```
http://127.0.0.1:8000/chat?message=你好
```

**预期返回**：

```json
{
  "code": 200,
  "message": "ok",
  "data": {
    "reply": "你说的是：你好。这是模拟回复，后续会接入大模型。"
  }
}
```

---

## Step 5：在 /docs 里调试（重点）

1. 打开 `http://127.0.0.1:8000/docs`
2. 找到 **GET /chat**（在 chat 分组下）
3. 点 **Try it out**
4. `message` 输入：`你好`
5. 点 **Execute**
6. 下方 **Response body** 应看到上面的 JSON

### 观察文档里显示了什么

| 文档项 | 来源 |
|--------|------|
| 接口标题"模拟对话" | `summary` |
| 接口说明 | docstring |
| 分组 chat | `tags` |
| 参数说明"用户输入的消息" | `Query(description=...)` |
| 参数必填 | `...` |
| 长度限制 | `min_length` / `max_length` |

> 这就是 FastAPI 最大的效率优势：**代码写完，文档和调试台都有了**。

---

## Step 6：验证参数校验

### 6.1 空消息

在 `/docs` 里把 `message` 清空后 Execute，返回 **422**：

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["query", "message"],
      "msg": "String should have at least 1 character",
      "input": ""
    }
  ]
}
```

### 6.2 超长消息

输入 201 个字符，同样返回 422（`max_length=200`）。

### 6.3 不带参数

浏览器访问：

```
http://127.0.0.1:8000/chat
```

返回 422，提示缺少 `message`。

---

## Step 7：查看 ReDoc

打开：

```
http://127.0.0.1:8000/redoc
```

能看到同样的接口，但排版更适合阅读，不能交互调试。

---

## 验证标准

- [ ] 服务能在 8000 端口启动
- [ ] 访问 `/` 返回运行提示
- [ ] `/chat?message=你好` 返回固定格式 JSON
- [ ] `/docs` 能看到 GET /chat，标题、说明、参数描述都显示
- [ ] 在 `/docs` 中 Try it out → Execute 能拿到正确响应
- [ ] 空 message 返回 422
- [ ] 超长 message 返回 422
- [ ] 不带 message 返回 422
- [ ] `/redoc` 能正常打开
- [ ] 修改代码保存后，服务自动重启（--reload 生效）

---

## 练习任务

1. **加回复风格参数**：新增可选查询参数 `style: str = Query("friendly", description="回复风格")`，在返回里带上 style。
2. **加路径参数接口**：`GET /chat/{user_id}`，接收 `user_id: int` 和可选 `message`，返回带用户 ID 的回复。
3. **加长度校验**：把 `message` 的 `max_length` 改成 50，测试超长返回 422。
4. **补文档**：给 `/` 接口加 `summary` 和 `tags`，在 `/docs` 里确认显示。
5. **（选做）再写一个 `/health` 接口**：返回 `{"status": "ok"}`，用于健康检查。

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `uvicorn: command not found` | venv 没激活 / 没装 uvicorn | 激活 venv 后 `pip install "uvicorn[standard]"` |
| 启动报 `Could not import module "main"` | 不在 main.py 所在目录 | `cd` 到项目目录再启动 |
| 404 | 路径写错 / 模块名不对 | 检查 `@app.get` 路径和 `main:app` |
| 422 | 参数缺失或类型不对 | 看响应里的 `loc` 和 `msg` |
| 8000 端口被占用 | 其他服务在用 | `--port 8001`，或停掉旧进程 |
| 改代码没重启 | 没加 `--reload` | 启动命令加 `--reload` |
| `/docs` 打不开 | 服务没启动 / 端口错 | 先确认根路径能访问 |
| 中文乱码 | 浏览器/终端编码 | 用现代浏览器，响应是 UTF-8 JSON |

---

*上一节理论：[05.4 Java 概念对照](../theory/05.4-spring-mapping.md) | 下一模块：[06.1 Pydantic 基础](../../06-pydantic/theory/06.1-pydantic-basics.md)*
