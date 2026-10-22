# 实操 11.1：下单异步发通知（两个接口跑通 Spring AMQP）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：监听不触发时，把启动日志和 MQ 后台截图发给 AI 帮你定位

---

## 前置要求

- ✅ 已读完理论 11.1 ~ 11.2
- ✅ 已完成实操 [10.1 Docker 部署 RabbitMQ](../10-rabbitmq/practice/practice-10.1-docker-rabbitmq-console.md)，容器名为 `rabbitmq`，账号 `admin / admin123`
- ✅ 需要：JDK 26、Maven、IDEA、Docker

## 案例目标

实现"**下单后异步发通知**"的最小闭环，只写**两个接口**：

| # | 接口 | 作用 |
|---|------|------|
| ① | POST `/api/order/create` | 创建订单 → 发送消息到 MQ（不等待通知） |
| ② | GET `/api/order/notify` | 查询已消费的通知列表（验证消费者确实干活了） |

消费动作由 `@RabbitListener` 在后台自动完成——**接口 ① 返回时，通知可能还没消费完，这正是异步**。

---

## 项目结构（7 个文件，全部自己敲）

```
order-notify-demo/
├── pom.xml
└── src/main/
    ├── java/com/demo/
    │   ├── OrderNotifyApplication.java      ← 启动类
    │   ├── config/RabbitConfig.java          ← 交换机/队列/绑定 + Jackson 转换器
    │   ├── dto/OrderMessage.java             ← 消息体
    │   ├── service/OrderService.java         ← RabbitTemplate 发送
    │   ├── listener/OrderNotifyListener.java ← @RabbitListener 消费
    │   └── controller/OrderController.java   ← 两个接口
    └── resources/
        └── application.yml
```

> 包结构必须都在启动类子包下（08.1 的扫描规则），否则监听器不会被注册。

---

## Step 1：启动 RabbitMQ 并确认

```bash
docker ps                      # 看 rabbitmq 容器是否在跑
```

如果没在跑：

```bash
docker start rabbitmq
```

> 容器没了（被 `docker rm` 过）就重新执行第 10 章实操 Step 1 的 `docker run` 命令。

浏览器打开 http://localhost:15672，用 `admin / admin123` 能登录即可。

---

## Step 2：pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.15</version>
    </parent>

    <groupId>com.demo</groupId>
    <artifactId>order-notify-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>26</java.version>
    </properties>

    <dependencies>
        <!-- web：提供两个 HTTP 接口 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- RabbitMQ：RabbitTemplate + @RabbitListener + Jackson 转换器 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## Step 3：application.yml

```yaml
server:
  port: 8080

spring:
  rabbitmq:
    host: localhost
    port: 5672                  # ⚠️ 不是 15672
    virtual-host: /
    username: admin
    password: admin123
```

> 账号密码要和第 10 章 `docker run` 时设的一致。

---

## Step 4：消息体 DTO

时间字段用 `String`，避免 Jackson 对 `LocalDateTime` 报错（理论 11.2 的坑二）。

```java
package com.demo.dto;

public class OrderMessage {
    private String orderId;
    private String username;
    private Double amount;
    private String createTime;

    public OrderMessage() {}      // 反序列化必须有空构造

    public OrderMessage(String orderId, String username, Double amount, String createTime) {
        this.orderId = orderId;
        this.username = username;
        this.amount = amount;
        this.createTime = createTime;
    }

    // ── getter / setter（Jackson 靠它们读写字段，全部补全）──
    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public Double getAmount() { return amount; }
    public void setAmount(Double amount) { this.amount = amount; }
    public String getCreateTime() { return createTime; }
    public void setCreateTime(String createTime) { this.createTime = createTime; }
}
```

---

## Step 5：RabbitConfig（声明基础设施 + Jackson 转换器）

```java
package com.demo.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    // ① 直连交换机
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("ex.order");
    }

    // ② 持久化队列
    @Bean
    public Queue orderQueue() {
        return new Queue("q.order.notify", true);
    }

    // ③ 绑定：ex.order ──(order.create)──▶ q.order.notify
    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
                .to(orderExchange())
                .with("order.create");
    }

    // ④ 用 JSON 序列化替代默认 JDK 序列化（11.2）
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

> 应用启动时，这三样会自动出现在管理后台——不用再手动点。

---

## Step 6：Service（发送消息）

```java
package com.demo.service;

