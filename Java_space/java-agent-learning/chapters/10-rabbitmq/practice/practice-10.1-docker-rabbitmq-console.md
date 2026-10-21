# 实操 10.1：Docker 部署 RabbitMQ 4.x + 管理后台手动玩转三种交换机

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：页面找不到某个按钮时，让 AI 按 RabbitMQ 4.x 管理后台的界面帮你定位

---

## 前置要求

- ✅ 已读完理论 10.1 ~ 10.4
- ✅ 已安装 **Docker Desktop**（Windows/Mac 均可），并已启动（任务栏鲸鱼图标不转圈）
- ✅ 不需要写任何 Java 代码——本次全程在浏览器里点

## 实践目标

1. 用 Docker 一条命令跑起 RabbitMQ 4.x（自带管理后台）
2. 认识管理后台的各个标签页
3. **亲手验证三种交换机的投递规则**：
   - Direct：路由键精确匹配才进队列
   - Fanout：不看路由键，全部广播
   - Topic：`*` 和 `#` 通配符怎么匹配
4. 学会"发消息 → 看队列 → 取消息"的完整调试流程

> 这次实操就是把理论 10.2~10.4 的图，在真实界面上走一遍。**做完你就能向别人讲清 MQ 模型。**

---

## Step 1：一条命令启动 RabbitMQ 4.x

打开终端（Windows 用 PowerShell 或 Git Bash），执行：

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  rabbitmq:4-management
```

**参数逐条解释**：

| 参数 | 作用 | 白话 |
|------|------|------|
| `-d` | 后台运行 | 关掉终端也不停 |
| `--name rabbitmq` | 容器命名 | 以后 `docker stop rabbitmq` 就能停 |
| `-p 5672:5672` | AMQP 协议端口 | **程序**连 RabbitMQ 用这个（后续写 Java 用） |
| `-p 15672:15672` | 管理后台端口 | **浏览器**访问用这个 |
| `RABBITMQ_DEFAULT_USER/PASS` | 初始账号密码 | 不用默认的 guest，避免远程登录限制 |
| `rabbitmq:4-management` | 镜像名 | `-management` 表示**自带管理后台插件** |

> ⚠️ 这里用的是 **4.x 版本**（你要求的），镜像自带 Erlang 运行时，Windows 不需要额外安装。

### 确认启动成功

```bash
docker ps                    # 能看到 rabbitmq 容器在运行
docker logs rabbitmq         # 等几秒，看到 "Server startup complete" 就好了
```

> 如果日志里出现 `Server startup complete`，说明 MQ 已就绪。首次启动要 10~30 秒，别急。

---

## Step 2：打开管理后台，认一遍界面

浏览器访问：**http://localhost:15672**

用刚才设的账号登录：`admin` / `admin123`

登录后逐一看这几个标签页（对应理论 10.2 的概念）：

| 标签页 | 看什么 | 对应的理论概念 |
|--------|--------|---------------|
| **Overview** | 消息总数、连接数、队列数 | 全局状态 |
| **Connections** | 谁连着 MQ | Connection |
| **Channels** | 连接里的通道 | Channel |
| **Exchanges** | 所有交换机（能看到类型） | 交换机 |
| **Queues and Streams** | 所有队列、消息堆积数 | 队列 |
| **Admin** | 用户、vhost | vhost |

> 点开 **Exchanges** 标签，你会看到 7 个系统自带的交换机（`amq.direct`、`amq.fanout`、`amq.topic` 等）——这些是 RabbitMQ 内置的，**我们自己建新的就行，别动它们**。

---

## Step 3：Direct 实验——验证"精确匹配"

### 3.1 创建直连交换机

**Exchanges → Add a new exchange**：

| 字段 | 填什么 |
|------|--------|
| Name | `ex.direct.order` |
| Type | `direct` |
| Durability | `Durable`（持久化，重启还在） |
| Auto delete | `No` |

点 **Add exchange**。

### 3.2 创建两个队列

**Queues and Streams → Add a new queue**，建两个：

| Name | Type |
|------|------|
| `q.order.create` | Default (Classic) |
| `q.order.pay` | Default (Classic) |

> 队列名和交换机名都能自定义，但**名字有意义**会让你后续排错轻松很多。

### 3.3 绑定（关键一步！）

点 `ex.direct.order` 进入详情 → **Bindings** 区域：

- 绑定 1：To queue = `q.order.create`，Routing key = `order.create` → Bind
- 绑定 2：To queue = `q.order.pay`，Routing key = `order.pay` → Bind

绑定完，Bindings 表格里应该能看到两行。

### 3.4 发消息验证

点 `ex.direct.order` → **Publish message**：

**测试 A：routing key = `order.create`**

| 字段 | 值 |
|------|-----|
| Routing key | `order.create` |
| Payload | `{"orderId":1001,"event":"created"}` |
| Delivery mode | `Persistent (2)` |

点 **Publish message**，然后去 **Queues** 页看：

- `q.order.create` 的 Ready 数变成 **1** ✅
- `q.order.pay` 仍然是 **0** ✅

**测试 B：改成 `order.cancel`（没有对应绑定）再发**

- 两个队列的 Ready 都还是 **0** ——消息**被丢弃**了 ✅（这就是理论 10.3 说的"无匹配即丢弃"）

> 💡 **亲身踩一次"消息消失"**，比看十遍文档都记得牢。

---

## Step 4：Fanout 实验——验证"全员广播"

### 4.1 创建 Fanout 交换机

**Exchanges → Add a new exchange**：Name = `ex.fanout.notify`，Type = `fanout`，Durable。

### 4.2 创建三个队列

`q.inventory`、`q.points`、`q.email`

### 4.3 绑定（Fanout 的绑定不写路由键）

点 `ex.fanout.notify` → Bindings：

依次把三个队列绑上，**Routing key 留空**（Fanout 不需要）。

### 4.4 发消息验证

Publish message：Routing key 随便写（比如 `anything`），Payload 随便填，发送。

去 Queues 页看：**三个队列的 Ready 全部变成 1** ✅

> 再发一条、换一个 routing key 发——结果一样。**Fanout 根本不看路由键**，只要绑定了就收。

---

## Step 5：Topic 实验——验证 `*` 和 `#`

