# 实操 04.1：独立 MyBatis 小项目（不用 Spring，main 方法直跑）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：写完让 AI 检查你的 mapper.xml 和运行报错

---

## 前置要求

- ✅ 已读完理论 04.1 ~ 04.5
- ✅ 需要安装：JDK 17+、Maven、MySQL 8+、IntelliJ IDEA

## 项目目标

**不用 Spring，不写 JSP，纯 Maven 项目 + main 方法直接操作 MySQL**，完成用户表的 CRUD、条件查询、批量插入。

---

## 项目结构（全部自己写，共 5 个文件）

```
mybatis-demo/
├── pom.xml
└── src/main/
    ├── java/
    │   ├── com.demo.entity.User.java
    │   ├── com.demo.util.MyBatisUtil.java
    │   └── com.demo.Main.java
    └── resources/
        ├── mybatis-config.xml
        └── mapper/UserMapper.xml
```

---

## Step 1：数据库准备

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
('admin',  '123456', 'admin@example.com',  '13800000001'),
('zhangsan','123456', 'zhang@example.com', '13800000002'),
('lisi',   '123456', 'li@example.com',     '13800000003');
```

---

## Step 2：pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>mybatis-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- MyBatis -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.16</version>
        </dependency>
        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>
        <!-- 日志（打印 SQL） -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
    </dependencies>
</project>
```

---

## Step 3：全局配置文件 mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC
    "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <!-- 打印 SQL 日志 -->
        <setting name="logImpl" value="STDOUT_LOGGING"/>
        <!-- 下划线列名自动转驼峰属性 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url"
                    value="jdbc:mysql://localhost:3306/agent_learn?useSSL=false&serverTimezone=Asia/Shanghai"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>

</configuration>
```

> ⚠️ 数据库密码如果和示例不同，改成你自己的。

---

## Step 4：实体类 User.java

```java
package com.demo.entity;

public class User {
    private Integer id;
    private String username;
    private String password;
    private String email;
    private String phone;

    public User() {}

    public User(String username, String password, String email, String phone) {
        this.username = username;
        this.password = password;
        this.email = email;
        this.phone = phone;
    }

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

    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username +
               "', email='" + email + "', phone='" + phone + "'}";
    }
}
```

---

## Step 5：工具类 MyBatisUtil.java

```java
package com.demo.util;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.InputStream;

public class MyBatisUtil {
    private static final SqlSessionFactory FACTORY;

    static {
        try (InputStream in = Resources.getResourceAsStream("mybatis-config.xml")) {
            FACTORY = new SqlSessionFactoryBuilder().build(in);
        } catch (Exception e) {
            throw new RuntimeException("MyBatis 初始化失败", e);
        }
    }

    public static SqlSession openSession() {
        return FACTORY.openSession();
    }
}
```

---

## Step 6：映射文件 mapper/UserMapper.xml（本章核心，自己动手写）

**要求**：包含以下 6 条 SQL，其中 **动态 SQL 部分自己写**（TODO 处）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC
    "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="userMapper">

    <!-- 1. 查询单个用户（按 id） -->
    <select id="findById" parameterType="int" resultType="User">
        SELECT * FROM users WHERE id = #{id}
    </select>

    <!-- 2. 查询所有用户 -->
    <select id="findAll" resultType="User">
        SELECT * FROM users
    </select>

    <!-- 3. 新增（回填自增主键） -->
    <insert id="insert" parameterType="User"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (username, password, email, phone)
        VALUES (#{username}, #{password}, #{email}, #{phone})
    </insert>

    <!-- 4. 更新 -->
    <update id="update" parameterType="User">
        UPDATE users
        SET username = #{username}, email = #{email}, phone = #{phone}
        WHERE id = #{id}
    </update>

    <!-- 5. 删除 -->
    <delete id="deleteById" parameterType="int">
        DELETE FROM users WHERE id = #{id}
    </delete>

    <!-- 6. TODO 条件查询：<where> + <if>
         传 username 按用户名查，传 email 按邮箱查，两个都传则都生效 -->
    <select id="search" parameterType="map" resultType="User">
        SELECT * FROM users
        <!-- TODO: 自己写 <where> + <if> 实现条件查询 -->
    </select>

    <!-- 7. TODO 批量插入：<foreach> 循环拼 VALUES -->
    <insert id="batchInsert" parameterType="list">
        INSERT INTO users (username, password) VALUES
        <!-- TODO: 自己写 <foreach> -->
    </insert>

</mapper>
```