import com.demo.dto.OrderMessage;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

@Service
public class OrderService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void createOrder(String orderId, String username, Double amount) {
        OrderMessage message = new OrderMessage(
                orderId, username, amount, LocalDateTime.now().toString());

        // 参数：交换机、路由键、消息体
        rabbitTemplate.convertAndSend("ex.order", "order.create", message);

        // 这行先打印，说明"发送"完成——但消费者可能还没开始处理
        System.out.println("【发送】订单已创建，消息已发出: " + orderId);
    }
}
```

---

## Step 7：Listener（自动消费）

```java
package com.demo.listener;

import com.demo.dto.OrderMessage;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;

@Component
public class OrderNotifyListener {

    /** 消费记录：用并发安全的队列，HTTP 线程和监听线程都会访问它 */
    private final Queue<String> noticeRecords = new ConcurrentLinkedQueue<>();

    @RabbitListener(queues = "q.order.notify")
    public void handleOrderCreated(OrderMessage message) {
        // 模拟发短信/发邮件（真实项目在这里调短信服务）
        String record = String.format(
                "通知已发送给 %s：您的订单 %s（%.2f 元）创建成功",
                message.getUsername(), message.getOrderId(), message.getAmount());

        System.out.println("【消费】" + record);
        noticeRecords.offer(record);      // 存起来供接口 ② 查询
    }

    /** 供 Controller 查询消费结果 */
    public Queue<String> getNoticeRecords() {
        return noticeRecords;
    }
}
```

> **两条线程**：Tomcat 线程处理 HTTP 请求，监听容器线程处理 MQ 消息。所以消费记录要用并发安全集合，不能随手用 `ArrayList`（回忆第二章）。

---

## Step 8：Controller（两个接口）

```java
package com.demo.controller;

