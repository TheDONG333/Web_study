# 实操 05.1：把 MyBatis CRUD 改造为 Spring 管理（main 方法直跑）

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：报错时把完整堆栈发给 AI 分析

---

## 前置要求

- ✅ 已读完理论 05.1 ~ 05.4
- ✅ 已完成实操 [04.1 MyBatis CRUD demo](../04-mybatis/practice/practice-04.1-mybatis-crud-demo.md)（改造它的基础）
- ✅ 需要安装：JDK 17+、Maven、MySQL 8+

## 项目目标

把第四章的 `mybatis-demo` 改造成 Spring 版：**不再有 MyBatisUtil 和 SqlSession**，所有对象（Mapper、Service）都从 Spring 容器获取。

---

## 改造前后对比

| | 第四章（纯 MyBatis） | 本章（Spring 整合） |
|---|---------------------|---------------------|
| 取 Session | `MyBatisUtil.openSession()` | 不需要了 |
| 调用 SQL | `session.selectOne("userMapper.findById", 1)` | `userMapper.findById(1)` |
| 配置 | mybatis-config.xml | applicationContext.xml |
| Service 获取 Dao | 手动 `new UserServiceImpl()` | `ctx.getBean()` 或 `@Autowired` |

---

## 项目结构

```
spring-mybatis-demo/
├── pom.xml
└── src/main/
    ├── java/com/demo/
    │   ├── entity/User.java
    │   ├── mapper/UserMapper.java        ← 接口（新增！）
    │   ├── service/
    │   │   ├── UserService.java
    │   │   └── impl/UserServiceImpl.java
    │   └── Main.java
    └── resources/
        ├── applicationContext.xml        ← 替换 mybatis-config.xml
        └── mapper/UserMapper.xml
```

> 与第四章相比：删掉 MyBatisUtil.java 和 mybatis-config.xml，新增 UserMapper.java 接口和 applicationContext.xml。

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
    <artifactId>spring-mybatis-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Spring 核心 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.3.31</version>
        </dependency>
        <!-- Spring JDBC（事务） -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>5.3.31</version>
        </dependency>
        <!-- MyBatis + Spring 整合包 -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>2.1.2</version>
        </dependency>
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
    </dependencies>
</project>
```

---

## Step 2：applicationContext.xml

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

    <!-- ① 组件扫描：@Service 等注册进容器 -->
    <context:component-scan base-package="com.demo"/>

    <!-- ② 数据源 -->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url"
            value="jdbc:mysql://localhost:3306/agent_learn?useSSL=false&amp;serverTimezone=Asia/Shanghai"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </bean>

    <!-- ③ SqlSessionFactory 交给 Spring -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    </bean>

    <!-- ④ 扫描 Mapper 接口，注册为 Bean -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.demo.mapper"/>
    </bean>

    <!-- ⑤ 开启注解事务（配合 @Transactional） -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <tx:annotation-driven transaction-manager="transactionManager"/>

</beans>
```

> 数据库密码如果不是 123456，改成你自己的。

---

## Step 3：实体类 User.java

与第四章完全一样（id/username/password/email/phone + getter/setter），直接复制即可。

---

## Step 4：Mapper 接口 UserMapper.java（新增）

```java
package com.demo.mapper;

import com.demo.entity.User;
import java.util.List;

/**
 * Mapper 接口：方法名和 XML 中 SQL 的 id 一一对应
 */
public interface UserMapper {

    User findById(int id);

    List<User> findAll();

    int insert(User user);

    int update(User user);

    int deleteById(int id);
}
```

---

## Step 5：映射文件 mapper/UserMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC
    "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- ⚠️ namespace 必须写接口的全类名 -->
