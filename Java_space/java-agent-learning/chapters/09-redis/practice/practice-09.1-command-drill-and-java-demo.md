# 实操 09.1：五大类型命令速练 + Java demo

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：命令敲不出来时问 AI"用 redis 命令实现 XXX"，对照理解

---

## 前置要求

- ✅ 已读完理论 09.1 ~ 09.6
- ✅ 已安装 Redis（Windows 免安装版或 Docker），并且服务已启动
- ✅ 会创建 Spring Boot 项目（08 章实操练过）

## 练习目标

1. 把 5 种类型的核心命令**亲手敲一遍**（这部分别偷懒，敲过才有手感）
2. 建一个 Spring Boot 项目，用 StringRedisTemplate 给每个类型写一个 demo 接口
3. 建立起"**命令 ↔ Java API**"的对应感——以后写代码就是回忆命令

---

## Step 1：启动 Redis，连上命令行

1. 双击 `redis-server.exe`（或 `docker start my-redis`），**窗口保持开着**
2. 另开窗口双击 `redis-cli.exe`，输入 `ping` 看到 `PONG` 即成功

```bash
127.0.0.1:6379> ping
PONG
```

---

## Step 2：通用命令热身（2 分钟）

```bash
set test:key hello
exists test:key          # 1
expire test:key 30
ttl test:key             # 30 左右
del test:key
ttl test:key             # -2（不存在）
```

> 忘了 ttl 的 -1 / -2 是什么意思就翻 [09.1 理论](../theory/09.1-redis-introduction-and-quickstart.md)。

---

## Step 3：五种类型逐个敲一遍（20 分钟）

**每个类型按"命令 → 自检问题"的顺序做，做完对照理论核对。**

### ① String

```bash
mset s1 v1 s2 v2
mget s1 s2                       # 两个值都出来了吗？
set counter 10
incr counter                     # 11
decr counter                     # 10
setex code:100 60 8848           # 验证码式存法
ttl code:100                     # 还剩几秒？
```

**自检**：`incr` 一个不存在 key 会怎样？（答：按 0 算，结果是 1）

### ② Hash

```bash
hset user:100 name tom age 25 balance 100      # 一次存多个字段
hget user:100 name                             # tom
hmget user:100 name balance                    # 两个字段
hset user:100 balance 80                       # 只改余额
hgetall user:100                               # 整个对象
hdel user:100 age                              # 删一个字段
```

**自检**：`hgetall` 的返回为什么是交替出现的？（field、value、field、value…）

### ③ List

```bash
lpush msg:new "第1条" "第2条" "第3条"
lrange msg:new 0 -1              # 注意！顺序是反的（每次插左边）
rpush msg:new "第4条"            # 从右边补一条
lpop msg:new                     # 弹出一个
lrange msg:new 0 -1
```

**自检**：为什么 `lpush` 三条后顺序是 3、2、1？（做"最新消息"正需要这样）

### ④ Set

```bash
sadd friends:1 张三 李四 王五
sadd friends:2 李四 王五 赵六
sismember friends:1 张三          # 1
sinter friends:1 friends:2        # 共同好友：李四 王五
sdiff friends:1 friends:2         # 只我有：张三
sadd friends:1 张三               # 再塞一次 → 返回 0，加不进去
```

**自检**：`sadd` 重复成员返回什么？（0 = 没加进去，这就是"去重"）

### ⑤ ZSet

```bash
zadd rank:sale 980 键盘 750 显示器 320 鼠标
zrevrange rank:sale 0 -1 WITHSCORES     # 从高到低，谁是第一？
zincrby rank:sale 500 鼠标              # 鼠标 320 + 500
zrevrange rank:sale 0 0                 # 现在第一是谁？
zrevrank rank:sale 键盘                 # 键盘排第几（0 开头）
zrangebyscore rank:sale 800 1000        # 800 分以上的有哪些？
zrem rank:sale 键盘                     # 删除
```

**自检**：`zrevrank` 返回 0 代表第几名？（第 1 名——展示时要 +1）

---

## Step 4：建 Java demo 项目（30 分钟）

**Step 4.1 创建项目 `redis-demo`**（Initializr 勾 Spring Web + Spring Data Redis，或手写 pom）

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.15</version>
</parent>

<properties>
    <java.version>26</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

**Step 4.2 application.yml**

```yaml
server:
  port: 8080
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

**Step 4.3 五个 demo 接口（每类型一个）**

```java
package com.demo.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.Duration;
import java.util.*;

@RestController
public class DemoController {

    @Autowired
    private StringRedisTemplate redis;

    // ① String：存验证码 60 秒，返回值和剩余时间
    @GetMapping("/demo/string")
    public Map<String, Object> stringDemo() {
        String key = "sms:code:13800138000";
        redis.opsForValue().set(key, "8848", Duration.ofSeconds(60));
        redis.opsForValue().increment("demo:counter");     // 顺便演示计数器

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("code", redis.opsForValue().get(key));
        result.put("ttl", redis.getExpire(key));
        result.put("counter", redis.opsForValue().get("demo:counter"));
        return result;
    }

