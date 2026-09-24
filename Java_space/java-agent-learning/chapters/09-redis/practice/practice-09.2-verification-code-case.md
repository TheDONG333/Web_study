# 实操 09.2：短信验证码案例（重点）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：写完后让 AI 检查"用户能不能绕过你的验证码逻辑"

---

## 前置要求

- ✅ 已读完理论 09.2（String 四大场景）
- ✅ 已完成实操 [09.1](practice-09.1-command-drill-and-java-demo.md)，**接着它的 `redis-demo` 项目继续写**

## 案例目标

实现真实的短信验证码流程——**这是每个 App/网站都会有的功能，也是 String + 过期时间的经典落地**：

| # | 接口 | 说明 |
|---|------|------|
| ① | POST `/api/code/send?phone=13800138000` | 发送验证码（60 秒有效；60 秒内不能重复发） |
| ② | POST `/api/code/verify` body `{phone, code}` | 校验验证码（通过后立即失效） |
| ③ | GET `/api/code/ttl?phone=13800138000` | 查验证码还剩多少秒 |

**用到的 Redis 知识**（全在 09.2）：`setex` 存验证码、`ttl` 查剩余、`setnx` 防重复发送、`del` 用完即删。

---

## 设计思路（先想清楚再写代码）

```
① 发送：生成的验证码 → setex sms:code:{手机号} 60
        防重复     → setnx sms:limit:{手机号} 60（抢不到 = 60 秒内发过了）

② 校验：get sms:code:{手机号} → 比对 → 通过后 del（一次性）
                               → 不存在 = 已过期或没发过

③ 查剩余：ttl sms:code:{手机号}
```

> 对照思考：如果用 MySQL 做，要建表、写定时任务清理过期数据、还要处理并发发送——Redis 每个需求都是一条命令。

---

## Step 1：Service 层（核心，仔细看注释）

```java
package com.demo.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.security.SecureRandom;
import java.time.Duration;

@Service
public class CodeService {

    @Autowired
    private StringRedisTemplate redis;

    private static final SecureRandom RANDOM = new SecureRandom();

    /** 发送验证码：成功返回验证码（模拟短信），失败返回 null */
    public String sendCode(String phone) {
        String limitKey = "sms:limit:" + phone;

        // ① 防重复：key 不存在才抢得到（setnx），抢不到说明 60 秒内已发过
        Boolean first = redis.opsForValue()
                .setIfAbsent(limitKey, "1", Duration.ofSeconds(60));
        if (first == null || !first) {
            return null;      // 调用方去查 ttl 告诉用户还剩几秒
        }

        // ② 生成 6 位验证码（用 SecureRandom，别用 Random——验证码是安全场景）
        String code = String.format("%06d", RANDOM.nextInt(1_000_000));

        // ③ 存 60 秒，到点自动消失
        redis.opsForValue().set("sms:code:" + phone, code, Duration.ofSeconds(60));

        // ④ 模拟"发短信"（生产环境这里换成短信服务商 SDK 的调用）
        System.out.println("【模拟短信】" + phone + " 的验证码: " + code);
        return code;
    }

    /** 校验验证码：返回结果消息 */
    public String verify(String phone, String inputCode) {
        String key = "sms:code:" + phone;
        String savedCode = redis.opsForValue().get(key);

        if (savedCode == null) {
            return "验证码已过期，请重新发送";
        }
        if (!savedCode.equals(inputCode)) {
            return "验证码错误";
        }
        redis.delete(key);        // ⭐ 通过后立即删除：验证码只能用一次
        return "验证通过";
    }

    /** 查验证码剩余秒数（-2 表示不存在） */
    public Long getTtl(String phone) {
        return redis.getExpire("sms:code:" + phone);
    }

    /** 查"距离能再次发送"还剩几秒（限流 key 的剩余时间） */
    public Long getLimitTtl(String phone) {
        return redis.getExpire("sms:limit:" + phone);
    }
}
```

---

## Step 2：Controller 层