### 5.1 创建 Topic 交换机

**Exchanges → Add a new exchange**：Name = `ex.topic.log`，Type = `topic`，Durable。

### 5.2 创建三个队列

`q.audit`、`q.alert`、`q.stat`

### 5.3 绑定三条规则（重点）

点 `ex.topic.log` → Bindings，依次添加：

| To queue | Routing key | 含义 |
|----------|------------|------|
| `q.audit` | `order.#` | 所有 order 开头的日志（零段或多段） |
| `q.alert` | `#.error` | 所有以 error 结尾的日志 |
| `q.stat` | `order.*.create` | 订单创建日志（中间恰好一段） |

### 5.4 发四条消息，填下面表格验证

在 `ex.topic.log` 的 Publish message 里，**依次**用这些 routing key 发送，每发一条就去 Queues 页记录结果（收到记 ✅，没收到记 ❌）：

| # | Routing key | q.audit | q.alert | q.stat |
|---|-------------|---------|---------|--------|
| 1 | `order.pc.create` | ？ | ？ | ？ |
| 2 | `order.pc.error` | ？ | ？ | ？ |
| 3 | `user.app.error` | ？ | ？ | ？ |
| 4 | `order.create` | ？ | ？ | ？ |

**对照答案**（先自己填完再看）：

| # | Routing key | q.audit | q.alert | q.stat | 为什么 |
|---|-------------|---------|---------|--------|--------|
| 1 | `order.pc.create` | ✅ | ❌ | ✅ | `order.#` 匹配；`order.*.create` 匹配 |
| 2 | `order.pc.error` | ✅ | ✅ | ❌ | `#.error` 匹配；create 规则不匹配 |
| 3 | `user.app.error` | ❌ | ✅ | ❌ | 只有 `#.error` 匹配 |
| 4 | `order.create` | ✅ | ❌ | ❌ | `order.#` 的 `#` 可匹配 0 段；`*` 需要 3 段不够 |

