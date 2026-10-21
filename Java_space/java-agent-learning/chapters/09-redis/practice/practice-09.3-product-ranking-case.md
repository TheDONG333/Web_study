# 实操 09.3：商品排行榜案例（重点）⭐

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：让 AI 扮演面试官，问你这套排行榜怎么设计、怎么扩展

---

## 前置要求

- ✅ 已读完理论 09.6（ZSet 全部内容）
- ✅ 已完成实操 [09.2](practice-09.2-verification-code-case.md)，**继续用同一个 `redis-demo` 项目**

## 案例目标

实现商品销量排行榜——**ZSet 最经典、面试最高频的落地场景**：

| # | 接口 | 说明 | 底层命令 |
|---|------|------|---------|
| ① | POST `/api/rank/sale?productId=1001&count=1` | 模拟商品卖出 count 件 | `zincrby` |
| ② | GET `/api/rank/top?n=5` | 查销量前 N 名（带分数） | `zrevrange` |
| ③ | GET `/api/rank/product/{id}` | 查某商品：排第几 + 销量 | `zrevrank` / `zscore` |
| ④ | GET `/api/rank/range?min=500&max=1000` | 查销量在区间内的商品 | `zrangebyscore` |
| ⑤ | DELETE `/api/rank/product/{id}` | 商品下架（移出榜单） | `zrem` |

> 对比 MySQL：`ORDER BY sale DESC LIMIT 10` 每次都要全表排序；ZSet 是**随时维护好的有序结构**，查询零排序开销——这就是选它的理由。

---

## Step 1：初始化榜单数据（redis-cli 里敲）

```bash
# 分数 = 销量。商品用 ID 表示（真实项目展示时再拿 ID 查数据库换名字）
zadd rank:sale 980 1001    # 1001 = 机械键盘
zadd rank:sale 750 1002    # 1002 = 显示器
zadd rank:sale 620 1003    # 1003 = 耳机
zadd rank:sale 320 1004    # 1004 = 鼠标
zadd rank:sale 150 1005    # 1005 = 鼠标垫

zrevrange rank:sale 0 -1 WITHSCORES      # 先看一眼榜单
```

> 真实项目里，这个初始化动作通常是"从 MySQL 把销量同步过来"（榜单是**可再生的**——数据丢了随时能重建，这就是敢放 Redis 的底气）。

**同时清掉演示用的旧数据：**

```bash
del rank:sale:hot      # 如果之前练过热搜相关命令
```

---

## Step 2：RankService（核心）

```java
package com.demo.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
public class RankService {

    @Autowired
    private StringRedisTemplate redis;

    private static final String RANK_KEY = "rank:sale";   // 榜单的 key

    /** ① 商品卖出 count 件：加分（榜单自动重排） */
    public Double sell(String productId, int count) {
        return redis.opsForZSet().incrementScore(RANK_KEY, productId, count);
    }

    /** ② Top N：从高到低取（rank 从 1 开始，方便展示） */
    public List<Map<String, Object>> top(int n) {
        List<Map<String, Object>> list = new ArrayList<>();
        Set<ZSetOperations.TypedTuple<String>> top =
                redis.opsForZSet().reverseRangeWithScores(RANK_KEY, 0, n - 1);
        for (ZSetOperations.TypedTuple<String> t : top) {
            Map<String, Object> item = new LinkedHashMap<>();
            item.put("rank", list.size() + 1);        // 第几名
            item.put("productId", t.getValue());
            item.put("score", t.getScore());          // 销量
            list.add(item);
        }
        return list;
    }

    /** ③ 查某个商品的排名和销量（不存在返回 null） */
    public Map<String, Object> detail(String productId) {
        Double score = redis.opsForZSet().score(RANK_KEY, productId);
        if (score == null) return null;
        Long rank = redis.opsForZSet().reverseRank(RANK_KEY, productId);  // 0 开头

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("productId", productId);
        result.put("score", score);
        result.put("rank", rank + 1);                 // ⚠️ 展示要 +1：下标 0 = 第 1 名
        return result;
    }

    /** ④ 销量区间查询 */
    public Set<String> range(double min, double max) {
        return redis.opsForZSet().rangeByScore(RANK_KEY, min, max);
    }

    /** ⑤ 下架（移出榜单），返回是否真的删掉了 */
    public boolean remove(String productId) {
        return Boolean.TRUE.equals(redis.opsForZSet().remove(RANK_KEY, productId));
    }
}
```

---

## Step 3：RankController

