# 实操 03.1：基于 Servlet + MVC 三层架构实现用户管理系统

> ⏱ **预计时长**：90 分钟
> 📌 **难度**：⭐⭐⭐⭐
> 🔧 **AI 辅助**：用 AI 生成基础 CRUD 代码，自己专注于分层设计、逻辑校验和边界处理

---

## 前置要求

- ✅ 已完成章节：03.1 ~ 03.4 全部理论
- ✅ 需要安装：JDK 17+、Tomcat 9+、MySQL 8+、IntelliJ IDEA（或 Eclipse）
- ✅ 环境准备：能用 IDEA 创建 Java Web 项目并部署到 Tomcat

## 实践目标

完成本次实操后，你将能够：

1. 从零搭建一个基于纯 Servlet 的 Web 项目
2. 用 MVC 三层架构写出 Controller → Service → Dao 的完整调用链
3. 用 Filter 实现全局编码处理和登录拦截
4. 用 Listener 实现启动初始化和在线人数统计
5. 理解纯 Servlet 项目和 Spring Boot 项目之间的关系

---

## 项目全貌

### 功能清单

```
用户管理系统
├── ① 用户注册（表单提交 → Controller → Service → Dao → 数据库）
├── ② 用户登录（表单 → Session 保存登录态 → 跳转首页）
├── ③ 用户列表（登录后才能看 → 支持关键词搜索 → 分页展示）
├── ④ 用户详情（点击列表项 → 跳转详情页）
├── ⑤ 编辑用户（修改邮箱、手机号）
├── ⑥ 删除用户（软删除）
└── ⑦ 退出登录（销毁 Session → 跳回登录页）
```

### 最终项目结构

```
user-management/
├── pom.xml  (Maven)
├── src/main/
│   ├── java/com/example/usermgmt/
│   │   ├── controller/
│   │   │   └── UserController.java        ← 前端控制器（统一路由入口）
│   │   ├── service/
│   │   │   ├── UserService.java            ← 业务接口
│   │   │   └── impl/
│   │   │       └── UserServiceImpl.java    ← 业务实现
│   │   ├── dao/
│   │   │   ├── UserDao.java               ← 数据访问接口
│   │   │   └── impl/
│   │   │       └── UserDaoImpl.java        ← JDBC 实现
│   │   ├── entity/
│   │   │   ├── User.java                  ← 用户实体
│   │   │   └── PageResult.java            ← 分页结果
│   │   ├── filter/
│   │   │   ├── EncodingFilter.java        ← 编码过滤器
│   │   │   └── LoginCheckFilter.java      ← 登录拦截过滤器
│   │   ├── listener/
│   │   │   ├── AppStartupListener.java    ← 启动监听器
│   │   │   └── OnlineCountListener.java   ← 在线人数监听器
│   │   └── util/
│   │       └── DBUtil.java                ← 数据库连接工具
│   │
│   └── webapp/
│       ├── WEB-INF/
│       │   └── web.xml
│       ├── login.html                     ← 登录页
│       ├── register.html                  ← 注册页
│       ├── css/
│       │   └── style.css                  ← 样式文件
│       └── js/
│           └── common.js                  ← 公共 JS
```

---

## 步骤

### Step 1：环境与数据库准备

**1.1 创建 MySQL 数据库和表**

```sql
CREATE DATABASE IF NOT EXISTS user_mgmt DEFAULT CHARSET utf8mb4;
USE user_mgmt;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    deleted TINYINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 插入测试数据
INSERT INTO users (username, password, email, phone) VALUES
('admin', MD5('123456'), 'admin@example.com', '13800000001'),
('zhangsan', MD5('123456'), 'zhang@example.com', '13800000002'),
('lisi', MD5('123456'), 'li@example.com', '13800000003');
```

**1.2 创建 Maven 项目并添加依赖**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>user-management</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Servlet API（Tomcat 自带，scope=provided）-->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>

        <!-- JSP API（如果你用 JSP 页面）-->
        <dependency>
            <groupId>javax.servlet.jsp</groupId>
            <artifactId>javax.servlet.jsp-api</artifactId>
            <version>2.3.3</version>
            <scope>provided</scope>
        </dependency>

        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>

        <!-- Jackson JSON（比手拼 JSON 方便太多）-->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.2</version>
        </dependency>
    </dependencies>
