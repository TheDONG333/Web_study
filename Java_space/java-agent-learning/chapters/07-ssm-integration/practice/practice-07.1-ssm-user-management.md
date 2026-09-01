# 实操 07.1：SSM 完整整合 — 简单用户管理系统

> ⏱ **预计时长**：120 分钟（排错时间不算😄）
> 📌 **难度**：⭐⭐⭐⭐
> 🔧 **AI 辅助**：整合报错最多，把**完整堆栈**发给 AI 是最快的排错方式

---

## 前置要求

- ✅ 已读完理论 07.1（整合全景）
- ✅ 已掌握：Spring（05）、SpringMVC（06）、MyBatis（04）
- ✅ 环境：JDK 26、Maven、MySQL 8+、IDEA + Tomcat 10.1

## 项目目标

把三个框架拼成一个 web 项目，实现**用户登录、新增、删除、分页查询**，页面用 JSP。功能优先，不做精美样式。

---

## 一、数据库准备

```sql
CREATE DATABASE IF NOT EXISTS agent_learn DEFAULT CHARSET utf8mb4;
USE agent_learn;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20)
);

INSERT INTO users (username, password, email, phone) VALUES
('admin', '123456', 'admin@example.com', '13800000001'),
('zhangsan', '123456', 'zhang@example.com', '13800000002'),
('lisi', '123456', 'li@example.com', '13800000003'),
('wangwu', '123456', 'wang@example.com', '13800000004'),
('zhaoliu', '123456', 'zhao@example.com', '13800000005');
```

---

## 二、项目结构（严格分层）

```
ssm-user-management/
├── pom.xml
└── src/main/
    ├── java/com/demo/
    │   ├── controller/UserController.java        ← web 层（只收参数、调 Service、放数据）
    │   ├── service/
    │   │   ├── UserService.java                   ← 业务接口
    │   │   └── impl/UserServiceImpl.java          ← 业务实现（事务在这层）
    │   ├── mapper/UserMapper.java                 ← 数据层接口
    │   ├── entity/User.java
    │   └── interceptor/LoginInterceptor.java      ← 登录拦截器
    ├── resources/
    │   ├── spring-context.xml                     ← ① 父容器：IOC、事务、MyBatis
    │   ├── spring-mvc.xml                         ← ② 子容器：Controller、视图
    │   └── mapper/UserMapper.xml                  ← ③ Mapper 映射文件
    └── webapp/
        ├── WEB-INF/
        │   ├── web.xml                            ← DispatcherServlet 入口
        │   └── jsp/
        │       ├── login.jsp                      ← 登录页
        │       └── list.jsp                       ← 列表页（分页+新增+删除）
        └── index.jsp
```

---

## 三、pom.xml（JDK 26 兼容版本，直接抄）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>ssm-user-management</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>26</maven.compiler.source>
        <maven.compiler.target>26</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- ⚠️ Spring 6.2.x：支持 JDK 17-26，用最新小版本 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>6.2.10</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>6.2.10</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>6.2.10</version>
        </dependency>

        <!-- ⚠️ jakarta 不是 javax！Spring 6 用 jakarta.servlet -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
        <!-- JSP（Tomcat 10.1 自带实现，provided 即可） -->
        <dependency>
            <groupId>jakarta.servlet.jsp</groupId>
            <artifactId>jakarta.servlet.jsp-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>

        <!-- MyBatis + Spring 6 需要 3.0.x -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.19</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>3.0.4</version>
        </dependency>

        <!-- ⚠️ 新版 MySQL 驱动 artifactId 是 mysql-connector-j -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
        </dependency>

        <!-- JSON + 日志 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.18.2</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.5.18</version>
        </dependency>
    </dependencies>
</project>
```

> ⚠️ 如果 6.2.10 报 `Unsupported class file major version`，把 Spring 版本升级到更新的 6.2.x 小版本。

---

## 四、web.xml（DispatcherServlet 入口）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- ⚠️ jakarta 命名空间 -->
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
         https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">

    <display-name>ssm-user-management</display-name>

    <!-- ① 启动时加载 spring-context.xml（父容器：Service/Mapper/事务） -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-context.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- ② DispatcherServlet（子容器：Controller） -->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <!-- ③ 编码过滤器（回忆 03.4：Filter 统一编码） -->
    <filter>
        <filter-name>encoding</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>encoding</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

</web-app>
```

---