```java
package com.demo.controller;

import com.demo.service.RankService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/rank")
public class RankController {

    @Autowired
    private RankService rankService;

    // ① 模拟卖出：POST /api/rank/sale?productId=1001&count=3
    @PostMapping("/sale")
    public Map<String, Object> sale(@RequestParam String productId,
                                    @RequestParam(defaultValue = "1") int count) {
        Double newScore = rankService.sell(productId, count);
        return Map.of("productId", productId, "newScore", newScore);
    }

    // ② 查榜单：GET /api/rank/top?n=5
    @GetMapping("/top")
    public Map<String, Object> top(@RequestParam(defaultValue = "5") int n) {
        return Map.of("top", rankService.top(n));
    }

    // ③ 查排名：GET /api/rank/product/1001
    @GetMapping("/product/{id}")
    public Map<String, Object> detail(@PathVariable("id") String id) {
        Map<String, Object> result = rankService.detail(id);
        return result != null ? result
                : Map.of("message", "商品不在榜上（可能已下架）");
    }

    // ④ 销量区间：GET /api/rank/range?min=500&max=1000
    @GetMapping("/range")
    public Map<String, Object> range(@RequestParam double min, @RequestParam double max) {
        return Map.of("products", rankService.range(min, max));
    }

    // ⑤ 下架：DELETE /api/rank/product/1005
    @DeleteMapping("/product/{id}")
    public Map<String, Object> remove(@PathVariable("id") String id) {
        boolean removed = rankService.remove(id);
        return Map.of("success", removed, "message", removed ? "已下架" : "商品不在榜上");
    }
}
```

---

## Step 4：测试全流程（观察"榜单自动变化"）

| 步骤 | 请求 | 预期结果 |
|------|------|---------|
| ① 看初始榜 | GET `/api/rank/top?n=5` | 1001(980) → 1002(750) → 1003(620) → 1004(320) → 1005(150) |
| ② 鼠标大卖 | POST `/api/rank/sale?productId=1004&count=500` | newScore = 820 |
| ③ 再看榜 | 再 GET `/api/rank/top?n=5` | **1004 从第 4 名升到第 2 名**（榜单自动重排！） |
| ④ 查排名 | GET `/api/rank/product/1004` | rank=2，score=820 |
| ⑤ 区间查询 | GET `/api/rank/range?min=500&max=1000` | 返回 1001、1002、1004 |
| ⑥ 下架 | DELETE `/api/rank/product/1005` | success=true |
| ⑦ 验证 | GET `/api/rank/top?n=5` | 榜单只剩 4 个商品；再查 1005 → "不在榜上" |
| ⑧ 命令对照 | redis-cli 里 `zrevrange rank:sale 0 -1 WITHSCORES` | 和接口返回完全一致 |

> **第 ③ 步是灵魂**：你只做了一次"加分"，榜单顺序自动就对了——**没有写任何排序代码**。这就是"选对数据结构，业务逻辑免费"。

---

## 练习任务

1. **做"今日榜"**：key 改成 `rank:sale:2026-09-24` 这种带日期的形式，每天一个榜；创建时顺手 `expire` 保留 7 天（提示：`redis.expire(key, ...)`）。
2. **Top3 加奖牌**：接口返回时给前 3 名加个 `medal` 字段（🥇🥈🥉），体会"展示逻辑"和"榜单数据"是分开的。
3. **热搜改造**：把 `zincrby` 的分数换成"被搜索次数"，改成热搜榜接口（原理完全一样，key 换成 `hot:search:今天`）。
4. **思考并回答**：
   - 为什么 `reverseRank` 返回 0 要展示成第 1 名？（下标 vs 名次——回忆 List/数组的下标习惯）
   - 榜单数据被误删了怎么办？（答：从 MySQL 重新按销量灌一遍 `zadd`——**可再生的数据才敢放内存**）
   - 如果两个商品销量完全相同，谁排前面？（提示：分数相同时 Redis 按成员名字典序排）

---

## 验证标准

- [ ] 5 个接口全部跑通，`top` 返回的名次从 1 开始
- [ ] 加分操作后榜单顺序**自动**变化，全程没写排序代码
- [ ] 查排名能返回正确名次（和榜单里数的一致）
- [ ] 区间查询、下架删除都生效
- [ ] redis-cli 里 `zrevrange ... WITHSCORES` 的排序和接口返回一致
- [ ] 说得出 zadd / zincrby / zrevrange / zrevrank / zrangebyscore / zrem 各自在业务里的角色

---

## 常见报错排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 榜单是空的 | 忘了 Step 1 初始化，或 key 名字不一致 | `keys rank:*` 看看真实 key 是什么 |
| 排名和榜单对不上 | 忘了 `+1`，或用了 `rank`（低→高）而不是 `reverseRank` | 排行榜从高到低要 reverse 系列 |
| `zrangebyscore` 查不到 | min/max 写反了（它要求从小到大） | 500 1000 而不是 1000 500 |
| 加分后名次没变 | 加分太少，没超过上一名 | 加大 count（如 500）再试 |
| 接口返回的 score 是小数 | Double 类型正常现象 | 展示时可取整（销量本来就该是整数） |

---

*上一节实操：[09.2 验证码案例](practice-09.2-verification-code-case.md) | 下一章：[10-RabbitMQ（消息队列）](../../10-rabbitmq/theory/10.1-mq-core-value.md)*