    // ② Hash：存用户对象，只改一个字段
    @GetMapping("/demo/hash")
    public Map<Object, Object> hashDemo() {
        String key = "user:info:1001";
        redis.opsForHash().put(key, "name", "tom");
        redis.opsForHash().put(key, "balance", "100");
        redis.opsForHash().put(key, "balance", "80");      // 部分字段更新
        return redis.opsForHash().entries(key);            // 取整个对象
    }

    // ③ List：最新消息列表
    @GetMapping("/demo/list")
    public List<String> listDemo() {
        String key = "msg:latest";
        redis.delete(key);
        redis.opsForList().leftPush(key, "消息A");          // 新消息插最前
        redis.opsForList().leftPush(key, "消息B");
        redis.opsForList().leftPush(key, "消息C");
        return redis.opsForList().range(key, 0, 9);        // 最近 10 条
    }

    // ④ Set：共同好友
    @GetMapping("/demo/set")
    public Map<String, Object> setDemo() {
        redis.delete("friends:1", "friends:2");
        redis.opsForSet().add("friends:1", "张三", "李四", "王五");
        redis.opsForSet().add("friends:2", "李四", "王五", "赵六");

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("共同好友", redis.opsForSet().intersect("friends:1", "friends:2"));
        result.put("只有我有", redis.opsForSet().difference("friends:1", "friends:2"));
        result.put("张三在不在", redis.opsForSet().isMember("friends:1", "张三"));
        return result;
    }

    // ⑤ ZSet：迷你排行榜
    @GetMapping("/demo/zset")
    public List<Map<String, Object>> zsetDemo() {
        String key = "rank:sale";
        redis.delete(key);
        redis.opsForZSet().add(key, "键盘", 980);
        redis.opsForZSet().add(key, "显示器", 750);
        redis.opsForZSet().add(key, "鼠标", 320);
        redis.opsForZSet().incrementScore(key, "鼠标", 500);   // 鼠标加 500 分

        List<Map<String, Object>> list = new ArrayList<>();
        // 从高到低取前 10 名（带分数）
        for (ZSetOperations.TypedTuple<String> t : redis.opsForZSet().reverseRangeWithScores(key, 0, 9)) {
            Map<String, Object> item = new LinkedHashMap<>();
            item.put("name", t.getValue());
            item.put("score", t.getScore());
            list.add(item);
        }
        return list;
    }
}
```

---

## Step 5：跑通并"两窗口对照"（10 分钟）

1. 启动项目，浏览器依次访问 5 个接口：
   `http://localhost:8080/demo/string`、`/demo/hash`、`/demo/list`、`/demo/set`、`/demo/zset`
2. **同时开着 redis-cli**，用命令对着看你刚存进去的数据：

```bash
keys demo:*
get sms:code:13800138000
hgetall user:info:1001
lrange msg:latest 0 -1
zrevrange rank:sale 0 -1 WITHSCORES
```

> **这一步是本实操的精髓**：接口返回值和 redis 里的数据对上号，你就真正理解"Java 怎么写 = 命令怎么敲"了。

---

## 验证标准

- [ ] 5 种类型的命令全部亲手敲过，每个"自检"都能答上来
- [ ] `/demo/string` 返回了验证码和剩余秒数；连点几次看 counter 递增
- [ ] `/demo/hash` 返回的 balance 是 80（部分字段更新生效）
- [ ] `/demo/list` 返回顺序是 消息C、消息B、消息A（新的在最前）
- [ ] `/demo/set` 返回共同好友 = [李四, 王五]
- [ ] `/demo/zset` 返回的榜单顺序：键盘 980、鼠标 820、显示器 750
- [ ] redis-cli 里能查到 Java 写进去的数据

---

## 常见报错排查

| 报错 | 原因 | 解决 |
|------|------|------|
| `Unable to connect to Redis` / Connection refused | Redis 服务没启动 | 启动 redis-server（窗口别关） |
| `NOAUTH Authentication required` | Redis 设了密码但没配 | yml 加 `spring.data.redis.password: 你的密码` |
| 存进去的值是一串乱码 | 用了 `RedisTemplate`（JDK 序列化） | 换 `StringRedisTemplate` |
| 端口 6379 已被占用（启动 redis-server 报错） | 已经有一个 Redis 在跑 | 直接用现成的；或 `docker ps` 看看 |
| 接口 404 | 包结构不在启动类子包 | 回忆 08.1 的扫描规则 |

---

*上一节理论：[09.7 了解即可](../theory/09.7-bitmap-hyperloglog-geo.md) | 下一步：[实操 09.2 验证码案例（重点）](practice-09.2-verification-code-case.md)*