## 五、spring-context.xml（父容器：IOC + 事务 + MyBatis）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/tx
           http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- ① 扫描 Service/Mapper 等（⚠️ 不扫 controller，留给 spring-mvc.xml） -->
    <context:component-scan base-package="com.demo">
        <context:exclude-filter type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <!-- ② 数据源 -->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url"
            value="jdbc:mysql://localhost:3306/agent_learn?useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </bean>

    <!-- ③ SqlSessionFactory 交给 Spring -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
        <property name="configuration">
            <!-- 内联 MyBatis 配置（替代 mybatis-config.xml，也支持驼峰映射） -->
            <bean class="org.apache.ibatis.session.Configuration">
                <property name="mapUnderscoreToCamelCase" value="true"/>
            </bean>
        </property>
    </bean>

    <!-- ④ 扫描 Mapper 接口 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.demo.mapper"/>
    </bean>

    <!-- ⑤ 事务管理 -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <tx:annotation-driven transaction-manager="transactionManager"/>

</beans>
```

> 💡 为什么没有 mybatis-config.xml？整合后 MyBatis 的全局设置可以通过 SqlSessionFactoryBean 的 `configuration` 属性内联配置（如上），少一份配置文件。想单独用 mybatis-config.xml 也可以：加 `<property name="configLocation" value="classpath:mybatis-config.xml"/>`。

---

## 六、spring-mvc.xml（子容器：Controller + 视图）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/mvc
           http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!-- ① 只扫描 Controller -->
    <context:component-scan base-package="com.demo.controller"/>

    <!-- ② 注解驱动（@RequestMapping/@ResponseBody 等） -->
    <mvc:annotation-driven/>

    <!-- ③ 视图解析器：返回 "login" → /WEB-INF/jsp/login.jsp -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <!-- ④ 静态资源放行 -->
    <mvc:default-servlet-handler/>

    <!-- ⑤ 登录拦截器（放行登录页和登录接口） -->
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <mvc:exclude-mapping path="/login"/>
            <mvc:exclude-mapping path="/doLogin"/>
            <mvc:exclude-mapping path="/css/**"/>
            <mvc:exclude-mapping path="/js/**"/>
            <bean class="com.demo.interceptor.LoginInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>

</beans>
```

---

## 七、Java 代码（分层）

### entity/User.java（字段：id/username/password/email/phone + getter/setter，复制之前章节的即可）

```java
package com.demo.entity;

public class User {
    private Integer id;
    private String username;
    private String password;
    private String email;
    private String phone;

    public User() {}

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
}
```

### mapper/UserMapper.java（数据层接口）

```java
package com.demo.mapper;

import com.demo.entity.User;
import java.util.List;

public interface UserMapper {
    User findByUsername(String username);
    List<User> findPage(int offset, int pageSize);
    long count();
    int insert(User user);
    int deleteById(int id);
}
```

### resources/mapper/UserMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC
    "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- namespace = 接口全类名 -->
<mapper namespace="com.demo.mapper.UserMapper">

    <select id="findByUsername" parameterType="string" resultType="com.demo.entity.User">
        SELECT * FROM users WHERE username = #{username}
    </select>

    <select id="findPage" resultType="com.demo.entity.User">
        SELECT * FROM users ORDER BY id LIMIT #{offset}, #{pageSize}
    </select>

    <select id="count" resultType="long">
        SELECT COUNT(*) FROM users
    </select>

    <insert id="insert" parameterType="com.demo.entity.User"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (username, password, email, phone)
        VALUES (#{username}, #{password}, #{email}, #{phone})
    </insert>

    <delete id="deleteById" parameterType="int">
        DELETE FROM users WHERE id = #{id}
    </delete>

</mapper>
```

### service/UserService.java + impl/UserServiceImpl.java（业务层 + 事务）

```java
package com.demo.service;

import com.demo.entity.User;
import java.util.List;

public interface UserService {
    boolean login(String username, String password);
    List<User> getPage(int page, int pageSize);
    long getTotalCount();
    String addUser(User user);
    String deleteUser(int id);
}
```

```java
package com.demo.service.impl;

import com.demo.entity.User;
import com.demo.mapper.UserMapper;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public boolean login(String username, String password) {
        User user = userMapper.findByUsername(username);
        return user != null && user.getPassword().equals(password);
    }

    @Override
    public List<User> getPage(int page, int pageSize) {
        int offset = (page - 1) * pageSize;
        return userMapper.findPage(offset, pageSize);
    }

    @Override
    public long getTotalCount() {
        return userMapper.count();
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public String addUser(User user) {
        if (user.getUsername() == null || user.getUsername().isEmpty()) {
            return "用户名不能为空";
        }
        if (userMapper.findByUsername(user.getUsername()) != null) {
            return "用户名已存在";
        }
        return userMapper.insert(user) > 0 ? "新增成功" : "新增失败";
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public String deleteUser(int id) {
        return userMapper.deleteById(id) > 0 ? "删除成功" : "删除失败";
    }
}
```

### interceptor/LoginInterceptor.java（⚠️ 注意 jakarta 包名）

```java
package com.demo.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.web.servlet.HandlerInterceptor;

public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        // 已登录（Session 有 user）就放行
        if (request.getSession().getAttribute("user") != null) {
            return true;
        }
        // 未登录 → 回登录页
        response.sendRedirect(request.getContextPath() + "/login");
        return false;
    }
}
```

### controller/UserController.java（web 层，只调 Service）

```java
package com.demo.controller;

