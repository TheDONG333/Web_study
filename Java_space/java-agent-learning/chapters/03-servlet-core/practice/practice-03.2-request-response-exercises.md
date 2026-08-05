# 实操 03.2：Request/Response 核心练习

> ⏱ **预计时长**：45 分钟
> 📌 **难度**：⭐⭐
> 🔧 **AI 辅助**：写完让 AI 帮你检查代码

---

## 前置要求

- ✅ 已读完理论 [03.1 Servlet 入门与 Request/Response](../theory/03.1-servlet-basics-request-response.md)
- ✅ 需要安装：JDK 17+、Tomcat 9+、IntelliJ IDEA

## 练习说明

4 个短练习，每个只复习一个核心点。代码骨架中 `TODO` 需要你自己写。

---

## 练习 1：参数获取（核心：getParameter）

**目标**：一个 Servlet 同时处理 GET 和 POST，获取参数并输出。

```java
@WebServlet("/param")
public class ParamServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO: 获取 username 参数并输出
        String name = /* TODO */;
        resp.getWriter().write("<h2>GET 方式: " + name + "</h2>");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // TODO: 设置请求编码（POST 中文不乱码的关键！）
        /* TODO */

        resp.setContentType("text/html;charset=UTF-8");

        // TODO: 获取表单参数并输出
        String name = /* TODO */;
        resp.getWriter().write("<h2>POST 方式: " + name + "</h2>");
    }
}
```

**测试：**
- 访问 `/param?username=zhangsan`
- 写一个 HTML 表单（`method="post"`），提交用户名

**验收：**
- [ ] GET 和 POST 都能拿到参数
- [ ] POST 提交中文不乱码

---

## 练习 2：请求头 + 响应输出（核心：getHeader / getWriter）

**目标**：读取请求头信息，用不同方式输出响应。

```java
@WebServlet("/header")
public class HeaderServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 1: 获取并输出 User-Agent（浏览器信息）和客户端 IP
        resp.getWriter().write("<p>浏览器: " + /* TODO 1 */ + "</p>");
        resp.getWriter().write("<p>IP: " + /* TODO 1 */ + "</p>");

        // TODO 2: 把状态码改成 200，然后输出一行 HTML
        /* TODO 2 */
    }
}
```

**验收：**
- [ ] 页面显示你的浏览器信息和 IP
- [ ] DevTools 中能看到响应状态码和 Content-Type

---

## 练习 3：转发 vs 重定向（核心：最易混淆的知识点）

**目标**：一个页面里放两个链接，分别体验转发和重定向，观察地址栏变化。

```java
// ====== ForwardServlet：转发 ======
@WebServlet("/forward")
public class ForwardServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // TODO 1: 放一个数据到 request 域
        req.setAttribute("msg", "转发过来的数据");

        // TODO 2: 转发到 /target（观察：地址栏变不变？）
        req.getRequestDispatcher("/target").forward(req, resp);
    }
}

// ====== RedirectServlet：重定向 ======
@WebServlet("/redirect")
public class RedirectServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // TODO 3: 重定向到 /target（观察：地址栏变不变？）
        resp.sendRedirect("/你的项目名/target");
    }
}

// ====== TargetServlet：目标页 ======
@WebServlet("/target")
public class TargetServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/html;charset=UTF-8");

        // TODO 4: 取出转发时放的 msg（重定向访问时它为 null）
        String msg = (String) req.getAttribute("msg");
        resp.getWriter().write("<h2>到达目标页</h2>");
        resp.getWriter().write("<p>msg = " + (msg != null ? msg : "null（重定向拿不到数据）") + "</p>");
    }
}
```

**测试步骤：**
1. 访问 `/forward` → 记录地址栏 URL → 观察 msg 值
2. 访问 `/redirect` → 记录地址栏 URL → 观察 msg 值
3. 对比两次访问后**地址栏**和 **msg 数据**的差异

**验收：**
- [ ] 转发：地址栏不变，msg 有值（1 次请求）
- [ ] 重定向：地址栏变成 /target，msg 为 null（2 次请求）
- [ ] 能说出"转发和重定向本质区别"就掌握了本章核心

---

## 练习 4（加分）：输出 JSON（核心：Content-Type + 状态码）

**目标**：写一个返回 JSON 的接口，含中文不乱码处理。

```java
@WebServlet("/api/user")
public class UserApiServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // TODO 1: 设置 Content-Type 为 application/json;charset=UTF-8
        /* TODO 1 */

        // TODO 2: 判断 id 参数，等于 1 返回成功 JSON，否则返回 404 + 错误 JSON
        // 成功: {"code":200,"name":"张三"}
        // 失败: 状态码 404 + {"code":404,"error":"用户不存在"}
        /* TODO 2 */
    }
}
```

**验收：**
- [ ] `/api/user?id=1` → 200 + `{"code":200,"name":"张三"}`
- [ ] `/api/user?id=2` → 404 + 错误 JSON
- [ ] 中文不乱码

---

## 完成标准

- [ ] 能熟练获取 GET/POST 参数
- [ ] 能设置响应头和 Content-Type
- [ ] **能清楚说出转发和重定向的区别**（本章最重要的点）
- [ ] 中文乱码知道在哪设编码

---

*上一节实操：[实操 03.1 完整项目](practice-03.1-login-register-crud-project.md) | 下一节实操：[实操 03.3 Session/Cookie 核心练习](practice-03.3-session-cookie-exercises.md)*
