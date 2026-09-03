# 实操 08.1：用户管理 API（前后端分离风格，Postman 测试）

> ⏱ **预计时长**：90 分钟
> 📌 **难度**：⭐⭐⭐⭐
> 🔧 **AI 辅助**：跑不通把完整报错发给 AI；Postman 不会配让 AI 截图指导

---

## 前置要求

- ✅ 已读完理论 08.1 ~ 08.5
- ✅ 已完成实操 [07.1 SSM 用户管理](../07-ssm-integration/practice/practice-07.1-ssm-user-management.md)
- ✅ 需要安装：JDK 26、Maven、MySQL 8+、IntelliJ IDEA、Postman（或 Apifox）

## 项目目标

写一个**只返回 JSON、不用页面**的后端项目（前后端分离风格），用 Postman 测接口：

| # | 接口 | 说明 | 登录要求 |
|---|------|------|---------|
| ① | POST `/api/user/login` | 登录，成功返回 token | ❌ |
| ② | POST `/api/user/logout` | 登出，销毁 token | ✅ |
| ③ | GET `/api/user/page?page=1&size=3` | 分页查询 | ✅ |
| ④ | GET `/api/user/{id}` | 查详情 | ✅ |
| ⑤ | POST `/api/user` | 新增（@Valid 校验） | ✅ |
| ⑥ | PUT `/api/user/{id}` | 修改 | ✅ |
| ⑦ | DELETE `/api/user/{id}` | 删除 | ✅ |

**这个项目把 08.3 讲的东西全部用上**：统一 Result、全局异常、登录拦截器（token 版）、CORS。**登录态不再是 Session，而是 token**（前后端分离的标准做法）——和 03/06 章对比着体会差异。

---

## 项目结构（共 11 个 Java 文件，全部自己敲一遍）

```
user-api/
├── pom.xml
└── src/main/
    ├── java/com/demo/
    │   ├── UserApiApplication.java        ← 启动类（@MapperScan）
    │   ├── common/Result.java             ← 统一返回
    │   ├── common/BusinessException.java  ← 业务异常（手动抛，全局接）
    │   ├── config/WebConfig.java          ← 拦截器注册 + CORS
    │   ├── controller/AuthController.java ← 登录/登出
    │   ├── controller/UserController.java ← 用户 CRUD
    │   ├── entity/User.java               ← 实体（带校验注解）
    │   ├── exception/GlobalExceptionHandler.java
    │   ├── interceptor/LoginInterceptor.java
    │   ├── mapper/UserMapper.java
    │   └── service/UserService.java
    └── resources/
        ├── application.yml
        └── mapper/UserMapper.xml
```

---

## Step 1：数据库准备

```sql
CREATE DATABASE IF NOT EXISTS springboot_learn DEFAULT CHARSET utf8mb4;
USE springboot_learn;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20)
);

INSERT INTO users (username, password, email, phone) VALUES
('admin',    MD5('123456'), 'admin@example.com',  '13800000001'),
('zhangsan', MD5('123456'), 'zhang@example.com', '13800000002');
```

> 密码用 MD5 存储（回忆实操 03.1 的写法），登录时把输入的密码 MD5 后再比对。

---

## Step 2：创建项目（两种方式等价，选一个）

**方式 A：IDEA Spring Initializr**——理论 08.1 讲过，勾选：Spring Web / Validation / MyBatis Framework / MySQL Driver，Java 版本选 26。

**方式 B：手写 pom.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.15</version>
    </parent>

    <groupId>com.demo</groupId>
    <artifactId>user-api</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>26</java.version>
    </properties>

    <dependencies>
        <!-- web：SpringMVC + 内嵌 Tomcat + JSON -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- @Valid 参数校验 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <!-- MyBatis 整合（不在 Boot 全家桶里，需自己写版本） -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>3.0.4</version>
        </dependency>
        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

> 对比第七章：以前 web.xml + 三份 xml + 8 个依赖，现在一个 parent + 4 个依赖搞定。web/validation 不用写版本（parent 统一管理）。

