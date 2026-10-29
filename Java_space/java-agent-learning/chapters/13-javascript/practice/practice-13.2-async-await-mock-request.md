# 实操 13.2：async/await 模拟异步请求（成功 + 失败）

> ⏱ **预计时长**：45 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：把报错整段发给 AI，重点问"是没 await 还是没 catch"

---

## 前置要求

- ✅ 已读完理论 13.4
- ✅ 已完成实操 [13.1](practice-13.1-array-operations.md)
- ✅ 浏览器即可（Chrome / Edge），不需要后端和任何依赖

## 使用方式

按 **F12** 打开控制台，把每步代码粘进去回车。**注意看输出的先后顺序**，这是理解异步的关键。

> 如果你把代码存成 `.js` 文件用 Node 跑：顶层 `await` 会报错，把调用包进 `async function main() { ... } main();` 即可。

---

## Step 1：先感受"异步不阻塞"

```javascript
console.log("1. 下单");

setTimeout(() => {
  console.log("3. 外卖到了");
}, 1000);

console.log("2. 继续干别的");
```

**预期输出顺序**：`1 → 2 → 3`（3 在 1 秒后才出现）。

**问题**：为什么代码里 3 写在 2 前面，输出却在后面？
（答：`setTimeout` 把回调交给浏览器计时，主线程没等它，继续执行了后面的代码。）

---

## Step 2：写一个模拟请求函数（返回 Promise）

真实项目里用 `fetch`，学习阶段先用 `setTimeout` 模拟"网络延迟"：

```javascript
function mockRequest(success = true) {
  return new Promise((resolve, reject) => {
    console.log("  请求已发出...");

    setTimeout(() => {
      if (success) {
        // 模拟后端返回的标准结构
        resolve({
          code: 200,
          message: "ok",
          data: { id: 1, name: "tom", role: "admin" }
        });
      } else {
        reject(new Error("网络超时，请稍后重试"));
      }
    }, 1000);
  });
}
```

> 这个函数就是"**取餐凭证**"：立刻返回一个 Promise，1 秒后才 resolve 或 reject。

---

## Step 3：用 then / catch 处理（Promise 写法）

### 3.1 成功场景

```javascript
mockRequest(true)
  .then(res => {
    console.log("成功:", res.data.name);
  })
  .catch(err => {
    console.log("失败:", err.message);
  });
```

**预期**：

```
  请求已发出...
成功: tom        ← 约 1 秒后
```

### 3.2 失败场景

```javascript
mockRequest(false)
  .then(res => {
    console.log("成功:", res.data.name);
  })
  .catch(err => {
    console.log("失败:", err.message);
  });
```

**预期**：

```
  请求已发出...
失败: 网络超时，请稍后重试
```

> 注意：失败时 `.then` 不执行，直接跳到 `.catch`。

---

## Step 4：改成 async/await（核心）

### 4.1 成功场景

```javascript
async function loadUser() {
  const res = await mockRequest(true);   // 等结果回来
  console.log("成功:", res.data.name);
}

loadUser();
console.log("loadUser 已调用（这行会先打印）");
```

**预期顺序**：

```
  请求已发出...
loadUser 已调用（这行会先打印）
成功: tom
```

> 这证明 `await` 只暂停了 `loadUser` 内部，**没有卡住主线程**。

### 4.2 失败场景（必须 try/catch）

```javascript
async function loadUserSafe() {
  try {
    const res = await mockRequest(false);
    console.log("成功:", res.data.name);
  } catch (err) {
    console.log("失败:", err.message);
  } finally {
    console.log("请求结束（无论成败）");
  }
}

loadUserSafe();
```

**预期**：

```
  请求已发出...
失败: 网络超时，请稍后重试
请求结束（无论成败）
```

> **规范**：用 `await` 就一定要处理失败。没有 `try/catch` 时，控制台会出现 `Uncaught (in promise)` 报错。

---

## Step 5：模拟真实的"业务错误"

