# 模块 6：Pydantic v2 与结构化接口

> **学习目标**：用 Pydantic v2 定义请求/响应模型，写 POST/PUT 接口，自动校验 JSON 并返回统一结果。这是 Agent 接口的基础——对话请求、模型配置、工具参数都靠结构化模型传递。

---

## 学习顺序

| 顺序 | 文档 | 预计时间 | 核心内容 |
|------|------|---------|---------|
| 1 | [06.1 Pydantic 基础](theory/06.1-pydantic-basics.md) | 2.5 小时 | BaseModel、字段类型、Field 校验、可选字段、Enum |
| 2 | [06.2 嵌套模型](theory/06.2-nested-models.md) | 2 小时 | 嵌套模型、模型列表、Agent 工具调用/对话历史结构 |
| 3 | [06.3 POST / PUT 与自动校验](theory/06.3-post-put-and-validation.md) | 2 小时 | JSON 请求体映射、422 错误结构 |
| 4 | [06.4 response_model 与统一返回](theory/06.4-response-model-and-result.md) | 2 小时 | 响应模型、字段过滤、Result<T> 封装 |

---

## 实操

| 实操 | 内容 |
|------|------|
| [实操 06.1 Agent 对话接口](practice/practice-06.1-agent-chat-api.md) | 请求模型（消息/模型/温度/流式）+ 响应模型（回复/token/耗时）+ POST 接口 + /docs 校验 |

---

## 建议安排（2 天）

```
第 1 天：06.1 + 06.2 + 06.3
  - BaseModel、Field 校验、嵌套模型必须亲手写
  - POST 请求体和 422 错误要跑一遍

第 2 天：06.4 + 实操 06.1
  - response_model 和 Result<T> 封装
  - 完成 Agent 对话接口
```

---

## 自检清单

- [ ] 会定义 BaseModel，标注 str/int/float/bool/list
- [ ] 知道必填字段和可选字段的区别
- [ ] 会用 Field 设置默认值、描述、长度/范围/正则校验
- [ ] 会用 Enum 限制取值
- [ ] 会定义嵌套模型和模型列表
- [ ] 会写 POST 接口接收 JSON 请求体
- [ ] 知道校验失败返回 422，能看懂 `loc` 和 `msg`
- [ ] 会用 response_model 定义响应结构
- [ ] 知道 response_model 能过滤内部字段
- [ ] 会写统一的 `ApiResponse[T]`
- [ ] 能说出 Pydantic 和 Java 实体类 + JSR380 的对应关系

---

## 说明

- 本模块使用 **Pydantic v2**，方法名是 `model_dump()` / `model_validate()`，不是 v1 的 `dict()` / `parse_obj()`。
- 流式输出、真实大模型调用、异步等后续再展开；本模块只做结构化接口闭环。

---

*返回：[Python 学习总览](../README.md)*