---

## Step 3：配置文件 + 启动类

**application.yml**：

```yaml
server:
  port: 8080

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot_learn?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 123456        # 改成你自己的密码

mybatis:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true

logging:
  level:
    com.demo.mapper: debug   # 打印 SQL，方便对照
```

**UserApiApplication.java**：

```java
package com.demo;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.demo.mapper")   // 扫描 Mapper 接口（理论 08.2：不用一个个加 @Mapper）
public class UserApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApiApplication.class, args);
    }
}
```

---

## Step 4：实体类和通用类（3 个小文件）

**entity/User.java**（校验注解 = 08.4 的"体检表"）：

```java
package com.demo.entity;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;

public class User {
    private Integer id;

    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 20, message = "用户名长度 3-20 个字符")
    private String username;

    private String password;    // 不贴注解：新增时 Service 里手动判空，修改时可不传

    @Email(message = "邮箱格式不对")
    private String email;

    @Pattern(regexp = "^1\\d{10}$", message = "手机号格式不对")
    private String phone;

    public User() {}
    public User(Integer id, String username) { this.id = id; this.username = username; }

    // ── Getter / Setter（id、username、password、email、phone，自己补全）──
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

**common/Result.java**（和理论 08.3 一样）：

```java
package com.demo.common;

public class Result {
    private int code;
    private String message;
    private Object data;

    public static Result ok(Object data) { return new Result(200, "操作成功", data); }
    public static Result fail(int code, String message) { return new Result(code, message, null); }

    public Result() {}
    public Result(int code, String message, Object data) {
        this.code = code; this.message = message; this.data = data;
    }
    public int getCode() { return code; }
    public void setCode(int code) { this.code = code; }
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    public Object getData() { return data; }
    public void setData(Object data) { this.data = data; }
}
```

**common/BusinessException.java**（业务错误也抛异常，让 Controller 保持干净）：

```java
package com.demo.common;

/** 业务异常：Service 里"预期内的错误"就抛它，全局异常处理器统一转 JSON */
public class BusinessException extends RuntimeException {
    private int code;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }

    public int getCode() { return code; }
}
```

---

## Step 5：Mapper 层（和第四章写法一样）

**mapper/UserMapper.java**：

```java
package com.demo.mapper;

import com.demo.entity.User;
import org.apache.ibatis.annotations.Param;
import java.util.List;

public interface UserMapper {

    User findByUsername(String username);

    User findById(int id);

    List<User> findPage(@Param("offset") int offset, @Param("size") int size);

    int count();

    int insert(User user);

    int update(User user);

    int deleteById(int id);
}
```

**resources/mapper/UserMapper.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.demo.mapper.UserMapper">

    <select id="findByUsername" resultType="com.demo.entity.User">
        SELECT * FROM users WHERE username = #{username}
    </select>

    <select id="findById" resultType="com.demo.entity.User">
        SELECT * FROM users WHERE id = #{id}
    </select>

    <!-- 分页：LIMIT 跳过 offset 条，再取 size 条 -->
    <select id="findPage" resultType="com.demo.entity.User">
        SELECT * FROM users ORDER BY id LIMIT #{offset}, #{size}
    </select>

    <select id="count" resultType="int">
        SELECT COUNT(*) FROM users
    </select>

    <insert id="insert" parameterType="com.demo.entity.User"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (username, password, email, phone)
        VALUES (#{username}, #{password}, #{email}, #{phone})
    </insert>

    <update id="update" parameterType="com.demo.entity.User">
        UPDATE users SET username = #{username}, email = #{email}, phone = #{phone}
        WHERE id = #{id}
    </update>

    <delete id="deleteById" parameterType="int">
        DELETE FROM users WHERE id = #{id}
    </delete>

</mapper>
```

---

## Step 6：Service 层（登录 token 存内存 Map）

**service/UserService.java** —— 本章核心业务，包含登录态 token 设计：

