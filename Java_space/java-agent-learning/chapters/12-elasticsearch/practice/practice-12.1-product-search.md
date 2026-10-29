# 实操 12.1：商品关键词搜索 + 价格区间筛选 + 标题高亮

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：查询结果不符合预期时，把请求 JSON 和返回结果一起发给 AI 分析

---

## 前置要求

- ✅ 已读完理论 12.1 ~ 12.4
- ✅ 已安装 Docker Desktop，并已启动（分配内存 ≥ 4GB）
- ✅ 不需要写 Java 代码——本次用 Kibana Dev Tools（或 Postman）直接调 ES

## 实践目标

在一份商品数据上，完成一个**电商搜索页背后真正的查询**：

1. 关键词搜索（match）
2. 分类精确筛选（term）
3. 价格区间筛选（range）
4. 排除某品牌（must_not）
5. 分页 + 标题高亮

最终把 1~5 组合成**一个 bool 查询**——这就是你在电商 App 里搜"Pro"时后端实际做的事。

---

## Step 1：启动 Elasticsearch

```bash
docker run -d --name elasticsearch \
  -p 9200:9200 \
  -e discovery.type=single-node \
  -e xpack.security.enabled=false \
  -e ES_JAVA_OPTS="-Xms1g -Xmx1g" \
  docker.elastic.co/elasticsearch/elasticsearch:8.17.0
```

| 参数 | 作用 |
|------|------|
| `-p 9200:9200` | ES 的 REST 接口端口 |
| `discovery.type=single-node` | 单节点模式（学习用，不做集群） |
| `xpack.security.enabled=false` | 关闭账号密码（学习环境，生产别关） |
| `ES_JAVA_OPTS` | 限制内存 1GB，防止吃满电脑 |

**等待 20~30 秒**，浏览器访问 **http://localhost:9200**，看到类似：

```json
{
  "name": "xxx",
  "cluster_name": "docker-cluster",
  "version": { "number": "8.17.0" },
  "tagline": "You Know, for Search"
}
```

就说明 ES 启动成功。

> 起不来先看日志：`docker logs elasticsearch`。

---

## Step 2：打开查询工具（二选一）

### 方式 A：Kibana Dev Tools（推荐）

```bash
docker run -d --name kibana \
  -p 5601:5601 \
  --add-host=host.docker.internal:host-gateway \
  -e ELASTICSEARCH_HOSTS=http://host.docker.internal:9200 \
  docker.elastic.co/kibana/kibana/8.17.0
```

等 **1~2 分钟**（Kibana 启动比 ES 慢），浏览器打开 **http://localhost:5601** → 左侧菜单 → **Dev Tools**。

Dev Tools 的用法：左边输入请求，点绿色三角形执行，右边看结果。

```
GET /products/_search
{
  "query": { "match_all": {} }
}
```

> 第一行是"方法 + 路径"，下面是要发送的 JSON。

### 方式 B：Postman（不想多起容器）

所有请求都发到 `http://localhost:9200`：

- 方法：`PUT` / `POST` / `GET` / `DELETE`
- Body 选 `raw` → `JSON`
- 例如：`POST http://localhost:9200/products/_search`，Body 填查询 JSON

> 下面的示例按 Dev Tools 的写法给出，改成 Postman 就是"URL + Body"。

---

## Step 3：创建索引 products