</project>
```

### Step 2：实现 Entity 层

**User.java**

```java
package com.example.usermgmt.entity;

import java.io.Serializable;
import java.sql.Timestamp;

/**
 * 用户实体 —— 对应数据库 users 表
 * 必须实现 Serializable（配合 Session 钝化）
 */
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private int id;
    private String username;
    private String password;   // 存储加密后的密码
    private String email;
    private String phone;
    private int deleted;       // 0=正常, 1=已删除
    private Timestamp createdAt;
    private Timestamp updatedAt;

    public User() {}

    public User(String username, String password, String email, String phone) {
        this.username = username;
        this.password = password;
        this.email = email;
        this.phone = phone;
    }

    // ── Getter / Setter ──
    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public int getDeleted() { return deleted; }
    public void setDeleted(int deleted) { this.deleted = deleted; }
    public Timestamp getCreatedAt() { return createdAt; }
    public void setCreatedAt(Timestamp createdAt) { this.createdAt = createdAt; }
    public Timestamp getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Timestamp updatedAt) { this.updatedAt = updatedAt; }
}
```

**PageResult.java**

```java
package com.example.usermgmt.entity;

import java.util.List;

/** 分页查询结果 */
public class PageResult<T> {
    private int page;        // 当前页
    private int pageSize;    // 每页大小
    private long total;      // 总记录数
    private int totalPages;  // 总页数
    private List<T> list;    // 当前页数据

    public PageResult(int page, int pageSize, long total, List<T> list) {
        this.page = page;
        this.pageSize = pageSize;
        this.total = total;
        this.totalPages = (int) Math.ceil((double) total / pageSize);
        this.list = list;
    }

    public int getPage() { return page; }
    public int getPageSize() { return pageSize; }
    public long getTotal() { return total; }
    public int getTotalPages() { return totalPages; }
    public List<T> getList() { return list; }
}
```

### Step 3：实现工具层与数据访问层

**DBUtil.java**

```java
package com.example.usermgmt.util;

import java.sql.*;

public class DBUtil {
    // 生产环境应使用连接池（Druid/HikariCP），这里简化演示
    private static final String URL =
        "jdbc:mysql://localhost:3306/user_mgmt?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8";
    private static final String USER = "root";
    private static final String PASSWORD = "123456";

    static {
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("MySQL 驱动加载失败", e);
        }
    }

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }

    public static void close(Connection conn, Statement stmt, ResultSet rs) {
        try { if (rs != null) rs.close(); } catch (SQLException e) { /* ignored */ }
        try { if (stmt != null) stmt.close(); } catch (SQLException e) { /* ignored */ }
        try { if (conn != null) conn.close(); } catch (SQLException e) { /* ignored */ }
    }
}
```

**UserDao.java（接口）**

```java
package com.example.usermgmt.dao;

import com.example.usermgmt.entity.User;
import java.util.List;

public interface UserDao {
    int save(User user);
    User findById(int id);
    User findByUsername(String username);
    List<User> findAll(String keyword, int page, int pageSize);
    long count(String keyword);
    int update(User user);
    int deleteById(int id);
}
```

**UserDaoImpl.java（JDBC 实现）**

```java
package com.example.usermgmt.dao.impl;

import com.example.usermgmt.dao.UserDao;
import com.example.usermgmt.entity.User;
import com.example.usermgmt.util.DBUtil;

import java.sql.*;
import java.util.*;

public class UserDaoImpl implements UserDao {

