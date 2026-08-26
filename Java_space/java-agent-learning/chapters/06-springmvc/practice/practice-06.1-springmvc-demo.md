# 实操 06.1：SpringMVC 接口 demo（GET/POST + 表单/JSON + 返回 JSON）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：报错时把完整堆栈发给 AI 分析

---

## 前置要求

- ✅ 已读完理论 06.1 ~ 06.5
- ✅ 已完成第五章 [Spring 整合 MyBatis demo](practice-05.1-spring-mybatis-demo.md)
- ✅ 需要安装：JDK 17+、Maven、IDEA + Tomcat 9

## 项目目标

写一个 SpringMVC 项目（SSM 的 web 层），提供 5 个接口：
- GET 接口（普通参数 + @PathVariable）
- POST 接口（表单 + 对象接收）
- POST 接口（@RequestBody 接收 JSON）
- 统一返回 JSON（Result 结构）
- 全局异常处理 + 登录拦截器

---

## 项目结构

```
springmvc-demo/
├── pom.xml
└── src/main/
    ├── java/com/demo/
    │   ├── controller/UserController.java
    │   ├── entity/User.java
    │   ├── entity/Result.java
    │   ├── interceptor/LoginInterceptor.java
    │   └── exception/GlobalExceptionHandler.java
    ├── resources/springmvc.xml
    └── webapp/
        ├── WEB-INF/web.xml
        └── index.html
```

---

## Step 1：pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>springmvc-demo</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- SpringMVC（包含 spring-web + spring-webmvc） -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>5.3.31</version>
        </dependency>
        <!-- Servlet API（Tomcat 提供） -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>
        <!-- Jackson：@ResponseBody 转 JSON 依赖它 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.2</version>
        </dependency>
    </dependencies>
</project>
```

> ⚠️ 注意：返回 JSON 必须引入 Jackson（@ResponseBody 自动转 JSON 靠它）。

---

## Step 2：web.xml（配置 DispatcherServlet 入口）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!-- ① 配置 DispatcherServlet（前端控制器） -->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- ② 指定 springmvc.xml 位置 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <!-- ③ Tomcat 启动时就初始化（而不是第一次访问） -->
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- ④ 拦截所有请求（/ 不拦截 jsp） -->
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

---

## Step 3：springmvc.xml（控制器扫描 + 拦截器 + 异常开关）

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

    <!-- ① 扫描 Controller -->
    <context:component-scan base-package="com.demo"/>

    <!-- ② 开启注解支持（@RequestMapping/@ResponseBody 等） -->
    <mvc:annotation-driven/>

    <!-- ③ 注册拦截器（拦截所有，放行 /login 和静态资源） -->
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <mvc:exclude-mapping path="/login"/>
            <mvc:exclude-mapping path="/css/**"/>
            <mvc:exclude-mapping path="/js/**"/>
            <bean class="com.demo.interceptor.LoginInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>

</beans>
```

---

## Step 4：实体类

**User.java：**

```java
package com.demo.entity;

public class User {
    private Integer id;
    private String name;
    private Integer age;
    private String email;

    public User() {}

    public User(Integer id, String name) {
        this.id = id;
        this.name = name;
    }

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

**Result.java（统一返回结构）：**

```java
package com.demo.entity;

public class Result {
    private int code;
    private String message;
    private Object data;

    public static Result ok(Object data) {
        return new Result(200, "操作成功", data);
    }

    public static Result fail(String message) {
        return new Result(500, message, null);
    }

    public Result() {}
    public Result(int code, String message, Object data) {
        this.code = code;
        this.message = message;
        this.data = data;
    }

    public int getCode() { return code; }
    public void setCode(int code) { this.code = code; }
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    public Object getData() { return data; }
    public void setData(Object data) { this.data = data; }
}
```

---

## Step 5：UserController（核心，5 个接口）

```java
package com.demo.controller;

import com.demo.entity.Result;
import com.demo.entity.User;
import org.springframework.web.bind.annotation.*;
import java.util.*;

@RestController
public class UserController {

    // ========== ① GET 普通参数 ==========
    // 请求: GET /demo/user?name=张三&age=20
    @GetMapping("/user")
    public Result getUser(String name, Integer age) {
        return Result.ok("收到参数 name=" + name + ", age=" + age);
    }

    // ========== ② GET @PathVariable（RESTful）==========
    // 请求: GET /demo/user/1
    @GetMapping("/user/{id}")
    public Result getUserById(@PathVariable("id") Integer id) {
        return Result.ok("查询用户 id=" + id);
    }

    // ========== ③ POST 表单 + 对象接收 ==========
    // 请求: POST /demo/user/add, 表单: name=张三&age=20&email=x@y.com
    @PostMapping("/user/add")
    public Result addUser(User user) {
        return Result.ok("新增用户: " + user.getName() + ", " + user.getAge());
    }

