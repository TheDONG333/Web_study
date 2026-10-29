# 实操 06.1：Agent 对话接口（Pydantic 请求/响应模型）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：422 报错时，把请求 JSON 和 `detail` 一起发给 AI，让它按 `loc` 定位

---

## 前置要求

- ✅ 已读完理论 06.1 ~ 06.4
- ✅ 已完成模块 5 的 FastAPI 实操
- ✅ 会用 venv 和 `uvicorn main:app --reload`

## 实践目标

定义一个标准的 Agent 对话接口：

- **请求模型**：用户消息、模型名称、温度、是否流式
- **响应模型**：回复内容、token 消耗、请求耗时
- **统一返回**：`ApiResponse<ChatResponse>`
- **自动校验**：空消息、温度越界、类型错误都返回 422
- **字段过滤**：内部调试字段不出现在响应里

---

## Step 1：创建项目

```bash
mkdir agent-chat-api
cd agent-chat-api

python -m venv .venv
```

激活后安装：

```bash
pip install fastapi "uvicorn[standard]"
pip freeze > requirements.txt
```

---

## Step 2：定义模型

新建 `main.py`，先写模型部分：

```python
import time
from typing import Generic, TypeVar

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(
    title="Agent Chat API",
    description="Pydantic v2 结构化请求/响应示例",
    version="0.1.0"
)

# ── 请求模型 ──
class ChatRequest(BaseModel):
    message: str = Field(
        ..., min_length=1, max_length=2000,
        description="用户消息"
    )
    model: str = Field("gpt-4o-mini", description="模型名称")
    temperature: float = Field(
        0.7, ge=0.0, le=2.0,
        description="温度，0~2"
    )
    stream: bool = Field(False, description="是否流式输出")

# ── 响应模型 ──
class TokenUsage(BaseModel):
    prompt_tokens: int = Field(..., ge=0)
    completion_tokens: int = Field(..., ge=0)
    total_tokens: int = Field(..., ge=0)

class ChatResponse(BaseModel):
    reply: str = Field(..., description="回复内容")
    usage: TokenUsage = Field(..., description="Token 消耗")
    elapsed_ms: int = Field(..., ge=0, description="请求耗时（毫秒）")

# ── 统一返回 ──
T = TypeVar("T")

class ApiResponse(BaseModel, Generic[T]):
    code: int = 200
    message: str = "ok"
    data: T | None = None
```

> 请求模型有 4 个字段，响应模型嵌套了 `TokenUsage`，统一返回用了泛型。

---

## Step 3：写 POST 对话接口

继续在 `main.py` 末尾添加：

```python
@app.post(
    "/chat",
    response_model=ApiResponse[ChatResponse],
    summary="Agent 对话",
    tags=["chat"]
)
def chat(req: ChatRequest):
    """
    接收结构化对话请求，返回模拟回复和 token 消耗。
    """
    start = time.time()
    time.sleep(0.05)          # 模拟模型调用耗时

    reply = (
        f"[{req.model}] 收到：{req.message}"
        f"（temperature={req.temperature}, stream={req.stream}）"
    )

    usage = TokenUsage(
        prompt_tokens=len(req.message),
        completion_tokens=len(reply),
        total_tokens=len(req.message) + len(reply)
    )
    elapsed_ms = int((time.time() - start) * 1000)

    return {
        "code": 200,
        "message": "ok",
        "data": {
            "reply": reply,
            "usage": usage.model_dump(),
            "elapsed_ms": elapsed_ms,
            "internal_trace": "内部调试信息，不应返回给调用方"
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

> `internal_trace` 不在 `ChatResponse` 里，`response_model` 会自动把它过滤掉。

---

## Step 4：启动并测试

```bash
uvicorn main:app --reload
```

打开 `http://127.0.0.1:8000/docs`，找到 **POST /chat**。

请求体：

```json
{
  "message": "你好",
  "model": "gpt-4o-mini",
  "temperature": 0.7,
  "stream": false
}
```

**预期响应**：

```json
{
  "code": 200,
  "message": "ok",
  "data": {
    "reply": "[gpt-4o-mini] 收到：你好（temperature=0.7, stream=False）",
    "usage": {
      "prompt_tokens": 2,
      "completion_tokens": 50,
      "total_tokens": 52
    },
    "elapsed_ms": 50
  }
}
```

> `completion_tokens` 的具体数字会变；`elapsed_ms` 大约 50 以上。**注意响应里没有 `internal_trace`**。

---

## Step 5：验证自动校验

在 `/docs` 里依次测试，全部应返回 **422**：

| 测试 | 请求体 | 错误位置 |
|------|--------|---------|
| 缺少 message | `{"model": "gpt-4o-mini"}` | `["body","message"]` |
| message 为空 | `{"message": ""}` | `["body","message"]` |
| 温度越界 | `{"message": "你好", "temperature": 3.0}` | `["body","temperature"]` |
| 类型错误 | `{"message": "你好", "temperature": "abc"}` | `["body","temperature"]` |
| 消息超长 | message 填 2001 个字符 | `["body","message"]` |
| stream 类型错 | `{"message": "你好", "stream": "abc"}` | `["body","stream"]` |

**422 响应结构**（示例）：

```json
{
  "detail": [
    {
      "type": "less_than_equal",
      "loc": ["body", "temperature"],
      "msg": "Input should be less than or equal to 2",
      "input": 3.0
    }
  ]
}
```

> 函数体里**一行校验代码都没有**——全部由 Pydantic 在进入函数前完成。

---

## Step 6：验证字段过滤

看响应 JSON：