```java
package com.demo.service;

import com.demo.common.BusinessException;
import com.demo.entity.User;
import com.demo.mapper.UserMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    /** 登录态的"小本本"：token → 用户名。生产用 Redis/JWT，这里先学原理 */
    private final Map<String, String> TOKENS = new ConcurrentHashMap<>();

    // ── ① 登录：成功返回 token，失败抛业务异常 ──
    public String login(String username, String password) {
        User user = userMapper.findByUsername(username);
        if (user == null || !user.getPassword().equals(md5(password))) {
            throw new BusinessException(401, "用户名或密码错误");
        }
        String token = UUID.randomUUID().toString().replace("-", "");
        TOKENS.put(token, username);
        return token;
    }

    // ── ② 登出 ──
    public void logout(String token) {
        TOKENS.remove(token);
    }

    // ── ③ 拦截器调用：token 还在小本本上吗 ──
    public boolean isLogin(String token) {
        return token != null && TOKENS.containsKey(token);
    }

    // ── ④ 分页查询 ──
    public Map<String, Object> page(int page, int size) {
        if (page < 1) page = 1;
        if (size < 1) size = 10;
        long total = userMapper.count();
        List<User> list = userMapper.findPage((page - 1) * size, size);

        Map<String, Object> data = new LinkedHashMap<>();
        data.put("page", page);
        data.put("size", size);
        data.put("total", total);
        data.put("totalPages", (int) Math.ceil((double) total / size));
        data.put("list", list);
        return data;
    }

    // ── ⑤ 查详情：不存在就抛 404 ──
    public User findById(int id) {
        User user = userMapper.findById(id);
        if (user == null) throw new BusinessException(404, "用户不存在");
        return user;
    }

    // ── ⑥ 新增（带事务）──
    @Transactional(rollbackFor = Exception.class)
    public void add(User user) {
        if (user.getPassword() == null || user.getPassword().isEmpty()) {
            throw new BusinessException(400, "密码不能为空");
        }
        if (userMapper.findByUsername(user.getUsername()) != null) {
            throw new BusinessException(400, "用户名已存在");
        }
        user.setPassword(md5(user.getPassword()));
        userMapper.insert(user);
    }

    // ── ⑦ 修改（事务演示点：先删旧数据再插新数据也行，这里标准 UPDATE）──
    @Transactional(rollbackFor = Exception.class)
    public void update(int id, User user) {
        findById(id);                       // 不存在 → 抛 404
        user.setId(id);
        userMapper.update(user);
    }

    public void delete(int id) {
        findById(id);
        userMapper.deleteById(id);
    }

    private String md5(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder();
            for (byte b : digest) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## Step 7：Controller 层（2 个，都很薄）

**controller/AuthController.java**：

```java
package com.demo.controller;

import com.demo.common.Result;
import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/user")
public class AuthController {

    @Autowired
    private UserService userService;

    // POST /api/user/login  body: {"username":"admin","password":"123456"}
    @PostMapping("/login")
    public Result login(@RequestBody User user) {
        String token = userService.login(user.getUsername(), user.getPassword());
        return Result.ok(Map.of("token", token));
    }

    // POST /api/user/logout（token 从请求头拿——回忆 @RequestHeader）
    @PostMapping("/logout")
    public Result logout(@RequestHeader("token") String token) {
        userService.logout(token);
        return Result.ok("已退出登录");
    }
}
```

**controller/UserController.java**：

```java
package com.demo.controller;

import com.demo.common.Result;
import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/user")
public class UserController {

    @Autowired
    private UserService userService;

    // GET /api/user/page?page=1&size=3
    @GetMapping("/page")
    public Result page(@RequestParam(defaultValue = "1") int page,
                       @RequestParam(defaultValue = "3") int size) {
        return Result.ok(userService.page(page, size));
    }

    // GET /api/user/1
    @GetMapping("/{id}")
    public Result detail(@PathVariable Integer id) {
        return Result.ok(userService.findById(id));
    }