> 不会写时先翻理论 [04.4 动态 SQL](../theory/04.4-dynamic-sql.md)，再让 AI 给提示。

---

## Step 7：Main.java（测试全部功能）

```java
package com.demo;

import com.demo.entity.User;
import com.demo.util.MyBatisUtil;
import org.apache.ibatis.session.SqlSession;

import java.util.*;

public class Main {
    public static void main(String[] args) {
        try (SqlSession session = MyBatisUtil.openSession()) {

            // ── 1. 查询单个 ──
            User u = session.selectOne("userMapper.findById", 1);
            System.out.println("① 查询单个: " + u);

            // ── 2. 查询所有 ──
            List<User> all = session.selectList("userMapper.findAll");
            System.out.println("② 总数: " + all.size());

            // ── 3. 新增（回填主键）──
            User newUser = new User("test", "123456", "test@example.com", "13900000000");
            session.insert("userMapper.insert", newUser);
            System.out.println("③ 新增成功, 新id = " + newUser.getId());

            // ── 4. 更新 ──
            newUser.setEmail("updated@example.com");
            session.update("userMapper.update", newUser);
            System.out.println("④ 更新成功");

            // ── 5. 删除 ──
            session.delete("userMapper.deleteById", newUser.getId());
            System.out.println("⑤ 删除成功");

            // ── 6. 条件查询（只传 username）──
            Map<String, Object> params = new HashMap<>();
            params.put("username", "admin");
            List<User> result = session.selectList("userMapper.search", params);
            System.out.println("⑥ 条件查询(用户名): " + result);

            // ── 7. 批量插入 ──
            List<User> batch = List.of(
                new User("batch1", "123456", null, null),
                new User("batch2", "123456", null, null)
            );
            int rows = session.insert("userMapper.batchInsert", batch);
            System.out.println("⑦ 批量插入成功, 条数 = " + rows);

            // ── ⚠️ 别忘了提交事务！ ──
            session.commit();
            System.out.println("\n全部操作完成！");
        }
    }
}
```

**运行 `main()`，预期看到（控制台 + 日志）：**

```
==>  Preparing: SELECT * FROM users WHERE id = ?
==> Parameters: 1(Integer)
<==      Total: 1
① 查询单个: User{id=1, username='admin', ...}
② 总数: 3
③ 新增成功, 新id = 4
...
⑥ 条件查询(用户名): [User{id=1, username='admin', ...}]
⑦ 批量插入成功, 条数 = 2
```

---

## 练习任务（进阶，加深理解）

跑通后，自己动手改：

1. **加一个功能**：按 `username` 模糊查询（`LIKE`），用 `#{}` 怎么写？提示：`LIKE CONCAT('%', #{username}, '%')`
2. **测试 SQL 注入**：把 `search` 里的 `#{}` 改成 `${}`，传入 `admin' -- ` 观察结果，再改回来对比
3. **加分**：把 6 条 SQL 中的 `findAll` 改成 LIMIT 分页版，main 里查第 2 页（每页 2 条）
4. **思考**：为什么批量插入要传 List？`collection` 为什么写 `list`？

---

## 验证标准

- [x] main 方法直接运行，7 个功能全部输出正确结果
- [ ] 控制台能看到 SQL 日志（Preparing + Parameters）
- [x] 数据库中新插入的数据确实存在（去 MySQL 里查一下）
- [ ] 动态 SQL：只传 username 时 SQL 是 `WHERE username = ?`
- [x] 批量插入后数据库多了 2 条 batch 开头的记录
- [ ] 忘写 `session.commit()` 时会怎样？试试看数据是否入库（理解事务！）

---

## 常见报错排查

| 报错                                           | 原因                               | 解决                             |
| -------------------------------------------- | -------------------------------- | ------------------------------ |
| `Could not find resource mybatis-config.xml` | 配置文件不在 resources 下               | 检查路径，放 `src/main/resources`    |
| `Invalid bound statement (not found)`        | mapper.xml 没注册，或 namespace/id 写错 | 检查 `<mappers>` 和 namespace     |
| `Unknown column 'user_name'`                 | 列名对不上                            | 用 AS 别名或开驼峰映射                  |
| `Communications link failure`                | 连不上数据库                           | 检查 URL/用户名/密码，MySQL 是否启动       |
| 中文乱码                                         | URL 缺字符编码                        | URL 加 `characterEncoding=utf8` |

---

*上一节理论：[04.5 注解开发与进阶](../theory/04.5-annotation-and-advanced.md) | 下一章：[05-AI Agent入门](../../05-introduction-to-ai-agent/theory/)*
