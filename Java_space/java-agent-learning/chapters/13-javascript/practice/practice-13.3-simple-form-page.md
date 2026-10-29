# 实操 13.3：最简表单测试页（输入框 + 按钮 + 控制台打印）

> ⏱ **预计时长**：40 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：页面没反应时，把 HTML 和报错一起发给 AI，重点问"是选择器错了还是事件没绑上"

---

## 前置要求

- ✅ 已读完理论 13.5 ~ 13.6
- ✅ 有一个浏览器（Chrome / Edge）
- ✅ 会用 VS Code 新建 HTML 文件（或任意文本编辑器）
- ✅ 不需要后端、不需要安装任何东西

## 实践目标

写一个**不用美化**的测试页面：

1. 用户名输入框
2. 密码输入框
3. 「提交」按钮
4. 点击提交后：校验非空 → 在控制台打印输入内容 → 页面显示提交结果

这个页面以后可以直接改造成"自测接口"的调试页。

---

## Step 1：新建文件

新建文件夹 `js-form-demo`，在里面创建 `index.html`：

```
js-form-demo/
└── index.html
```

---

## Step 2：写 HTML 结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>登录测试页</title>
</head>
<body>
  <h2>登录测试</h2>

  <form id="loginForm">
    <label>用户名：
      <input id="username" type="text" placeholder="请输入用户名">
    </label>
    <br><br>

    <label>密码：
      <input id="password" type="password" placeholder="请输入密码">
    </label>
    <br><br>

    <button type="submit">提交</button>
  </form>

  <p id="message"></p>
</body>
</html>
```

**先只写 HTML**，用浏览器打开，确认页面能看到两个输入框和一个按钮。

> 注意三个 id：`loginForm`、`username`、`password`，后面的 JS 全靠它们找元素。

---

## Step 3：加 JS（放在 `</body>` 之前）

```html
<script>
  const form = document.getElementById("loginForm");
  const message = document.getElementById("message");

  form.onsubmit = function (event) {
    event.preventDefault();                 // 阻止页面刷新

    const username = document.getElementById("username").value.trim();
    const password = document.getElementById("password").value;

    // 校验：用户名不能为空
    if (!username) {
      message.innerText = "请输入用户名";
      return;
    }

    // 校验：密码至少 6 位
    if (password.length < 6) {
      message.innerText = "密码至少 6 位";
      return;
    }

    // 校验通过：打印 + 页面提示
    console.log("用户名:", username);
    console.log("密码:", password);
    message.innerText = "提交成功，请查看控制台";
  };
</script>
```

### 为什么 script 要放在 body 末尾？

因为 JS 执行时要去页面里找元素。**如果放在 `<head>` 里，元素还没被浏览器创建，`getElementById` 会返回 `null`。**

> 另一种写法是用 `DOMContentLoaded` 事件，但本阶段放在末尾最简单。

---

## Step 4：运行并测试

1. 双击 `index.html`，浏览器打开
2. 按 **F12** → Console（控制台）
3. 按顺序测试：

| 场景 | 操作 | 预期结果 |
|------|------|---------|
| ① 用户名为空 | 直接点提交 | 页面显示"请输入用户名"，控制台无输出 |
| ② 密码太短 | 用户名填 `tom`，密码填 `123` | 页面显示"密码至少 6 位" |
| ③ 提交成功 | 用户名 `tom`，密码 `123456` | 控制台打印 `用户名: tom`、`密码: 123456`，页面显示"提交成功" |
| ④ 回车提交 | 在密码框按回车 | 与点击提交效果一致 |
| ⑤ 空格校验 | 用户名只输入空格 | 提示"请输入用户名"（trim 生效） |
| ⑥ 确认没刷新 | 提交成功后看页面 | 输入框内容还在，页面没有跳转 |

> 如果提交后页面一闪、输入清空——说明 `preventDefault()` 没生效，检查这行是否写在 `onsubmit` 最前面。

---

## Step 5：加一个下拉框（巩固取值）

在密码框下面加：

```html
<label>角色：
  <select id="role">
    <option value="user">普通用户</option>
    <option value="admin">管理员</option>
  </select>
