# 实操 13.1：数组遍历、过滤、转换小练习

> ⏱ **预计时长**：45 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 检查你的链式调用是否可以更简洁

---

## 前置要求

- ✅ 已读完理论 13.1 ~ 13.3
- ✅ 有一个浏览器（Chrome / Edge 均可）
- ✅ 不需要安装任何东西——用浏览器控制台就能做

## 使用方式

1. 打开浏览器，按 **F12**（或右键 → 检查）
2. 切到 **Console（控制台）** 标签
3. 把下面的数据定义粘进去，回车
4. 按任务顺序，一行行敲代码，看输出

> 也可以把代码存成 `practice.js`，用 `node practice.js` 跑（需要装 Node.js）。控制台更方便，推荐。
>
> ⚠️ 控制台里重复粘贴同一段 `const` 声明会报 `Identifier 'xxx' has already been declared`——刷新页面重新来即可。

---

## 准备数据

```javascript
const products = [
  { id: 1, title: "Apple iPhone 15 Pro", category: "手机", price: 7999, stock: 100 },
  { id: 2, title: "Apple iPhone 14",     category: "手机", price: 4999, stock: 80  },
  { id: 3, title: "华为 Mate 60 Pro",     category: "手机", price: 6499, stock: 60  },
  { id: 4, title: "小米 14 Pro",          category: "手机", price: 5299, stock: 120 },
  { id: 5, title: "荣耀 Magic6 Pro",      category: "手机", price: 5699, stock: 90  },
  { id: 6, title: "机械键盘 87键",        category: "外设", price: 299,  stock: 200 },
  { id: 7, title: "罗技 MX Master 3S",    category: "外设", price: 699,  stock: 150 },
  { id: 8, title: "OPPO Find X7 Ultra",   category: "手机", price: 5999, stock: 70  }
];
```

> 这批数据就是上一章 ES 实操里的商品，只是换成了 JS 对象。

---

## 任务 1：forEach 遍历打印

**要求**：用 `forEach` 打印每个商品，格式：

```
1. Apple iPhone 15 Pro - 7999元
2. Apple iPhone 14 - 4999元
...
```

```javascript
products.forEach((p, index) => {
  // TODO：拼接字符串并打印
  // 提示：模板字符串 `${index + 1}. ${p.title} - ${p.price}元`
});
```

**验收**：控制台输出 8 行，编号从 1 开始。

---

## 任务 2：filter 按分类筛选

**要求**：筛出所有 `category === "手机"` 的商品。

```javascript
const phones = products.filter(/* TODO */);
console.log(phones);
```

**预期**：6 条（id 1、2、3、4、5、8）。

---

## 任务 3：filter 按价格筛选

**要求**：筛出价格低于 1000 元的商品。

```javascript
const cheap = products.filter(/* TODO */);
console.log(cheap.map(p => p.title));
```

**预期**：`["机械键盘 87键", "罗技 MX Master 3S"]`

---

## 任务 4：map 提取字段

**要求**：把所有商品标题提取成一个新数组。

```javascript
const titles = products.map(/* TODO */);
console.log(titles);
```

**预期**：长度为 8 的字符串数组，第一项是 `"Apple iPhone 15 Pro"`。

---

## 任务 5：map 做转换

**要求**：所有商品打 8 折，输出打折后的价格数组，保留两位小数（字符串即可）。

```javascript
const discounted = products.map(/* TODO */);
console.log(discounted);
```

**预期**（前 3 项）：

```
["6399.20", "3999.20", "5199.20", ...]
```

> 提示：`(p.price * 0.8).toFixed(2)`。`toFixed` 返回的是**字符串**。

---

## 任务 6：链式调用（重点）

**要求**：筛出 `category === "手机"` **且** `price <= 6000` 的商品，只输出标题。

```javascript
const result = products
  .filter(/* TODO：两个条件 */)
  .map(/* TODO：取标题 */);

console.log(result);
```

**预期**：

```
["Apple iPhone 14", "小米 14 Pro", "荣耀 Magic6 Pro", "OPPO Find X7 Ultra"]
```