    @Override
    public int save(User user) {
        String sql = "INSERT INTO users (username, password, email, phone) VALUES (?, ?, ?, ?)";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getPassword());
            ps.setString(3, user.getEmail());
            ps.setString(4, user.getPhone());
            ps.executeUpdate();
            ResultSet rs = ps.getGeneratedKeys();
            if (rs.next()) return rs.getInt(1);
        } catch (SQLException e) { e.printStackTrace(); }
        return -1;
    }

    @Override
    public User findById(int id) {
        String sql = "SELECT * FROM users WHERE id = ? AND deleted = 0";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) return mapRow(rs);
        } catch (SQLException e) { e.printStackTrace(); }
        return null;
    }

    @Override
    public User findByUsername(String username) {
        String sql = "SELECT * FROM users WHERE username = ? AND deleted = 0";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, username);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) return mapRow(rs);
        } catch (SQLException e) { e.printStackTrace(); }
        return null;
    }

    @Override
    public List<User> findAll(String keyword, int page, int pageSize) {
        List<User> list = new ArrayList<>();
        StringBuilder sql = new StringBuilder("SELECT * FROM users WHERE deleted = 0");
        if (keyword != null && !keyword.trim().isEmpty()) {
            sql.append(" AND (username LIKE ? OR email LIKE ?)");
        }
        sql.append(" ORDER BY id DESC LIMIT ?, ?");

        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql.toString())) {
            int paramIndex = 1;
            if (keyword != null && !keyword.trim().isEmpty()) {
                String pattern = "%" + keyword.trim() + "%";
                ps.setString(paramIndex++, pattern);
                ps.setString(paramIndex++, pattern);
            }
            ps.setInt(paramIndex++, (page - 1) * pageSize);
            ps.setInt(paramIndex, pageSize);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) list.add(mapRow(rs));
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }

    @Override
    public long count(String keyword) {
        StringBuilder sql = new StringBuilder("SELECT COUNT(*) FROM users WHERE deleted = 0");
        if (keyword != null && !keyword.trim().isEmpty()) {
            sql.append(" AND (username LIKE ? OR email LIKE ?)");
        }
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql.toString())) {
            if (keyword != null && !keyword.trim().isEmpty()) {
                String pattern = "%" + keyword.trim() + "%";
                ps.setString(1, pattern);
                ps.setString(2, pattern);
            }
            ResultSet rs = ps.executeQuery();
            if (rs.next()) return rs.getLong(1);
        } catch (SQLException e) { e.printStackTrace(); }
        return 0;
    }

    @Override
    public int update(User user) {
        String sql = "UPDATE users SET username=?, email=?, phone=? WHERE id=? AND deleted=0";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getEmail());
            ps.setString(3, user.getPhone());
            ps.setInt(4, user.getId());
            return ps.executeUpdate();
        } catch (SQLException e) { e.printStackTrace(); }
        return 0;
    }

    @Override
    public int deleteById(int id) {
        String sql = "UPDATE users SET deleted = 1 WHERE id = ?";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            return ps.executeUpdate();
        } catch (SQLException e) { e.printStackTrace(); }
        return 0;
    }

    private User mapRow(ResultSet rs) throws SQLException {
        User user = new User();
        user.setId(rs.getInt("id"));
        user.setUsername(rs.getString("username"));
        user.setPassword(rs.getString("password"));
        user.setEmail(rs.getString("email"));
        user.setPhone(rs.getString("phone"));
        user.setDeleted(rs.getInt("deleted"));
        user.setCreatedAt(rs.getTimestamp("created_at"));
        user.setUpdatedAt(rs.getTimestamp("updated_at"));
        return user;
    }
}
```

### Step 4：实现 Service 层

**UserService.java（接口）**

```java
package com.example.usermgmt.service;

import com.example.usermgmt.entity.PageResult;
import com.example.usermgmt.entity.User;

public interface UserService {
    /** 注册，返回结果消息（含成功/失败） */
    String register(String username, String password, String email, String phone);

    /** 登录，返回结果消息 */
    String login(String username, String password);

    /** 分页查询用户列表（支持关键词搜索）*/
    PageResult<User> getUserPage(String keyword, int page, int pageSize);

    /** 查询用户详情 */
    User getUserDetail(int id);

    /** 更新用户 */
    String updateUser(int id, String username, String email, String phone);

    /** 删除用户 */
    String deleteUser(int id);
}
```

**UserServiceImpl.java**

```java
package com.example.usermgmt.service.impl;

