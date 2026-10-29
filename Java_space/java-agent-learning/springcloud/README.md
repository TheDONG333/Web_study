# Spring Cloud 微服务学习

> 本目录独立于 `chapters/`，专门存放微服务相关文档。后续所有微服务模块都放在这里。

---

## 技术选型（本学习路线统一）

| 项 | 版本 | 说明 |
|----|------|------|
| JDK | 26 | 你的环境 |
| Spring Boot | **3.5.x** | 与前面章节保持一致 |
| Spring Cloud | **2025.0.x** | 与 Boot 3.5.x 对应（见模块 1 的版本文档） |
| Spring Cloud Alibaba | 与 Spring Cloud 2025.0.x 对应的版本 | 以官方版本说明为准 |

> **原则**：先定 Spring Cloud 版本，再反推 Spring Boot 版本，最后选 Alibaba 版本。不要反过来。

---

## 明确不学（已淘汰，任何模块都不生成）

| 领域 | 淘汰组件 |
|------|---------|
| 服务注册 | Eureka |
| 负载均衡 | Ribbon |
| 熔断降级 | Hystrix |
| 网关 | Zuul 1.x / 2.x |
| 配置中心 | Spring Cloud Config |
| 链路追踪 | Spring Cloud Sleuth |
| 配套监控 | Hystrix Dashboard、Turbine |

> 这些组件在文档中最多一句"已淘汰，不学"带过，不展开原理、配置和代码。

---

## 模块索引

| 模块 | 主题 | 文档 |
|------|------|------|
| 模块 1 | 微服务基础认知 | [01-microservices-basics/README.md](01-microservices-basics/README.md) |

> 后续模块按你的学习大纲逐步添加，不提前占位。

---

## 学习原则

1. **先搞懂问题，再学组件**——每个组件都是为了解决一个具体问题，不盲目堆技术栈。
2. **对照单体项目理解**——你写过的 SSM/Spring Boot 项目就是最好的参照物。
3. **版本先对齐，再写代码**——微服务 90% 的启动报错来自版本不匹配。