> 这就是"接口返回一堆数据 → 筛选 → 取字段 → 渲染"的标准套路。

---

## 任务 7：push / pop 观察原数组变化

```javascript
// ① 新增一个商品，打印新长度
products.push({ id: 9, title: "测试商品", category: "外设", price: 99, stock: 1 });
console.log(/* TODO：长度 */);      // 预期 9

// ② 弹出最后一个，打印被弹出的对象和剩余长度
const removed = /* TODO：pop */;
console.log(/* TODO：removed.title */);   // 测试商品
console.log(/* TODO：长度 */);            // 8
```

**观察**：`push` / `pop` 直接修改了 `products` 本身（和 `map` / `filter` 不一样）。

---

## 任务 8：对象操作 + 解构

```javascript
const item = products[0];

// ① 新增属性 sales = 500
/* TODO */

// ② 读取 sales 并打印
console.log(/* TODO */);        // 500

// ③ 删除 stock 属性
/* TODO */
console.log(item.stock);        // undefined

// ④ 解构出 title 和 price，打印
const { title, price } = /* TODO */;
console.log(title, price);      // Apple iPhone 15 Pro 7999

// ⑤ 解构 + 默认值（item 没有 brand 属性）
const { brand = "未知品牌" } = item;
console.log(brand);             // 未知品牌
```

---

## 任务 9（综合）：计算手机总库存

**要求**：用 `filter` + `forEach` 算所有手机类商品的库存总和。

```javascript
let totalStock = 0;

products
  .filter(/* TODO：手机分类 */)
  .forEach(/* TODO：累加 stock */);

console.log(totalStock);
```

**预期**：`520`（100+80+60+120+90+70）。

---

## 验证标准

- [ ] 任务 1 输出 8 行，编号和价格格式正确
- [ ] 任务 2 返回 6 条手机
- [ ] 任务 3 返回键盘和鼠标两条
- [ ] 任务 4 得到 8 个标题
- [ ] 任务 5 价格是打折后的两位小数字符串
- [ ] 任务 6 得到 4 个标题，说明链式调用写对了
- [ ] 任务 7 长度从 8 → 9 → 8，且能说出 push/pop 改了原数组
- [ ] 任务 8 增删读、解构、默认值全部正确
- [ ] 任务 9 得到 520
- [ ] 能说出 forEach / map / filter 三者返回值的区别

---

## 思考题

1. `map` 和 `forEach` 都能遍历数组，什么时候必须用 `map`？（提示：需要"加工后的新数组"时）
2. `filter` 的回调函数返回值必须是什么类型？如果返回 `1` 或 `"a"` 会怎样？（提示：真假值）
3. 为什么任务 6 里 `filter` 用 `&&` 连两个条件，而不是写两个 `filter`？（两种都能实现，想想哪种更直观）
4. `const products` 不能重新赋值，为什么还能 `push`？（回忆 13.1：const 锁指向不锁内容）

---

## 参考答案（先自己做，再对照）

<details>
<summary>点开查看</summary>

```javascript
// 任务 1
products.forEach((p, i) => console.log(`${i + 1}. ${p.title} - ${p.price}元`));

// 任务 2
const phones = products.filter(p => p.category === "手机");

// 任务 3
const cheap = products.filter(p => p.price < 1000);

// 任务 4
const titles = products.map(p => p.title);

// 任务 5
const discounted = products.map(p => (p.price * 0.8).toFixed(2));

// 任务 6
const result = products
  .filter(p => p.category === "手机" && p.price <= 6000)
  .map(p => p.title);

// 任务 7
products.push({ id: 9, title: "测试商品", category: "外设", price: 99, stock: 1 });
const removed = products.pop();

// 任务 8
item.sales = 500;
delete item.stock;
const { title, price } = item;
const { brand = "未知品牌" } = item;

// 任务 9
let totalStock = 0;
products.filter(p => p.category === "手机").forEach(p => { totalStock += p.stock; });
```

</details>

---

*上一节理论：[13.3 数组与对象](../theory/13.3-arrays-and-objects.md) | 下一步：[实操 13.2 async/await 模拟请求](practice-13.2-async-await-mock-request.md)*