import com.example.usermgmt.dao.UserDao;
import com.example.usermgmt.dao.impl.UserDaoImpl;
import com.example.usermgmt.entity.PageResult;
import com.example.usermgmt.entity.User;
import com.example.usermgmt.service.UserService;

import java.security.MessageDigest;
import java.util.List;

public class UserServiceImpl implements UserService {

    private final UserDao userDao = new UserDaoImpl();

    @Override
    public String register(String username, String password, String email, String phone) {
        // ── 参数校验 ──
        if (isBlank(username)) return "用户名不能为空";
        if (username.length() < 3 || username.length() > 20) return "用户名长度需要 3-20 个字符";
        if (isBlank(password) || password.length() < 6) return "密码至少 6 位";
        if (userDao.findByUsername(username) != null) return "用户名已存在";

        User user = new User(username, md5(password), email, phone);
        int id = userDao.save(user);
        return id > 0 ? "注册成功" : "注册失败，请重试";
    }

    @Override
    public String login(String username, String password) {
        if (isBlank(username) || isBlank(password)) return "用户名和密码不能为空";

        User user = userDao.findByUsername(username);
        if (user == null) return "用户不存在";
        if (!md5(password).equals(user.getPassword())) return "密码错误";
        return "登录成功";
    }

    @Override
    public PageResult<User> getUserPage(String keyword, int page, int pageSize) {
        // 参数修正
        if (page < 1) page = 1;
        if (pageSize < 1) pageSize = 10;
        long total = userDao.count(keyword);
        List<User> list = userDao.findAll(keyword, page, pageSize);
        return new PageResult<>(page, pageSize, total, list);
    }

    @Override
    public User getUserDetail(int id) {
        return userDao.findById(id);
    }

    @Override
    public String updateUser(int id, String username, String email, String phone) {
        User exist = userDao.findById(id);
        if (exist == null) return "用户不存在";

        // 检查新用户名是否被别人占用
        User nameHolder = userDao.findByUsername(username);
        if (nameHolder != null && nameHolder.getId() != id) return "用户名已被占用";

        if (isBlank(username) || username.length() < 3) return "用户名格式不正确";

        User user = new User();
        user.setId(id);
        user.setUsername(username);
        user.setEmail(email);
        user.setPhone(phone);
        return userDao.update(user) > 0 ? "更新成功" : "更新失败";
    }

    @Override
    public String deleteUser(int id) {
        if (userDao.findById(id) == null) return "用户不存在";
        return userDao.deleteById(id) > 0 ? "删除成功" : "删除失败";
    }

    // ── 工具方法 ──
    private boolean isBlank(String s) { return s == null || s.trim().isEmpty(); }