    // ========== ④ POST @RequestBody 接收 JSON ==========
    // 请求: POST /demo/user/save
    // Body(JSON): {"name":"张三","age":20,"email":"x@y.com"}
    @PostMapping("/user/save")
    public Result saveUser(@RequestBody User user) {
        return Result.ok("保存用户: " + user.getName());
    }

    // ========== ⑤ 返回 List → JSON 数组 ==========
    @GetMapping("/user/list")
    public Result list() {
        List<User> users = List.of(new User(1, "张三"), new User(2, "李四"));
        return Result.ok(users);
    }
}
```

---

## Step 6：全局异常处理器 + 登录拦截器

**GlobalExceptionHandler.java：**

```java
package com.demo.exception;

import com.demo.entity.Result;
import org.springframework.web.bind.annotation.*;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public Result handleRuntime(RuntimeException e) {
        return Result.fail(e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result handleOther(Exception e) {
        e.printStackTrace();
        return Result.fail("服务器开小差了，请稍后再试");
    }
}
```

**LoginInterceptor.java：**

```java
package com.demo.interceptor;

import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.*;

public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        System.out.println("【拦截器】方法执行前: " + request.getRequestURI());

        // 模拟：URL 带 token 参数就放行，否则拦截
        if (request.getParameter("token") != null) {
            return true;
        }
        response.getWriter().write("{\"code\":401,\"message\":\"请先登录\"}");
        return false;   // 拦截
    }
}
```

---

## Step 7：部署测试（IDEA + Tomcat）

**部署步骤：**

1. Run → Edit Configurations → 添加 Tomcat Server → Local
2. Deployment → 添加 Artifact（springmvc-demo:war exploded）
3. Application context 设为 `/demo`
4. 启动 Tomcat，访问：`http://localhost:8080/demo/index.html`

**测试清单（用浏览器或 Postman）：**

| # | 请求 | URL | 预期返回 JSON |
|---|------|-----|--------------|
| ① | GET | `/demo/user?name=张三&age=20&token=1` | `{"code":200,"data":"收到参数 name=张三, age=20"}` |
| ② | GET | `/demo/user/1?token=1` | `{"code":200,"data":"查询用户 id=1"}` |
| ③ | POST 表单 | `/demo/user/add?token=1`（表单 name=张三&age=20） | `{"code":200,"data":"新增用户: 张三, 20"}` |
| ④ | POST JSON | `/demo/user/save?token=1`，Body: `{"name":"张三","age":20}`，Content-Type: application/json | `{"code":200,"data":"保存用户: 张三"}` |
| ⑤ | GET | `/demo/user/list?token=1` | `{"code":200,"data":[{...},{...}]}` |
| ⑥ | GET 无 token | `/demo/user/list` | `{"code":401,"message":"请先登录"}`（拦截器生效） |

---

## 练习任务（自己动手）

1. **测全局异常**：加一个接口 `@GetMapping("/error")`，直接 `throw new RuntimeException("用户不存在")`，访问它——看返回是不是规范的 JSON（而不是报错页）
2. **加 @RequestParam**：把接口 ① 改成前端传 `username`，方法参数叫 `name`，用 `@RequestParam("username")` 绑定
3. **返回 Map**：写一个接口返回 `Map<String, Object>`（含 code/message/data 三字段），对比和 Result 的差异
4. **拦截器排除**：把 `exclude-mapping` 里的 `/user` 加上，测 `⑥` 是否放行（理解放行配置的作用）

---

## 验证标准

- [ ] 5 个接口全部返回规范 JSON（code + message + data）
- [ ] 无 token 访问被拦截器拦截（返回 401 JSON）
- [ ] Controller 里没写 try-catch，异常却被全局处理器接住
- [ ] 中文不乱码（URL 编码 + JSON 编码正常）
- [ ] POST JSON 时能正确收到对象（@RequestBody 生效）

---

## 常见报错排查

| 报错 | 原因 | 解决 |
|------|------|------|
| 404 访问不到接口 | web.xml 的 servlet-mapping 或 context 配置不对 | 检查 `/` 拦截 + Application context |
| 415 Unsupported Media Type | 前端没传 Content-Type: application/json | 检查请求头 |
| 返回的不是 JSON | 没引 Jackson 依赖 | 加 jackson-databind |
| 405 Method Not Allowed | GET 接口用了 POST 请求（或反之） | 检查请求方法 |
| 中文乱码 | 没配编码过滤器 | 回忆 03.4 写一个 EncodingFilter |

---

*上一节理论：[06.5 文件上传下载](../theory/06.5-file-upload-download.md) | 下一章：[07-Spring Boot](../../07-springboot/theory/)*
