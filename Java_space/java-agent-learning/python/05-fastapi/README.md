# 模块 5：FastAPI 接口开发

> **学习目标**：跑通第一个 FastAPI 服务，掌握 GET 接口、路径/查询参数，熟练用 `/docs` 自动文档调试接口。

---

## 学习顺序

| 顺序 | 文档 | 预计时间 | 核心内容 |
|------|------|---------|---------|
| 1 | [05.1 环境与启动](theory/05.1-quickstart.md) | 1.5 小时 | 安装 fastapi/uvicorn、启动命令、热重载、入口文件 |
| 2 | [05.2 路由与参数](theory/05.2-routes-and-params.md) | 2 小时 | GET、路径参数、查询参数、类型校验、默认值、可选参数 |
| 3 | [05.3 自动文档](theory/05.3-auto-docs.md) | 1.5 小时 | Swagger UI、ReDoc、接口描述、参数说明 |
| 4 | [05.4 Java 概念对照](theory/05.4-spring-mapping.md) | 1 小时 | 路由 ↔ Controller、参数 ↔ @PathVariable/@RequestParam、返回值 ↔ ResponseBody |

---

## 实操

| 实操 | 内容 |
|------|------|
| [实操 05.1 模拟对话接口](practice/practice-05.1-chat-mock-api.md) | 写一个 GET `/chat` 接口，在 `/docs` 里手动调用并查看返回 |

---

## 建议安排（1~2 天）

```
第 1 天：05.1 + 05.2 + 实操 05.1
  - 装环境、跑服务、写接口、在 /docs 里调通

第 2 天：05.3 + 05.4
  - 把接口描述和参数说明补全
  - 对照 Spring 的写法建立迁移直觉
```

---

## 自检清单

- [ ] 能独立安装 fastapi 和 uvicorn
- [ ] 知道 `uvicorn main:app --reload` 每个部分的含义
- [ ] 会用 `if __name__ == "__main__"` 启动服务
- [ ] 会写 GET 接口并返回 JSON
- [ ] 会写路径参数并做类型校验
- [ ] 会写查询参数，知道默认值和可选参数的区别
- [ ] 知道参数类型不对会返回 422
- [ ] 能打开 `/docs` 并手动调用接口
- [ ] 会写 `summary`、`description`、`Query(description=...)`
- [ ] 能说出 FastAPI 和 Spring Boot 的对应关系

---

## 说明

- FastAPI 的请求体、Pydantic 模型、POST 接口、异步等，后续模块再展开；本模块只做最小可用闭环。
- 本模块所有代码都在 Python 虚拟环境中运行（回忆模块 1 的 venv）。

---

*返回：[Python 学习总览](../README.md)*