- 有 `reply`、`usage`、`elapsed_ms`
- **没有 `internal_trace`**

把 `response_model` 临时去掉再请求一次，`internal_trace` 就会出现——对比一下，理解过滤效果。

---

## 加餐：再加一个 PUT 配置接口（可选）

```python
class AgentConfig(BaseModel):
    model: str = Field("gpt-4o-mini", description="模型名称")
    temperature: float = Field(0.7, ge=0.0, le=2.0)
    max_tokens: int = Field(1024, ge=1, le=8192)

@app.put(
    "/agents/{agent_id}/config",
    response_model=ApiResponse[AgentConfig],
    summary="更新 Agent 配置",
    tags=["agent"]
)
def update_config(agent_id: int, config: AgentConfig):
    return ApiResponse(data=config)
```

测试：

```
PUT /agents/1001/config
{"temperature": 0.3, "max_tokens": 2048}
```

预期响应里 `agent_id` 不会出现（因为 `AgentConfig` 没有这个字段），但接口逻辑里可以拿到路径参数。

---

## 完整代码

<details>
<summary>点开查看完整 main.py</summary>

```python
import time
from typing import Generic, TypeVar

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(
    title="Agent Chat API",
    description="Pydantic v2 结构化请求/响应示例",
    version="0.1.0"
)

class ChatRequest(BaseModel):
    message: str = Field(..., min_length=1, max_length=2000,
                         description="用户消息")
    model: str = Field("gpt-4o-mini", description="模型名称")
    temperature: float = Field(0.7, ge=0.0, le=2.0,
                               description="温度，0~2")
    stream: bool = Field(False, description="是否流式输出")

class TokenUsage(BaseModel):
    prompt_tokens: int = Field(..., ge=0)
    completion_tokens: int = Field(..., ge=0)
    total_tokens: int = Field(..., ge=0)

class ChatResponse(BaseModel):
    reply: str = Field(..., description="回复内容")
    usage: TokenUsage = Field(..., description="Token 消耗")
    elapsed_ms: int = Field(..., ge=0, description="请求耗时（毫秒）")

T = TypeVar("T")

class ApiResponse(BaseModel, Generic[T]):
    code: int = 200
    message: str = "ok"
    data: T | None = None

@app.post("/chat", response_model=ApiResponse[ChatResponse],
          summary="Agent 对话", tags=["chat"])
def chat(req: ChatRequest):
    """接收结构化对话请求，返回模拟回复和 token 消耗。"""
    start = time.time()
    time.sleep(0.05)

    reply = (
        f"[{req.model}] 收到：{req.message}"
        f"（temperature={req.temperature}, stream={req.stream}）"
    )

    usage = TokenUsage(
        prompt_tokens=len(req.message),
        completion_tokens=len(reply),
        total_tokens=len(req.message) + len(reply)
    )
    elapsed_ms = int((time.time() - start) * 1000)

    return {
        "code": 200,
        "message": "ok",
        "data": {
            "reply": reply,
            "usage": usage.model_dump(),
            "elapsed_ms": elapsed_ms,
            "internal_trace": "内部调试信息，不应返回给调用方"
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

</details>

---

## 验证标准

- [ ] POST /chat 能接收 JSON 请求体
- [ ] 响应是 `{code, message, data}` 结构
- [ ] data 里有 reply、usage、elapsed_ms
- [ ] 响应里没有 internal_trace（字段过滤生效）
- [ ] 缺少 message 返回 422
- [ ] 空 message 返回 422
- [ ] temperature=3.0 返回 422
- [ ] temperature="abc" 返回 422
- [ ] 2001 字符的消息返回 422
- [ ] `/docs` 里能看到请求体模板和响应结构
- [ ] 能说出 `ApiResponse[ChatResponse]` 的含义
- [ ] 能说出 Pydantic 模型和 Java 实体类 + JSR380 的对应关系

---

## 练习任务

1. **改嵌套请求**：把 `message: str` 改成 `messages: list[Message]`（`Message` 含 `role` 和 `content`），用多轮对话测试。
2. **加工具列表**：给请求加 `tools: list[ToolCall] = []`，`ToolCall` 含 `name` 和 `arguments`。
3. **加配置接口**：完成上面的 PUT `/agents/{agent_id}/config`，测试温度越界返回 422。
4. **加响应字段**：给 `ChatResponse` 加 `model: str`，确认响应里出现。
5. **思考**：
   - 为什么 `internal_trace` 不会返回？
   - 如果去掉 `response_model`，返回结构会怎样？
   - `Field(..., ge=0, le=2)` 里 `...` 和 `ge/le` 各起什么作用？

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 422 `missing` | 请求体缺必填字段 | 看 `loc` 补字段 |
| 422 `float_parsing` | 类型不对 | 检查 JSON 值类型 |
| 422 `less_than_equal` | 超出 `le` 限制 | 调整值或上限 |
| 响应里多出内部字段 | 没写 response_model 或模型里定义了该字段 | 检查 `response_model` |
| 响应里少了字段 | 响应模型没定义该字段 | 加到 ChatResponse |
| `ApiResponse[ChatResponse]` 报错 | Pydantic 版本或泛型写法问题 | 确认 pydantic v2，`from typing import Generic, TypeVar` |
| 请求体不生效 | 用了 GET 或没设 Content-Type | 用 POST/PUT；`/docs` 会自动设 JSON |
| 改了代码没反应 | 没加 `--reload` | 重启 uvicorn |

---

*上一节理论：[06.4 response_model 与统一返回](../theory/06.4-response-model-and-result.md) | 下一步：待规划*