    // POST /api/user  body: {"username":"tom","password":"123456","email":"tom@demo.com","phone":"13800000005"}
    @PostMapping
    public Result add(@Valid @RequestBody User user) {   // ⚠️ @Valid 触发校验
        userService.add(user);
        return Result.ok("新增成功, id=" + user.getId());
    }

    // PUT /api/user/1  body: {"username":"tom","email":"t@demo.com","phone":"13800000006"}
    @PutMapping("/{id}")
    public Result update(@PathVariable Integer id, @Valid @RequestBody User user) {
        userService.update(id, user);
        return Result.ok("修改成功");
    }

    // DELETE /api/user/1
    @DeleteMapping("/{id}")
    public Result delete(@PathVariable Integer id) {
        userService.delete(id);
        return Result.ok("删除成功");
    }
}
```

---

## Step 8：拦截器 + CORS + 全局异常（3 个文件）

**interceptor/LoginInterceptor.java**（登录检查从 Session 换成 Header token）：

```java
package com.demo.interceptor;

import com.demo.service.UserService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class LoginInterceptor implements HandlerInterceptor {

    @Autowired
    private UserService userService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("token");
        if (userService.isLogin(token)) {
            return true;                                    // 放行
        }
        response.setStatus(401);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":401,\"message\":\"请先登录\"}");
        return false;
    }
}
```

**config/WebConfig.java**（拦截器注册 + CORS，对照理论 08.3）：

```java
package com.demo.config;

import com.demo.interceptor.LoginInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/user/login");   // 登录接口放行
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("*")                // 开发期放开，生产按域名收紧
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*");
    }
}
```

**exception/GlobalExceptionHandler.java**（一个总收银台，接住所有异常）：

```java
package com.demo.exception;

import com.demo.common.BusinessException;
import com.demo.common.Result;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // ① 业务异常：Service 里手动抛的（如 401 登录失败、400 用户名已存在）
    @ExceptionHandler(BusinessException.class)
    public Result handleBusiness(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    // ② @Valid 校验失败（理论 08.4 的体检打回）
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .findFirst().map(FieldError::getDefaultMessage)
                .orElse("参数错误");
        return Result.fail(400, msg);
    }

    // ③ 其他意外异常（兜底，不把内部信息暴露给前端）
    @ExceptionHandler(Exception.class)
    public Result handleOther(Exception e) {
        e.printStackTrace();
        return Result.fail(500, "服务器开小差了，请稍后再试");
    }
}
```

---

## Step 9：启动 + Postman 全流程测试

**启动**：右键 `UserApiApplication` → Run。看到 `Tomcat started on port 8080` 即成功。

**测试流程（按顺序做）**：

| 步骤 | 请求 | Body / Header | 预期结果 |
|------|------|--------------|---------|
| ① 登录成功 | POST `http://localhost:8080/api/user/login` | Body(JSON): `{"username":"admin","password":"123456"}` | `code:200`，data 里有 **token**，复制它 |
| ② 密码错误 | 再发一次登录 | password 改成 `111111` | `code:401`，message"用户名或密码错误" |
| ③ 无 token 被拦 | GET `/api/user/page?page=1&size=3` | 不带 Header | `code:401`"请先登录"（拦截器生效） |
| ④ 分页查询 | GET `/api/user/page?page=1&size=3` | Header: `token: 复制的token` | `code:200`，data 有 page/total/list |
| ⑤ 校验拦截 | POST `/api/user` | token + body: `{"password":"123456"}`（没 username） | `code:400`"用户名不能为空"（@Valid 生效） |
| ⑥ 校验拦截 2 | POST `/api/user` | token + body: `{"username":"a","password":"123456"}` | `code:400`"用户名长度 3-20" |
| ⑦ 新增成功 | POST `/api/user` | token + body: `{"username":"tom","password":"123456","email":"tom@demo.com","phone":"13800000005"}` | `code:200`，新增成功 |
| ⑧ 查详情 | GET `/api/user/5` | token（id 换成⑦返回的） | `code:200`，返回 tom 的数据 |
| ⑨ 修改 | PUT `/api/user/5` | token + body: `{"username":"tom","email":"t@demo.com","phone":"13800000006"}` | `code:200`，再去 ⑧ 查，email 已变 |
| ⑩ 删除 | DELETE `/api/user/5` | token | `code:200`，再 ⑧ 查 → `code:404`"用户不存在" |
| ⑪ 登出 | POST `/api/user/logout` | token | `code:200`；再用同一 token 查 ⑧ → `code:401`（token 失效） |