在 Dev Tools 里执行：

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title":     { "type": "text" },
      "category":  { "type": "keyword" },
      "brand":     { "type": "keyword" },
      "price":     { "type": "double" },
      "stock":     { "type": "integer" },
      "createdAt": { "type": "date" }
    }
  }
}
```

返回 `"acknowledged": true` 即成功。

> 记住 `text` 和 `keyword` 的分配：title 要**关键词搜索**，所以 text；category/brand 要**精确过滤**，所以 keyword（理论 12.2 的 term 全靠它）。

---

## Step 4：灌入 8 条商品数据

用 `_bulk` 批量写入。规则很简单：**每个商品两行**——第一行声明"写哪条"，第二行是商品内容。

```json
POST /products/_bulk
{"index":{"_id":"1"}}
{"title":"Apple iPhone 15 Pro 手机","category":"手机","brand":"Apple","price":7999,"stock":100,"createdAt":"2026-09-01"}
{"index":{"_id":"2"}}
{"title":"Apple iPhone 14 手机","category":"手机","brand":"Apple","price":4999,"stock":80,"createdAt":"2025-09-01"}
{"index":{"_id":"3"}}
{"title":"华为 Mate 60 Pro 手机","category":"手机","brand":"华为","price":6499,"stock":60,"createdAt":"2026-08-15"}
{"index":{"_id":"4"}}
{"title":"小米 14 Pro 手机","category":"手机","brand":"小米","price":5299,"stock":120,"createdAt":"2026-07-20"}
{"index":{"_id":"5"}}
{"title":"荣耀 Magic6 Pro 手机","category":"手机","brand":"荣耀","price":5699,"stock":90,"createdAt":"2026-06-10"}
{"index":{"_id":"6"}}
{"title":"机械键盘 87键 青轴","category":"外设","brand":"罗技","price":299,"stock":200,"createdAt":"2025-03-01"}
{"index":{"_id":"7"}}
{"title":"罗技 MX Master 3S 鼠标","category":"外设","brand":"罗技","price":699,"stock":150,"createdAt":"2025-05-20"}
{"index":{"_id":"8"}}
{"title":"OPPO Find X7 Ultra 手机","category":"手机","brand":"OPPO","price":5999,"stock":70,"createdAt":"2026-05-05"}
```

返回的 `items` 里每条都是 `"status": 201`（或 200）就说明全部写入成功。

先确认数据总量：

```json
GET /products/_count
```

预期 `"count": 8`。

---

## Step 5：关键词搜索（match）

搜标题含 "iPhone" 的商品：

```json
GET /products/_search
{
  "query": {
    "match": { "title": "iPhone" }
  }
}
```

**预期**：命中 2 条（id=1、id=2），每条有 `_score`。

> 试一下搜 `15 iPhone`（词序反了）——照样能命中 id=1。这就是 match 分词的好处，`LIKE` 做不到。

---

## Step 6：精确筛选（term）

查所有"手机"分类（category 是 keyword）：

```json
GET /products/_search
{
  "query": {
    "term": { "category": "手机" }
  }
}
```

**预期**：命中 6 条（id=1,2,3,4,5,8），外设两条不出现。

> **对照实验（重要）**：把 `"category"` 换成 `"title"`，值还是 `"iPhone"`：
>
> ```json
> { "term": { "title": "iPhone" } }
> ```
>
> 结果是 **0 条**。为什么？title 是 text，写入时已被拆成小写词 `iphone`；term 不分词、不转小写，拿整个 `"iPhone"` 去比对，当然对不上。**查 text 字段请用 match。**

---

## Step 7：价格区间（range）

查 3000~6000 元的商品：

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": { "gte": 3000, "lte": 6000 }
    }
  }
}
```

**预期**：命中 4 条（id=2 4999、id=4 5299、id=5 5699、id=8 5999）。

> 把 `gte` 换成 `gt`，结果不变（没有正好 3000 的商品）；试试 `{"gte": 7999}`，只有 id=1。

---

## Step 8：终极查询——bool + 分页 + 高亮

需求（电商搜索页常见组合）：

- 搜索标题含 **"Pro"**
- 价格 **5000~7000**
- **排除 Apple** 品牌
- 每页 **2 条**，看第 1 页
- 标题中的 **Pro 高亮**

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "Pro" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 5000, "lte": 7000 } } }
      ],
      "must_not": [
        { "term": { "brand": "Apple" } }
      ]
    }
  },
  "from": 0,
  "size": 2,
  "highlight": {
    "fields": { "title": {} },
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"]
  }
}
```

**预期**：命中 3 条（id=3 华为 6499、id=4 小米 5299、id=5 荣耀 5699），第 1 页返回 **2 条**（按相关度排序，顺序可能略有差异）。

每条命中的 `highlight` 里能看到：

```json
"highlight": {
  "title": ["华为 Mate 60 <em>Pro</em> 手机"]
}
```

**验证分页**：把 `"from": 0` 改成 `"from": 2`，再执行——返回剩下的 1 条。

**验证 must_not**：去掉 `must_not` 再查——id=1 的 `Apple iPhone 15 Pro`（7999）价格不在区间内，其实仍然不会出现。想看 must_not 的效果，把价格上限改成 8000 再对比。

---

## Step 9：如果不用 Kibana（Postman / curl）

Dev Tools 的请求翻译成 HTTP 就是：

```
POST http://localhost:9200/products/_search
Content-Type: application/json