后端常见的返回是 **HTTP 200，但 `code` 不是 200**（业务失败）。这种错误 Promise 不会自动 reject，需要你自己判断：

```javascript
async function fetchUser(success) {
  try {
    const res = await mockRequest(success);

    // ① 业务层判断：code 不对就主动抛错
    if (res.code !== 200) {
      throw new Error(res.message);
    }

    return res.data;              // 成功：返回数据
  } catch (err) {
    console.log("加载失败:", err.message);
    return null;                  // 失败：返回 null，调用方好判断
  }
}

async function main() {
  const user = await fetchUser(true);

  if (user) {
    console.log("渲染页面，欢迎", user.name);   // 成功分支
  } else {
    console.log("显示错误提示");
  }
}

main();
```

**预期**：`渲染页面，欢迎 tom`

把 `fetchUser(true)` 改成 `fetchUser(false)` 再跑：

**预期**：

```
加载失败: 网络超时，请稍后重试
显示错误提示
```

---

## Step 6：两个请求：串行 vs 并行（选做，了解）

### 串行：一个等一个，总耗时约 2 秒

```javascript
async function serial() {
  console.time("串行");
  await mockRequest(true);
  await mockRequest(true);
  console.timeEnd("串行");       // 约 2000ms
}
serial();
```

### 并行：同时发出，总耗时约 1 秒

```javascript
async function parallel() {
  console.time("并行");
  await Promise.all([mockRequest(true), mockRequest(true)]);
  console.timeEnd("并行");       // 约 1000ms
}
parallel();
```

> **结论**：两个请求互不依赖时，用 `Promise.all` 并行更快；有依赖（第二个要用第一个的结果）才串行。这和 Java 里连续 `join()` 两个 CompletableFuture vs `allOf` 是同一个道理。

---

## 验证标准

- [ ] Step 1 输出顺序是 1 → 2 → 3，能解释原因
- [ ] Step 3 成功和失败两种场景都能正确打印
- [ ] Step 4 的 `loadUser 已调用` 先于 `成功: tom` 打印
- [ ] Step 4.2 失败被 catch 住，没有 `Uncaught (in promise)`
- [ ] Step 5 能处理"HTTP 200 但 code != 200"的业务错误
- [ ] 能说出 `await` 和 `.then` 的关系（await 是 Promise 的语法糖）
- [ ] 能说出忘记写 `await` 时，拿到的是什么（Promise 对象，不是数据）

---

## 常见报错排查

| 报错 / 现象 | 原因 | 解决 |
|------------|------|------|
| `SyntaxError: await is only valid in async functions` | 在普通函数里用了 await | 函数前加 `async`，或用控制台顶层 await |
| `console.log(data)` 打印出 `Promise {<pending>}` | 忘了 `await` | 改成 `const data = await ...` |
| `Uncaught (in promise) Error: ...` | 没有 catch | 用 `try/catch` 或 `.catch()` |
| `err.message` 是 undefined | reject 时传了字符串而不是 Error | `reject(new Error("..."))` |
| 输出顺序和预期不符 | 对 `setTimeout` / `await` 的暂停范围理解有误 | 回看 Step 1 和 Step 4.1 的说明 |
| Node 里运行报 await 错误 | CommonJS 不支持顶层 await | 包进 `async function main() {}` 再调用 |

---

## 思考题

1. 为什么 `await` 不会像 Java 的 `Thread.sleep()` 那样卡住整个程序？
2. 如果 `mockRequest` 里既调了 `resolve` 又调了 `reject`，最终结果是什么？（提示：状态不可逆）
3. 调用一个 async 函数但**不写 await**，会发生什么？错误还能被 try/catch 抓到吗？
4. 把 Step 5 改成"失败后重试一次"，你会怎么用 try/catch 实现？（选做）

---

*上一节实操：[13.1 数组操作练习](practice-13.1-array-operations.md) | 下一阶段：[13.5 DOM 基础](../theory/13.5-dom-basics.md)*