> 💡 **Postman 技巧**：登录接口的 Tests 里写一行 `pm.globals.set("token", pm.response.json().data.token)`，之后所有请求 Header 填 `{{token}}` 就不用手动复制了（不会写就让 AI 教你）。

---

## 练习任务（自己动手）

1. **加"注册"接口**：`POST /api/user/register`，复用 `userService.add()`，`excludePathPatterns` 加上它。思考：它和"新增"逻辑一样，实际项目中你会怎么区分？（提示：权限——谁在什么时候能调新增）
2. **把校验换成 @Pattern**：给 username 加 `@Pattern(regexp = "^[a-zA-Z0-9_]{3,20}$")`，试试中文用户名被拦的报错。
3. **profile 实战**：建 `application-prod.yml`（端口改 9090、日志级别 warn），用启动参数 `--spring.profiles.active=prod` 启动一次，观察端口变化（08.5 的内容，动一次手就记住了）。
4. **（选做）CORS 实验**：浏览器打开任意网站（如百度）的开发者工具 Console，执行 `fetch('http://localhost:8080/api/user/login', {method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:'admin',password:'123456'})})`——先带着 WebConfig 跑，再注释掉 `addCorsMappings` 跑一次，对比两次的报错，你就理解 CORS 是谁在拦你了。
5. **（挑战）把登录失败次数限流**：连续输错 3 次锁定 1 分钟，用 `Map<String, LoginFailInfo>` 记录（加分题，让 AI 带路）。

---

## 验证标准

- [ ] 只写 JSON 接口，全程没用 JSP/HTML，用 Postman 完成全部测试
- [ ] 登录成功返回 token；错密码返回 401
- [ ] 不带 token 访问任何业务接口都返回 401（拦截器 + 全局异常都工作）
- [ ] @Valid 校验生效：username 空/过短、email 格式错、phone 格式错都有友好 message
- [ ] 所有返回都是 {code, message, data} 结构
- [ ] 控制台能看到 MyBatis 打印的 SQL（对照参数是否和预期一致）
- [ ] 说得出：token 版登录和 03 章 Session 版登录，本质区别是什么？

---

## 常见报错排查

| 报错 | 原因 | 解决 |
|------|------|------|
| `Failed to configure a DataSource` | yml 数据源没配全（最常见：url 拼错、密码错） | 检查 application.yml 的 `spring.datasource` |
| `Failed to bind properties under 'server'` / 启动解析错误 | yml 格式错 | 冒号后要有空格、缩进对齐，看报错行号 |
| `Consider defining a bean of type 'com.demo.mapper.UserMapper'` | @MapperScan 没加或包路径错 | 检查启动类上的 `@MapperScan("com.demo.mapper")` |
| `Invalid bound statement (not found)` | namespace/id 和接口不匹配 | namespace=接口全类名，id=方法名 |
| 所有业务接口返回 401 | token 没带进 Header，或 exclude 配错 | 检查 Postman Header；检查拦截器 excludePathPatterns |
| @Valid 不生效 | 忘了引 starter-validation，或方法参数没写 @Valid | 检查依赖和 `@Valid @RequestBody` |
| 访问接口 Whitelabel 404 | 路径写错，或 Controller 没放在启动类子包 | 检查 @RequestMapping 和包结构 |
| 8080 端口被占用 | 上次没关掉 | 改 `server.port: 8081` 重试 |

---

*上一节理论：[08.5 进阶配置](../theory/08.5-advanced-profile-and-configuration.md) | 下一章：[09-AI Agent 入门](../../09-introduction-to-ai-agent/theory/)*