    private String md5(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(input.getBytes("UTF-8"));
            StringBuilder sb = new StringBuilder();
            for (byte b : digest) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

### Step 5：实现 Filter 层

**EncodingFilter.java**

```java
package com.example.usermgmt.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebFilter("/*")
public class EncodingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;

        req.setCharacterEncoding("UTF-8");
        resp.setContentType("application/json;charset=UTF-8");

        chain.doFilter(request, response);
    }
}
```

**LoginCheckFilter.java**

```java
package com.example.usermgmt.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;

/** 登录拦截器 —— 需要登录的接口必须在 Session 中有用户信息 */
@WebFilter("/api/user/*")
public class LoginCheckFilter implements Filter {

    // 不需要登录的路径
    private static final String[] PUBLIC_API = {
        "/api/user/login",   // 登录接口
        "/api/user/register" // 注册接口
    };

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;
        String path = req.getRequestURI();

        // 公开接口放行
        for (String api : PUBLIC_API) {
            if (path.contains(api)) {
                chain.doFilter(request, response);
                return;
            }
        }

        // 检查登录
        HttpSession session = req.getSession(false);
        if (session != null && session.getAttribute("loginUser") != null) {
            chain.doFilter(request, response);
        } else {
            resp.setStatus(401);
            resp.getWriter().write("{\"code\":401, \"message\":\"请先登录\"}");
        }
    }
}
```

### Step 6：实现 Listener 层

**AppStartupListener.java**

```java
package com.example.usermgmt.listener;

import javax.servlet.*;
import javax.servlet.annotation.WebListener;

@WebListener
public class AppStartupListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ServletContext ctx = sce.getServletContext();
        ctx.setAttribute("appName", "用户管理系统 V1.0");
        ctx.setAttribute("startupTime", System.currentTimeMillis());
        System.out.println("[启动] 用户管理系统已启动, contextPath=" + ctx.getContextPath());
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        long uptime = System.currentTimeMillis()
            - (Long) sce.getServletContext().getAttribute("startupTime");
        System.out.println("[关闭] 应用运行时长: " + (uptime / 60000) + " 分钟");
    }
}
```

**OnlineCountListener.java**

```java
package com.example.usermgmt.listener;

import javax.servlet.*;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.*;
import java.util.concurrent.atomic.AtomicInteger;

@WebListener
public class OnlineCountListener implements HttpSessionListener {
    private static final AtomicInteger count = new AtomicInteger(0);

    @Override
    public void sessionCreated(HttpSessionEvent se) {
        se.getSession().getServletContext()
            .setAttribute("onlineCount", count.incrementAndGet());
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        se.getSession().getServletContext()
            .setAttribute("onlineCount", count.decrementAndGet());
    }
}
```

### Step 7：实现 Controller 层

**UserController.java**（核心文件，统一路由入口）

```java
package com.example.usermgmt.controller;

import com.example.usermgmt.entity.PageResult;
import com.example.usermgmt.entity.User;
import com.example.usermgmt.service.UserService;
import com.example.usermgmt.service.impl.UserServiceImpl;
import com.fasterxml.jackson.databind.ObjectMapper;

import javax.servlet.*;
import javax.servlet.http.*;
import javax.servlet.annotation.*;
import java.io.*;
import java.util.*;

@WebServlet("/api/user/*")
public class UserController extends HttpServlet {
    private final UserService userService = new UserServiceImpl();
    private final ObjectMapper jsonMapper = new ObjectMapper();

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
        String pathInfo = req.getPathInfo();
        String method = req.getMethod();

        try {
            switch (getRouteKey(pathInfo, method)) {
                case "POST:/register" -> handleRegister(req, resp);
                case "POST:/login"    -> handleLogin(req, resp);
                case "GET:/list"      -> handleList(req, resp);
                case "GET:/detail"    -> handleDetail(req, resp);
                case "PUT:/update"    -> handleUpdate(req, resp);
                case "DELETE:/delete" -> handleDelete(req, resp);
                case "GET:/logout"    -> handleLogout(req, resp);
                default               -> writeJson(resp, 404, "接口不存在");
            }
        } catch (Exception e) {
            e.printStackTrace();
            writeJson(resp, 500, "服务器错误: " + e.getMessage());
        }
    }

    // ── ① 注册 ──
    private void handleRegister(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        String username = req.getParameter("username");
        String password = req.getParameter("password");
        String email = req.getParameter("email");
        String phone = req.getParameter("phone");
        String result = userService.register(username, password, email, phone);

        if ("注册成功".equals(result)) {
            writeJson(resp, 200, result);
        } else {
            writeJson(resp, 400, result);
        }
    }

    // ── ② 登录 ──
    private void handleLogin(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        String username = req.getParameter("username");
        String password = req.getParameter("password");
        String result = userService.login(username, password);

        if ("登录成功".equals(result)) {
            // 登录成功 → 把用户信息存入 Session
            User user = new User(); // 简化：实际应该从 DB 加载完整的用户对象
            user.setUsername(username);
            req.getSession().setAttribute("loginUser", user);
            writeJson(resp, 200, result);
        } else {
            writeJson(resp, 401, result);
        }
    }

    // ── ③ 用户列表（分页+搜索）──
    private void handleList(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        String keyword = req.getParameter("keyword");
        int page = parseInt(req.getParameter("page"), 1);
        int pageSize = parseInt(req.getParameter("pageSize"), 10);
        PageResult<User> result = userService.getUserPage(keyword, page, pageSize);
        writeJson(resp, 200, result);
    }

    // ── ④ 用户详情 ──
    private void handleDetail(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        int id = parseInt(req.getParameter("id"), -1);
        if (id < 0) { writeJson(resp, 400, "缺少 id 参数"); return; }
        User user = userService.getUserDetail(id);
        if (user != null) writeJson(resp, 200, user);
        else writeJson(resp, 404, "用户不存在");
    }

    // ── ⑤ 更新用户 ──
    private void handleUpdate(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        int id = parseInt(req.getParameter("id"), -1);
        String username = req.getParameter("username");
        String email = req.getParameter("email");
        String phone = req.getParameter("phone");
        String result = userService.updateUser(id, username, email, phone);
        writeJson(resp, "更新成功".equals(result) ? 200 : 400, result);
    }

    // ── ⑥ 删除用户 ──
    private void handleDelete(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        int id = parseInt(req.getParameter("id"), -1);
        String result = userService.deleteUser(id);
        writeJson(resp, "删除成功".equals(result) ? 200 : 400, result);
    }

    // ── ⑦ 退出登录 ──
    private void handleLogout(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        HttpSession session = req.getSession(false);
        if (session != null) session.invalidate();
        writeJson(resp, 200, "已退出");
    }

    // ── 工具方法 ──
    private String getRouteKey(String pathInfo, String method) {
        return method.toUpperCase() + ":" + (pathInfo != null ? pathInfo : "");
    }

    private int parseInt(String s, int defaultVal) {
        try { return Integer.parseInt(s); } catch (Exception e) { return defaultVal; }
    }

    private void writeJson(HttpServletResponse resp, int code, Object data)
            throws IOException {
        Map<String, Object> result = new LinkedHashMap<>();
        result.put("code", code);
        result.put("data", data);
        resp.getWriter().write(jsonMapper.writeValueAsString(result));
    }
}
```

### Step 8：前端页面

**webapp/WEB-INF/web.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <display-name>用户管理系统</display-name>

    <welcome-file-list>
        <welcome-file>login.html</welcome-file>
    </welcome-file-list>

    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>
</web-app>
```

**webapp/login.html**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>用户管理系统 - 登录</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            display: flex; justify-content: center; align-items: center;
            min-height: 100vh; background: #f0f2f5; font-family: sans-serif;
        }
        .login-box {
            background: white; padding: 40px; border-radius: 8px;
            box-shadow: 0 2px 12px rgba(0,0,0,0.1); width: 360px;
        }
        .login-box h2 { text-align: center; margin-bottom: 30px; color: #333; }
        .form-group { margin-bottom: 20px; }
        .form-group label { display: block; margin-bottom: 6px; color: #666; font-size: 14px; }
        .form-group input {
            width: 100%; padding: 10px 12px; border: 1px solid #ddd;
            border-radius: 4px; font-size: 14px;
        }
        .form-group input:focus { outline: none; border-color: #409eff; }
        .btn {
            width: 100%; padding: 10px; background: #409eff; color: white;
            border: none; border-radius: 4px; font-size: 16px; cursor: pointer;
        }
        .btn:hover { background: #3a8ee6; }
        .tip { text-align: center; margin-top: 15px; color: #999; font-size: 14px; }
        .tip a { color: #409eff; text-decoration: none; }
        .error { color: #f56c6c; font-size: 13px; margin-top: 8px; display: none; }
    </style>
</head>
<body>
<div class="login-box">
    <h2>用户管理系统</h2>
    <form id="loginForm">
        <div class="form-group">
            <label>用户名</label>
            <input type="text" name="username" id="username" placeholder="请输入用户名" required>
        </div>
        <div class="form-group">
            <label>密码</label>
            <input type="password" name="password" id="password" placeholder="请输入密码" required>
        </div>
        <button type="submit" class="btn">登 录</button>
        <div class="error" id="errorMsg"></div>
    </form>
    <p class="tip">还没有账号？<a href="register.html">去注册</a></p>
</div>

<script>
    document.getElementById('loginForm').addEventListener('submit', async (e) => {
        e.preventDefault();
        const username = document.getElementById('username').value;
        const password = document.getElementById('password').value;

        try {
            const params = new URLSearchParams();
            params.append('username', username);
            params.append('password', password);

            const resp = await fetch('/user-management/api/user/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: params
            });
            const result = await resp.json();

            if (result.code === 200) {
                // 登录成功 → 简单跳转到控制台页面（自己写 console.html）
                window.location.href = 'list.html';
            } else {
                showError(result.data || '登录失败');
            }
        } catch (err) {
            showError('网络错误，请重试');
        }
    });

    function showError(msg) {
        const el = document.getElementById('errorMsg');
        el.textContent = msg;
        el.style.display = 'block';
    }
</script>
</body>
</html>
```

**webapp/register.html**（结构同 login.html，增加邮箱和手机号字段）

```html
<!-- 注册页：比登录页多 email 和 phone 两个字段 -->
<!-- POST /api/user/register
参数: username, password, email, phone -->
<!-- 代码结构与 login.html 类似，省略具体实现 -->
```

> 💡 **AI 辅助用法**：把 login.html 的代码发给 AI，让它帮你生成 register.html 和 list.html。

### Step 9：部署、测试与验证

**部署步骤：**

1. IDEA → Run → Edit Configurations → Tomcat Server → Deployment → 添加 Artifact
2. 启动 Tomcat，访问 `http://localhost:8080/user-management/`
3. 看到登录页 → 注册一个账号 → 登录 → 测试增删改查

**API 测试清单：**

| 测试项 | 请求 | URL | 预期结果 |
|--------|------|-----|---------|
| 注册成功 | POST | `/api/user/register?username=test&password=123456` | {"code":200} |
| 用户名重复 | POST | 再次注册 test | {"code":400,"data":"用户名已存在"} |
| 登录成功 | POST | `/api/user/login?username=test&password=123456` | {"code":200} |
| 密码错误 | POST | 登录密码错 | {"code":401} |
| 查列表 | GET | `/api/user/list?page=1&pageSize=5` | {"code":200,"data":{"list":[...]}} |
| 查列表（未登录） | GET | 清除 Cookie 后查列表 | {"code":401} |
| 出退登录 | GET | `/api/user/logout` | {"code":200} |

---

## 验证标准

- [ ] 能注册新用户，用户名重复时返回错误
- [ ] 能登录，密码错误时返回 401
- [ ] 登录后能查看用户列表（分页+搜索）
- [ ] 未登录时访问列表返回 401（Filter 正常工作）
- [ ] 中文不乱码（EncodingFilter 正常工作）
- [ ] /api/user/list 返回的是 JSON（Content-Type 在 Filter 中统一设置）
- [ ] 退出登录后 Session 销毁

---

## 思考题

1. 为什么 Filter 中的 `resp.setContentType("application/json;charset=UTF-8")` 对所有 API 都生效？如果有个接口要返回 HTML 怎么办？
2. 如果用户量很大（百万级），你会优化 Dao 层的哪些 SQL 和索引策略？用 AI 分析你的答案。
3. 在纯 Servlet 项目中，`new UserDaoImpl()` 写在 Service 里，在 Spring Boot 中是怎么去掉这个 `new` 的？（提示：DI / IoC）
4. 如果把这个系统的 "Session 登录态" 改成 "JWT Token 无状态登录"，需要改哪些地方？Service 层需要动吗？

---

## 常见问题

**Q：Maven 的 dependency scope=provided 是什么意思？**
A：编译时需要这个 jar，但部署到 Tomcat 时不包括（Tomcat 自带 Servlet API jar）。

**Q：访问 /api/user/list 返回 404？**
A：检查 `@WebServlet("/api/user/*")` 和请求路径是否匹配，Tomcat 的 contextPath 是否配对。

**Q：返回的中文是乱码？**
A：检查 EncodingFilter 是否在 `/*` 上生效，`setCharacterEncoding` 是否在 `doFilter` 最前面调用。

---

*上一节理论：[03.4 Filter与Listener](../theory/03.4-filter-and-listener.md) | 下一章：[04-AI Agent入门](../../04-introduction-to-ai-agent/theory/)*