import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Controller
public class UserController {

    @Autowired
    private UserService userService;

    // 登录页
    @GetMapping("/login")
    public String loginPage() {
        return "login";
    }

    // 登录动作（POST，成功后重定向到列表）
    @PostMapping("/doLogin")
    public String doLogin(String username, String password, HttpSession session) {
        if (userService.login(username, password)) {
            session.setAttribute("user", username);
            return "redirect:/user/list";   // 重定向：防重复提交
        }
        return "redirect:/login?error=1";   // 失败带标记
    }

    // 用户列表（分页）
    @GetMapping("/user/list")
    public String list(@RequestParam(defaultValue = "1") int page,
                       @RequestParam(defaultValue = "5") int pageSize,
                       Model model) {
        List<User> users = userService.getPage(page, pageSize);
        long total = userService.getTotalCount();
        int totalPages = (int) Math.ceil((double) total / pageSize);

        model.addAttribute("users", users);
        model.addAttribute("page", page);
        model.addAttribute("totalPages", totalPages);
        model.addAttribute("total", total);
        return "list";
    }

    // 新增用户
    @PostMapping("/user/add")
    public String add(User user) {
        userService.addUser(user);
        return "redirect:/user/list";
    }

    // 删除用户
    @GetMapping("/user/delete/{id}")
    public String delete(@PathVariable("id") Integer id) {
        userService.deleteUser(id);
        return "redirect:/user/list";
    }

    // 退出登录
    @GetMapping("/logout")
    public String logout(HttpSession session) {
        session.invalidate();
        return "redirect:/login";
    }
}
```

---

## 八、JSP 页面（功能优先，不求美观）

### webapp/WEB-INF/jsp/login.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
    <h2>用户登录</h2>
    <% if (request.getParameter("error") != null) { %>
        <p style="color:red">用户名或密码错误</p>
    <% } %>
    <form action="${pageContext.request.contextPath}/doLogin" method="post">
        用户名: <input type="text" name="username"><br><br>
        密码: <input type="password" name="password"><br><br>
        <button type="submit">登录</button>
    </form>
</body>
</html>
```

### webapp/WEB-INF/jsp/list.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="com.demo.entity.User, java.util.List" %>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>用户列表</title>
</head>
<body>
    <h2>用户列表</h2>
    <p>当前用户: ${sessionScope.user}
       | 总记录数: ${total}
       | <a href="${pageContext.request.contextPath}/logout">退出</a></p>

    <!-- 新增表单 -->
    <form action="${pageContext.request.contextPath}/user/add" method="post">
        用户名: <input type="text" name="username" required>
        密码: <input type="password" name="password" required>
        邮箱: <input type="text" name="email">
        <button type="submit">新增</button>
    </form>

    <hr>

    <!-- 用户表格 -->
    <table border="1" cellpadding="5" cellspacing="0">
        <tr>
            <th>ID</th><th>用户名</th><th>邮箱</th><th>电话</th><th>操作</th>
        </tr>
        <%
            List<User> users = (List<User>) request.getAttribute("users");
            if (users != null) {
                for (User u : users) {
        %>
        <tr>
            <td><%= u.getId() %></td>
            <td><%= u.getUsername() %></td>
            <td><%= u.getEmail() %></td>
            <td><%= u.getPhone() %></td>
            <td>
                <a href="${pageContext.request.contextPath}/user/delete/<%= u.getId() %>"
                   onclick="return confirm('确定删除?')">删除</a>
            </td>
        </tr>
        <%  } } %>
    </table>

    <hr>

    <!-- 分页 -->
    <p>
        第 ${page} / ${totalPages} 页
        <% if ((Integer) request.getAttribute("page") > 1) { %>
            <a href="${pageContext.request.contextPath}/user/list?page=${page - 1}">上一页</a>
        <% } %>
        <% if ((Integer) request.getAttribute("page") < (Integer) request.getAttribute("totalPages")) { %>
            <a href="${pageContext.request.contextPath}/user/list?page=${page + 1}">下一页</a>
        <% } %>
    </p>