{ ...上面的 JSON... }
```

curl 版本（Git Bash）：

```bash
curl -X POST "http://localhost:9200/products/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"title":"iPhone"}}}'
```

> 学习期推荐 Kibana：JSON 写错会直接标红，返回结果也能折叠查看。

---

## 验证标准

- [ ] http://localhost:9200 能返回集群信息
- [ ] `GET /products/_count` 返回 8
- [ ] match "iPhone" → 2 条；搜 "15 iPhone" 也能命中
- [ ] term category "手机" → 6 条
- [ ] term title "iPhone" → **0 条**，并能解释原因（text 被分词）
- [ ] range 3000~6000 → 4 条
- [ ] 终极 bool 查询命中 3 条，第 1 页返回 2 条
- [ ] `from` 改成 2 后返回剩余 1 条
- [ ] highlight 返回 `<em>Pro</em>`，不是改动了 `_source` 原文
- [ ] 能说出 must / filter / must_not 各负责什么

---

## 练习任务

1. **filter 换 must 对比**：把价格 range 从 `filter` 移到 `must`，查询结果一样，但思考：为什么官方推荐放 filter？（提示：打分与缓存）
2. **加一个过滤条件**：只查有货的（`stock > 0`），放进 filter。
3. **换成日期范围**：查 2026 年上架的商品（`createdAt` gte `2026-01-01`）。
4. **试试 fuzzy 和 wildcard**（了解，观察即可）：
   - `{"fuzzy": {"title": "iphnoe"}}`（故意拼错）→ 能纠错吗？
   - `{"wildcard": {"title": "iPho*"}}` → 能匹配吗？
   - 思考：为什么 wildcard 开头带 `*` 性能差？
5. **改高亮标签**：把 `pre_tags` / `post_tags` 换成 `<span class="hl">` / `</span>`，模拟前端自定义样式。
6. **（选做）删除重建**：`DELETE /products` 后重新建索引灌数据，体会"索引结构不对就得重建"。

---

## 常见报错排查

| 报错 / 现象 | 原因 | 解决 |
|------------|------|------|
| 浏览器打不开 9200 | ES 还在启动 / 容器退出了 | 等 30 秒；`docker logs elasticsearch` 看原因 |
| 容器反复退出，日志提示内存不足 | Docker 分配内存太小 | Docker Desktop → Settings → Resources 调到 4GB+ |
| 日志提示 `vm.max_map_count too low` | WSL2 限制 | PowerShell 执行 `wsl -d docker-desktop sysctl -w vm.max_map_count=262144` 后重启 ES |
| `security_exception` / 需要认证 | 没关 security | 检查 `-e xpack.security.enabled=false` |
| `index_not_found_exception` | 索引还没建 / 名字写错 | 先执行 Step 3，确认 `GET /_cat/indices` 里有 products |
| term 查询返回 0 | 查了 text 字段 | 改用 match，或把字段建成 keyword |
| highlight 没有内容 | 高亮字段没参与查询 / 字段名写错 | 确保 query 里搜的就是该字段 |
| `Result window is too large` | from + size 超过 10000 | 学习阶段别翻那么深；生产用 search_after |
| Kibana 页面一直 "not ready" | Kibana 启动慢 / 连不上 ES | 等 2 分钟；检查 `ELASTICSEARCH_HOSTS` |
| Dev Tools 报解析错误 | JSON 少了逗号/括号 | 看红色提示的位置，逐行检查 |

---

## 环境清理（可选）

```bash
docker stop elasticsearch kibana
docker rm elasticsearch kibana
```

> 没挂载数据卷，删容器 = 数据清空。学习阶段无所谓，需要时重新跑一遍脚本即可。

---

*上一节理论：[12.4 分页、高亮与了解项](../theory/12.4-pagination-highlight-and-more.md) | 下一章：[13-JavaScript（前端基础）](../../13-javascript/theory/13.1-variables-types-and-basics.md)*