<mapper namespace="com.demo.mapper.UserMapper">

    <!-- id 必须和接口方法名一致 -->
    <select id="findById" parameterType="int" resultType="com.demo.entity.User">
        SELECT * FROM users WHERE id = #{id}
    </select>

    <select id="findAll" resultType="com.demo.entity.User">
        SELECT * FROM users
    </select>

    <insert id="insert" parameterType="com.demo.entity.User"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (username, password, email, phone)
        VALUES (#{username}, #{password}, #{email}, #{phone})
    </insert>

    <update id="update" parameterType="com.demo.entity.User">
        UPDATE users SET username=#{username}, email=#{email}, phone=#{phone}
        WHERE id = #{id}
    </update>

    <delete id="deleteById" parameterType="int">
        DELETE FROM users WHERE id = #{id}
    </delete>

</mapper>
```

---

## Step 6：Service 层（用 @Service + @Autowired + @Transactional）

```java
package com.demo.service;

import com.demo.entity.User;
import java.util.List;

public interface UserService {
    User findById(int id);
    List<User> findAll();
    String register(User user);
    String delete(int id);
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

@Service                        // ① 交给 Spring 管理
public class UserServiceImpl implements UserService {

    @Autowired                  // ② 容器自动注入 Mapper
    private UserMapper userMapper;

    @Override
    public User findById(int id) {
        return userMapper.findById(id);
    }

    @Override
    public List<User> findAll() {
        return userMapper.findAll();
    }

    @Override
    @Transactional(rollbackFor = Exception.class)   // ③ 注册方法加事务
    public String register(User user) {
        // 模拟业务校验
        if (user.getUsername() == null || user.getUsername().isEmpty()) {
            return "用户名不能为空";
        }
        int rows = userMapper.insert(user);
        return rows > 0 ? "注册成功" : "注册失败";
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public String delete(int id) {
        int rows = userMapper.deleteById(id);
        return rows > 0 ? "删除成功" : "删除失败";
    }
}
```

---

## Step 7：Main.java — 从容器获取 Service 操作数据库

```java
package com.demo;

import com.demo.entity.User;
import com.demo.service.UserService;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {
    public static void main(String[] args) {
        // 启动 Spring 容器
        ClassPathXmlApplicationContext ctx =
            new ClassPathXmlApplicationContext("applicationContext.xml");

        // 从容器获取 Service（Spring 管理了一切对象）
        UserService userService = ctx.getBean(UserService.class);

        // ① 查询单个
        System.out.println("① 查询单个: " + userService.findById(1));

        // ② 查询所有
        System.out.println("② 总数: " + userService.findAll().size());

        // ③ 注册（带事务）
        String result = userService.register(new User("springUser", "123456", "s@demo.com", "13000000000"));
        System.out.println("③ " + result);

        // ④ 验证事务：打印容器里是不是同一个对象（singleton）
        UserService s1 = ctx.getBean(UserService.class);
        UserService s2 = ctx.getBean(UserService.class);
        System.out.println("④ 两次获取是同一个对象吗? " + (s1 == s2));

        // ⑤ 删除刚注册的用户
        ctx.close();
    }
}
```

**运行 `main()`，预期看到：**

```
① 查询单个: User{id=1, username='admin', ...}
② 总数: 3
③ 注册成功
④ 两次获取是同一个对象吗? true
```

---

## 练习任务（自己动手）

1. **测试事务回滚**：给 `register` 加一个参数 `throwException`，为 true 时在 `insert` 之后 `throw new RuntimeException("模拟失败")`。运行两次对比：抛异常时数据库里有没有新用户？
2. **测试传播行为**：新建 `CouponService`，方法标 `@Transactional(propagation = Propagation.REQUIRES_NEW)`；`register` 里调用它并让它抛异常——观察：用户注册成功了吗？分别用 REQUIRED 和 REQUIRES_NEW 试一次，对比结果差异。
3. **写一个切面**：按 [05.2 AOP](../theory/05.2-aop.md) 第 5 节写 `LogAspect`，用 `@Around` 给 `UserService` 所有方法加日志，观察控制台输出。
4. **思考**：为什么 `@Transactional` 能生效？没有 `@Service` 注解时它还会生效吗？（提示：代理对象）

---

## 验证标准

- [ ] main 方法直接运行，所有功能正常
- [ ] 从容器获取 Service（不再是 new 出来的）
- [ ] `④ 两次获取是同一个对象吗? true` → 验证了 singleton
- [ ] 注册的用户能在数据库查到（事务提交）
- [ ] 抛异常时数据库没有新用户（事务回滚生效）
- [ ] 切面日志能打印（AOP 生效）

---

## 常见报错排查

| 报错 | 原因 | 解决 |
|------|------|------|
| `Property 'dataSource' is required` | 忘了给 SqlSessionFactoryBean 配数据源 | 检查 `<property name="dataSource" ref="dataSource"/>` |
| `Invalid bound statement` | namespace 或 id 和接口不匹配 | namespace=接口全类名，id=方法名 |
| `No bean named 'userMapper'` | MapperScannerConfigurer 的 basePackage 写错 | 检查包路径 |
| `Consider defining a bean of type` | 类没被扫描到 | 检查 @Service/@Component 和 component-scan |
| 事务不生效 | 内部自调用 / 方法非 public / 异常被吞 | 翻理论 05.3 失效场景 |

---

*上一节理论：[05.4 Spring 整合 MyBatis](../theory/05.4-integrate-mybatis.md) | 下一章：[06-AI Agent入门](../../06-introduction-to-ai-agent/theory/)*