```java
package com.demo.controller;

import com.demo.service.CodeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/code")
public class CodeController {

    @Autowired
    private CodeService codeService;

    // ① 发送验证码
    @PostMapping("/send")
    public Map<String, Object> send(@RequestParam String phone) {
        String code = codeService.sendCode(phone);
        if (code == null) {
            // ⚠️ 要查的是"限流 key"剩余时间，不是验证码 key——两个 key 别搞混
            Long wait = codeService.getLimitTtl(phone);
            return Map.of("success", false, "message", "发送太频繁，请 " + wait + " 秒后再试");
        }
        // 开发模式把验证码直接返回，方便测试（生产绝不能返回！）
        return Map.of("success", true, "message", "发送成功", "devCode", code);
    }

    // ② 校验验证码
    @PostMapping("/verify")
    public Map<String, Object> verify(@RequestBody Map<String, String> body) {
        String message = codeService.verify(body.get("phone"), body.get("code"));
        return Map.of("success", "验证通过".equals(message), "message", message);
    }

    // ③ 查剩余秒数
    @GetMapping("/ttl")
    public Map<String, Object> ttl(@RequestParam String phone) {
        return Map.of("phone", phone, "ttlSeconds", codeService.getTtl(phone));
    }
}
```

> 💡 **易错点**：这里有两个 key，别混——验证码是 `sms:code:{phone}`（有效期 60 秒），限流是 `sms:limit:{phone}`（60 秒内只能发一次）。"还剩几秒能再发"查的是**限流 key**。

---

## Step 3：测试全流程（按顺序做，别跳步）

| 步骤 | 请求 | 预期结果 |
|------|------|---------|
| ① 首次发送 | POST `/api/code/send?phone=13800138000` | success=true，devCode=六位数字；**控制台打印【模拟短信】** |
| ② 立刻重发 | 再发一次同样的请求 | success=false，"发送太频繁…"（setnx 拦住了） |
| ③ 查剩余 | GET `/api/code/ttl?phone=13800138000` | ttlSeconds ≈ 55（倒计时中） |
| ④ 错误校验 | POST `/api/code/verify`，body `{"phone":"13800138000","code":"000000"}` | "验证码错误"（除非你猜中了，概率百万分之一） |
| ⑤ 正确校验 | 用控制台打印的验证码再校验 | "验证通过" |
| ⑥ 重复校验 | 把 ⑤ 的请求再发一次 | "验证码已过期，请重新发送"（已被删除，**一次性生效**） |
| ⑦ 等 60 秒 | 等 1 分钟后再 GET `/api/code/ttl` | ttlSeconds = -2（验证码消失了） |
| ⑧ 再发送 | 等限流 key 过期后再发 | 又能发出新验证码 |

> 到此为止，你完整走了一遍"**生成 → 过期 → 限流 → 校验 → 销毁**"的生命周期——这就是验证码的全部本质。

---

## 练习任务

1. **加错误次数限制**：同一个手机号验证码最多错 3 次，超了锁定 10 分钟。提示：`incr sms:fail:{phone}` + 第一次时 `expire`。
2. **加日志观察**：在 verify 里用 `System.out.println` 打印"取出的码 / 用户输入的码"，再跑一遍全流程——亲眼看到每次 Redis 里的值怎么变。
3. **思考并回答**：
   - 为什么验证通过要**立刻删除**验证码？（提示：防止同一码被反复使用 / 被暴力尝试）
   - 为什么限制 60 秒只能发一次？（提示：短信按条收费 + 防短信轰炸）
   - 生产环境"真正发短信"替换哪一行？（答：Step 1 里 `System.out.println` 那一行）

---

## 验证标准

- [ ] 验证码 60 秒自动失效（ttl 变 -2）
- [ ] 60 秒内重复发送被拦截，提示剩余等待秒数
- [ ] 验证码错误时不会删除原码（用户还有机会再试）
- [ ] 验证通过后同一个码再用就失效了
- [ ] 说得出 setex / setnx / del / ttl 各自在流程中的角色

---

## 常见报错排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 保存后再查 ttl 是 -1 | 存的时候没用 Duration 参数 | 用 `set(key, value, Duration.ofSeconds(60))` |
| `Map.of(...)` 报空指针 | Map.of 的 value 不允许 null | `devCode` 为空时改放空字符串 |
| 重发一直成功（没拦住） | setIfAbsent 的参数写错/没设过期 | 检查 `setIfAbsent(key, "1", Duration.ofSeconds(60))` |
| 校验总说已过期 | 手机号前后不一致（空格/区号） | 统一用同一个号码测试 |

---

*上一节实操：[09.1 命令速练 + Java demo](practice-09.1-command-drill-and-java-demo.md) | 下一步：[实操 09.3 商品排行榜案例（重点）](practice-09.3-product-ranking-case.md)*