> 如果第 4 条你答错了，回去重看理论 10.4 的 `#` 匹配零段——这是最容易被忽略的点。

---

## Step 6：取消息，理解 "Ready / Unacked"

进入 **Queues and Streams** → 点某个队列（比如 `q.order.create`）→ 展开 **Get messages**：

| 字段 | 选项 | 说明 |
|------|------|------|
| Messages | 1 | 取几条 |
| Ack Mode | `Nack message requeue true` | 取出来看看，**看完放回队列**（消息还在） |
| Ack Mode | `Automatic ack` | 取出来就算消费了（**消息消失**） |

先选 `Nack message requeue true` 点 **Get message(s)**：消息内容显示出来了，刷新页面发现 Ready 数**没变**（放回去了）。

再选 `Automatic ack` 点一次：Ready 数**减 1**（真消费掉了）。

> **这就是 ack 机制**：消费者处理完要"签收"，没签收的消息在消费者断开后会重新投递。理论 10.2 的第 5 节在这里落地。

---

## 验证标准

- [ ] `docker ps` 能看到 rabbitmq 容器运行中，管理后台能登录
- [ ] 认识 Exchanges / Queues / Bindings 三个关键页面
- [ ] Direct：`order.create` 只进 `q.order.create`；`order.cancel` 两个队列都收不到
- [ ] Fanout：任意 routing key 发送，三个队列全部收到
- [ ] Topic：四条消息的匹配结果和答案表一致（尤其第 4 条）
- [ ] 能说出 `*` 和 `#` 的区别（各用一句话）
- [ ] 理解 Nack requeue 和 Automatic ack 的区别（消息留不留下）
- [ ] 能说出生产者、交换机、队列、消费者、路由键、绑定分别在哪一步出现

---

## 练习任务

1. **改绑定做实验**：把 `q.stat` 的绑定从 `order.*.create` 改成 `order.#`，重发四条消息，观察结果变化。
2. **设计路由键**：如果要做"短信系统"，支持按业务（验证码/营销/通知）和按运营商（移动/联通/电信）订阅，你会怎么设计路由键和绑定？写下来，再让 AI 点评。
3. **思考并回答**：
   - 为什么 Direct 交换机下，同一个路由键绑两个队列会"两个都收到"，而不是"只有一个收到"？
   - Fanout 交换机能不能替代 Direct？什么情况下不能？
   - 一条消息没有匹配任何队列时去了哪里？（答：被交换机丢弃，除非做特殊配置）
4. **清理环境（可选）**：

```bash
docker stop rabbitmq      # 停止
docker start rabbitmq     # 再启动（数据还在，因为容器没删）
docker rm -f rabbitmq     # 删除容器（数据全没，下次重新 run）
```

> 本次没挂载数据卷，删容器 = 数据清空。学习阶段没问题；真实项目要加 `-v rabbitmq-data:/var/lib/rabbitmq` 做持久化。

---

## 常见报错排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `docker: command not found` | 没装 Docker 或没启动 | 安装并启动 Docker Desktop |
| `port is already allocated` | 5672/15672 被占用 | `docker ps` 看是否有旧容器；或改端口 `-p 15673:15672` |
| 浏览器打不开 15672 | 容器还在启动中 | 等 20 秒；`docker logs rabbitmq` 看是否 startup complete |
| 登录失败 | 账号密码不是 admin/admin123 | 检查 run 命令的 `-e` 参数；或重新 run |
| 消息发了 Ready 是 0 | 路由键没匹配 / 没绑定 / 发错交换机 | 按 10.3 的误区自查顺序检查 |
| 队列里消息取不出来 | 用了 Nack requeue | 改 Automatic ack 再取 |
| 容器启动后立刻退出 | 内存不足 / Docker 资源限制 | 给 Docker Desktop 分配至少 2GB 内存 |

---

*上一节理论：[10.4 Topic 交换机](../theory/10.4-topic-exchange.md) | 下一步：RabbitMQ Java 集成（Spring AMQP，待规划）*