import com.demo.listener.OrderNotifyListener;
import com.demo.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/order")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderNotifyListener notifyListener;

    // ① 创建订单：发送消息后立即返回
    @PostMapping("/create")
    public Map<String, Object> create(@RequestParam String orderId,
                                      @RequestParam String username,
                                      @RequestParam Double amount) {
        orderService.createOrder(orderId, username, amount);
        return Map.of("success", true,
                "message", "下单成功，通知将异步发送");
    }

    // ② 查看已消费的通知
    @GetMapping("/notify")
    public Map<String, Object> notifyList() {
        List<String> list = List.copyOf(notifyListener.getNoticeRecords());
        return Map.of("count", list.size(), "records", list);
    }
}
```

---

## Step 9：启动类

```java
package com.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderNotifyApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderNotifyApplication.class, args);
    }
}
```

---

## Step 10：跑通全流程（按顺序做）

### 10.1 启动应用并确认基础设施

1. 右键 `OrderNotifyApplication` → Run
2. 控制台无报错，看到 Tomcat started on port 8080
3. 打开管理后台 http://localhost:15672：
   - **Exchanges** 里有 `ex.order`
   - **Queues** 里有 `q.order.notify`
   - 点 `ex.order` → Bindings 里有一条 `order.create → q.order.notify`

> 这三样你一行命令都没在后台点，全是配置类的 `@Bean` 声明出来的。

### 10.2 调接口 ① 发送消息

Postman 或浏览器（POST 建议用 Postman）：

```
POST http://localhost:8080/api/order/create?orderId=1001&username=tom&amount=99.9
```

预期返回：

```json
{"success":true,"message":"下单成功，通知将异步发送"}
```

**紧接着看应用控制台**（可能几乎同时，也可能晚几十毫秒）：

```
【发送】订单已创建，消息已发出: 1001
【消费】通知已发送给 tom：您的订单 1001（99.90 元）创建成功
```

### 10.3 调接口 ② 查看消费结果

```
GET http://localhost:8080/api/order/notify
```

预期返回：

```json
{"count":1,"records":["通知已发送给 tom：您的订单 1001（99.90 元）创建成功"]}
```

> 能查到记录 = 消费者真的处理了。再调一次 ①（换个 orderId），再查 ②，count 变成 2。

### 10.4 验证序列化（理论 11.2 的效果）

在管理后台 **Queues** → `q.order.notify` → 点开消息 Payload：

- 配了 Jackson 转换器：能看到 `{"orderId":"1001",...}` 这样的 JSON
- 把 `messageConverter()` 这个 Bean 注释掉重启再发一条：Payload 变成 `rO0ABXNy...` 乱码

**亲眼对比一次**，这个知识点就再也忘不掉。看完记得把 Bean 加回来。

### 10.5 验证"异步解耦"（最有说服力的一步）

1. **停掉应用**（点 IDEA 的红色 Stop）
2. 在管理后台 `ex.order` → Publish message，Routing key 填 `order.create`，Payload 填：

```json
{"orderId":"2002","username":"jerry","amount":50.0,"createTime":"2026-10-22T10:00:00"}
```

3. 去 Queues 看：`q.order.notify` 的 **Ready = 1**（消息在队列里等着）
4. **重新启动应用**：几秒内 Ready 变 0，控制台打印 `【消费】... 2002 ...`
5. GET `/api/order/notify`：记录里多了 jerry 的订单

> 应用停着的时候，订单消息**没有丢**——这就是 MQ"削峰/解耦"的直观体现：发送方和消费方互不等待。

---

## 验证标准

- [ ] 应用启动后，`ex.order`、`q.order.notify`、绑定关系自动出现在管理后台
- [ ] 接口 ① 返回成功，控制台先打印【发送】再打印【消费】
- [ ] 接口 ② 能查到消费记录，多次下单记录累加
- [ ] 管理后台看到的 Payload 是可读 JSON（不是 `rO0AB...`）
- [ ] 应用停止时发消息，队列 Ready 增加；重启后消息被消费
- [ ] 能用一句话说清：为什么接口 ① 返回时通知可能还没发出？（异步）
- [ ] 能指出代码里哪个注解是消费者、哪个类是生产者

---

## 练习任务

1. **加第二个通知渠道**：新建队列 `q.order.email`，用同一个交换机 `ex.order`、同一个路由键 `order.create` 绑定；再写一个 `@RabbitListener` 打印"邮件已发送"。观察：一条消息是不是**两个队列都收到**了？（回忆 10.3：Direct 同键多队列会复制）
2. **换个路由键试试**：把 Service 里的路由键改成 `order.pay`（但绑定没改），发消息，观察队列 Ready 是否一直是 0——亲自验证"无匹配即丢弃"。
3. **思考并回答**：
   - `@RabbitListener` 的方法如果抛异常，消息会怎样？（提示：默认重新入队，可能无限重试）
   - 为什么监听器里存记录要用 `ConcurrentLinkedQueue`？换成 `ArrayList` 可能出什么问题？
   - 如果消费者服务临时下线，消息会丢吗？为什么？
4. **（选做）改成 Fanout**：把 `DirectExchange` 换成 `FanoutExchange`，绑定时不写路由键，加两个通知队列，体验"一次下单，短信和邮件都收到"。

---

## 常见报错排查

| 报错 | 原因 | 解决 |
|------|------|------|
| `Connection refused: localhost:5672` | RabbitMQ 没启动 | `docker start rabbitmq` |
| `ACCESS_REFUSED - Login was refused` | 账号密码/vhost 不对 | 检查 yml 与 `docker run` 时的参数 |
| 启动报 `PRECONDITION_FAILED - inequivalent arg` | 后台已有同名队列但参数不同 | 在管理后台删掉该队列，重启应用 |
| 监听器不触发 | 类不在启动类子包 / 队列名写错 | 检查包结构、`@RabbitListener(queues=...)` |
| Payload 是 `rO0AB...` 乱码 | 没配 Jackson 转换器 | 加上 `messageConverter()` Bean |
| 消费时报转换异常 | 收发转换器不一致 / DTO 缺空构造或 setter | 统一用同一个 Bean，补全 DTO |
| 接口 404 | Controller 不在子包 | 回忆 08.1 的扫描规则 |
| 8080 被占用 | 上个项目没停 | 改 `server.port` 或停掉旧进程 |

---

*上一节理论：[11.2 消息序列化](../theory/11.2-message-serialization.md) | 下一章：[12-Elasticsearch（搜索）](../../12-elasticsearch/theory/12.1-elasticsearch-quickstart.md)*