</label>
<br><br>
```

在 JS 打印部分加上：

```javascript
const role = document.getElementById("role").value;
console.log("角色:", role);
```

切换选项再提交，观察控制台打印的 role 变化。

---

## 完整代码（对照检查）

<details>
<summary>点开查看完整 index.html</summary>

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>登录测试页</title>
</head>
<body>
  <h2>登录测试</h2>

  <form id="loginForm">
    <label>用户名：
      <input id="username" type="text" placeholder="请输入用户名">
    </label>
    <br><br>

    <label>密码：
      <input id="password" type="password" placeholder="请输入密码">
    </label>
    <br><br>

    <label>角色：
      <select id="role">
        <option value="user">普通用户</option>
        <option value="admin">管理员</option>
      </select>
    </label>
    <br><br>

    <button type="submit">提交</button>
  </form>

  <p id="message"></p>

  <script>
    const form = document.getElementById("loginForm");
    const message = document.getElementById("message");

    form.onsubmit = function (event) {
      event.preventDefault();

      const username = document.getElementById("username").value.trim();
      const password = document.getElementById("password").value;
      const role = document.getElementById("role").value;

      if (!username) {
        message.innerText = "请输入用户名";
        return;
      }

      if (password.length < 6) {
        message.innerText = "密码至少 6 位";
        return;
      }

      console.log("用户名:", username);
      console.log("密码:", password);
      console.log("角色:", role);
      message.innerText = "提交成功，请查看控制台";
    };
  </script>
</body>
</html>
```

</details>

---

## 验证标准

- [ ] 页面能看到用户名、密码、提交按钮
- [ ] 点击提交不会刷新页面（输入内容还在）
- [ ] 用户名为空时提示"请输入用户名"
- [ ] 密码少于 6 位时提示"密码至少 6 位"
- [ ] 校验通过时控制台打印用户名和密码
- [ ] 密码框按回车也能提交
- [ ] 用户名只输空格时被 trim 拦住
- [ ] 下拉框的 value 能正确打印
- [ ] 能说出 `event.preventDefault()` 的作用

---

## 练习任务

1. **加确认密码**：再加一个"确认密码"输入框，两次密码不一致时提示"两次密码不一致"。
2. **加错误样式**：错误时把 `message` 的文字变红（用 `message.style.color = "red"`，了解一下 style 属性）。
3. **提交后清空**：成功后把输入框清空（`input.value = ""`）。
4. **换选择器**：把 `getElementById("username")` 改成 `document.querySelector("#username")`，确认效果一样。
5. **按钮禁用**：提交成功后把按钮 `disabled = true`，文字改成"已提交"。
6. **思考**：为什么事件要绑在 `form` 的 `onsubmit` 上，而不是只绑在按钮的 `onclick` 上？（提示：回车提交）

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 点击没反应 | script 在元素之前执行，或 id 写错 | script 放 body 末尾；核对 id 拼写 |
| 控制台报 `Cannot read properties of null` | 没找到元素 | 检查 `getElementById` 的 id 和 HTML 是否一致 |
| 提交后页面刷新、输入清空 | 没写 `event.preventDefault()` | 放在 `onsubmit` 第一行 |
| 控制台没有任何输出 | 没按 F12 / 校验提前 return 了 | 打开 Console；先看页面提示信息 |
| 中文乱码 | HTML 没声明编码 | 加 `<meta charset="UTF-8">` |
| 下拉框拿到的是显示文字 | 取错属性 | 用 `.value`，不是 `.innerText` |
| 回车不提交 | 按钮不是 `type="submit"` | 改成 `type="submit"` |

---

*上一节理论：[13.6 事件绑定与表单操作](../theory/13.6-events-and-forms.md) | 下一步：待规划*
