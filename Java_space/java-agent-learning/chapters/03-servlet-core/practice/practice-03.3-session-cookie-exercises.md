# 实操 03.3：Session/Cookie 核心练习

> ⏱ **预计时长**：45 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 帮你检查代码

---

## 前置要求

- ✅ 已读完理论 [03.2 Session 与 Cookie](../theory/03.2-session-and-cookie.md)
- ✅ 已完成实操 [03.2 Request/Response 核心练习](practice-03.2-request-response-exercises.md)
- ✅ 需要安装：JDK 17+、Tomcat 9+、IntelliJ IDEA

## 练习说明

3 个短练习，覆盖本章最核心的 3 个知识点：Cookie 存取、Session 存取、登录态。代码骨架中 `TODO` 需要你自己写。

---

## 练习 1：Cookie 存取（核心：addCookie / getCookies）

**目标**：设置一个 Cookie，再在另一个页面读取它。

```java
// ====== 设置 Cookie ======
@WebServlet("/set-cookie")
public class SetCookieServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 1: 创建 Cookie（username=zhangsan），设置存活 1 小时
        Cookie cookie = new Cookie("username", "zhangsan");
        cookie.setMaxAge(/* TODO 1 */);   // 单位：秒

        // TODO 2: 把 Cookie 发给浏览器
        /* TODO 2 */

        resp.getWriter().write("<h2>Cookie 已设置</h2>");
        resp.getWriter().write("<a href='/你的项目名/read-cookie'>去读取</a>");
    }
}

// ====== 读取 Cookie ======
@WebServlet("/read-cookie")
public class ReadCookieServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 3: 遍历 req.getCookies()，把 username 的值显示出来
        // 没有则显示"没有 Cookie"
        Cookie[] cookies = req.getCookies();
        String username = null;
        if (cookies != null) {
            for (Cookie c : cookies) {
                if ("username".equals(/* TODO 3 */)) {
                    username = c.getValue();
                }
            }
        }
        resp.getWriter().write("<h2>" + (username != null ? "你好, " + username : "没有 Cookie") + "</h2>");
    }
}
```

**观察（F12 → Application → Cookies）：** 看 JSESSIONID 之外，还有 username 这个 Cookie。

**验收：**
- [ ] 先访问 /set-cookie，再访问 /read-cookie，能显示"你好, zhangsan"
- [ ] 直接在无痕窗口访问 /read-cookie，显示"没有 Cookie"

---

## 练习 2：Session 存取 + 访问计数（核心：setAttribute / getAttribute）

**目标**：用 Session 记住"访问了几次"，刷新页面计数递增。

```java
@WebServlet("/visit")
public class VisitCountServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 1: 获取 Session（没有会自动创建）
        HttpSession session = /* TODO 1 */;

        // TODO 2: 取出访问次数，没有则从 1 开始，有则 +1
        Integer count = (Integer) session.getAttribute("count");
        count = (count == null) ? /* TODO 2 */ : /* TODO 2 */;

        // TODO 3: 存回 Session
        /* TODO 3 */

        resp.getWriter().write("<h2>第 " + count + " 次访问</h2>");
    }
}
```

**测试：**
1. 连续刷新 3 次，观察计数
2. 打开**无痕窗口**再访问 → 计数从 1 开始（不同 Session！）

**验收：**
- [ ] 刷新页面计数每次 +1
- [ ] 无痕窗口访问计数从 1 开始
- [ ] 能说出：计数为什么不会因为刷新而清零？（Session 存在服务器）

---

## 练习 3：登录态检查（核心：登录 → 校验 → 退出）

**目标**：把 Session 用在真实的登录场景——这也是所有网站登录功能的最小实现。

```java
// ====== 登录接口（表单 POST 提交） ======
@WebServlet("/login")
public class LoginServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        req.setCharacterEncoding("UTF-8");
        String username = req.getParameter("username");
        String password = req.getParameter("password");

        if ("admin".equals(username) && "123456".equals(password)) {
            // TODO 1: 登录成功 → 把用户名存入 Session
            req.getSession().setAttribute("username", username);

            // TODO 2: 重定向到 /home（为什么用重定向？——PRG 模式防重复提交）
            resp.sendRedirect("/你的项目名/home");
        } else {
            // TODO 3: 失败 → 回到登录页并带错误标记
            resp.sendRedirect("/你的项目名/login.html?error=1");
        }
    }
}

// ====== 受保护页面 ======
@WebServlet("/home")
public class HomeServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 4: 检查是否登录（getSession(false) 不创建新 Session）
        HttpSession session = req.getSession(false);
        String username = (session != null)
                ? (String) session.getAttribute("username")
                : null;

        if (username != null) {
            resp.getWriter().write("<h2>欢迎, " + username + "!</h2>");
            resp.getWriter().write("<a href='/你的项目名/logout'>退出</a>");
        } else {
            // TODO 5: 未登录 → 跳回登录页
            resp.sendRedirect("/你的项目名/login.html");
        }
    }
}

// ====== 退出登录 ======
@WebServlet("/logout")
public class LogoutServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // TODO 6: 销毁 Session，回到登录页
        HttpSession session = req.getSession(false);
        if (session != null) {
            /* TODO 6 */
        }
        resp.sendRedirect("/你的项目名/login.html");
    }
}
```

**login.html（放 webapp 目录下）：**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head><meta charset="UTF-8"><title>登录</title></head>
<body>
    <h2>登录</h2>
    <!-- 提交到 /你的项目名/login，POST 方式 -->
    <form action="/你的项目名/login" method="post">
        用户名: <input type="text" name="username"><br><br>
        密码: <input type="password" name="password"><br><br>
        <button type="submit">登录</button>
    </form>
    <p style="color:red">错误信息：<span id="err"></span></p>
    <script>
        // 从 URL 参数读取 error
        if (new URLSearchParams(location.search).get('error')) {
            document.getElementById('err').textContent = '用户名或密码错误';
        }
    </script>
</body>
</html>
```

**测试流程：**
1. 直接访问 `/home` → 被跳回登录页
2. 输入 admin / 123456 → 登录成功进入 /home
3. 点"退出" → 再访问 /home → 又被跳回登录页

**验收：**
- [ ] 未登录访问 /home 被拦截
- [ ] 登录成功显示欢迎页
- [ ] 错误密码显示"用户名或密码错误"
- [ ] 退出后再次访问 /home 被拦截
- [ ] 能说出：登录态存在哪里？Session 还是 Cookie？

---

## 完成标准

- [ ] 会设置和读取 Cookie
- [ ] 会用 Session 存数据（getSession / setAttribute / getAttribute）
- [ ] **能独立实现"登录 → 校验 → 退出"闭环**（本章最重要的能力）
- [ ] 知道 `getSession()` 和 `getSession(false)` 的区别
- [ ] 知道不同浏览器之间为什么登录态不共享

---

*上一节实操：[实操 03.2 Request/Response 核心练习](practice-03.2-request-response-exercises.md) | 下一步：[实操 03.1 完整项目](practice-03.1-login-register-crud-project.md)*