</body>
</html>
```

---

## 九、部署测试

**IDEA 部署（Tomcat 10.1）：**

1. Run → Edit Configurations → Tomcat Server → Local
2. Deployment → 添加 Artifact（war exploded）→ Application context 设为 `/ssm`
3. 启动，访问 `http://localhost:8080/ssm/login`

**测试流程：**

| 步骤 | 操作 | 预期 |
|------|------|------|
| ① | 直接访问 `/ssm/user/list` | 被拦截器弹回登录页 |
| ② | 输入 admin / 123456 | 登录成功进列表页 |
| ③ | 列表显示 5 条数据 + 分页 | 第 1 页 5 条，可翻页 |
| ④ | 新增用户 test/123456 | 列表出现新用户 |
| ⑤ | 删除刚才的用户 | 列表消失 |
| ⑥ | 错误密码登录 | 显示"用户名或密码错误" |
| ⑦ | 点退出 | 回到登录页，再访问列表被拦截 |

---

## 十、常见整合报错排查（本章重点！）

| 报错 | 原因 | 解决 |
|------|------|------|
| `Unsupported class file major version 70` | Spring 版本太老（ASM 读不懂 JDK 26 字节码） | Spring 升级到 6.2.x 最新版 |
| `ClassNotFoundException: javax.servlet.*` | 用了 javax 依赖 | 换 jakarta.servlet-api |
| `NoSuchBeanDefinitionException: userService` | spring-context.xml 没扫到 Service，或包路径错 | 检查 component-scan 的 base-package |
| `Invalid bound statement` | mapper.xml 的 namespace/id 和接口不匹配 | namespace=接口全类名 |
| `BeanCreationException: dataSource` | 数据库连不上 | 检查 URL/用户名/密码、MySQL 是否启动 |
| 页面 404 / 返回不了 JSP | 视图解析器前缀后缀错，或 JSP 不在 /WEB-INF/jsp | 检查 spring-mvc.xml 的 prefix/suffix |
| 拦截器拦截了登录页（死循环） | exclude-mapping 没配 /login | 放行 /login 和 /doLogin |
| 中文乱码 | 没配编码过滤器 | web.xml 配 CharacterEncodingFilter |
| 页面能打开但接口 405 | 请求方法和 @GetMapping/@PostMapping 不匹配 | 检查表单 method |

> 💡 **排错套路**：看到异常先看**第一个 Caused by**（根因），把完整堆栈发给 AI 是最快的。

---

## 十一、练习任务（做完理解就到位了）

1. **故意制造一个整合报错**：把 spring-context.xml 的 component-scan 包名改错，启动看报什么错，再改回来
2. **演示扫描重叠**：把 controller 也加入 spring-context.xml 扫描，观察启动日志（会报重复 Bean 或空指针）
3. **加一个功能**：给 User 加一个 `age` 字段，走通 数据库 → Mapper → Service → Controller → JSP 全链路
4. **写事务测试**：给 `addUser` 加一个参数，为 true 时 insert 后抛异常，验证数据库有没有新数据（事务回滚）
5. **思考**：这个项目哪些配置在 Spring Boot 里被 `application.yml` 一行搞定？（数据源？事务？视图？）

---

## 验证标准

- [ ] 登录 / 新增 / 删除 / 分页 四个功能全部跑通
- [ ] 未登录访问列表被拦截器拦住
- [ ] 分层严格：Controller 没有直接调用 Mapper
- [ ] 新增用户的 SQL 在数据库真实生效（查库确认）
- [ ] 遇到过至少一个整合报错并独立解决（这就是本章的价值）
- [ ] 能说出：如果换成 Spring Boot，哪些配置会被自动化

---

*上一节理论：[07.1 SSM 整合全景](../theory/07.1-ssm-integration-overview.md) | 下一章：[08-Spring Boot](../../08-springboot/theory/)*
