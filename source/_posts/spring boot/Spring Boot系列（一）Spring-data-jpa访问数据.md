---
layout: post
title: Spring Boot系列（一）Spring-data-jpa 访问数据
date: 2019-8-26 19:34:33
categories: blog
tags: [springboot,jpa]
description: spring-data-jpa使用实践

---



## Spring-data-jpa 访问数据

由于Spring-data-jpa依赖于Hibernate。如果您对Hibernate有一定了解，下面内容可以毫不费力的看懂并上手使用Spring-data-jpa。如果您还是Hibernate新手，您可以先按如下方式入门，再建议回头学习一下Hibernate以帮助这部分的理解和进一步使用。

### 使用示例

#### 工程配置

在`pom.xml`中添加相关依赖，加入以下内容：

<!-- more -->

```xml
<dependency
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

在`application.xml`中配置：数据库连接信息（如使用嵌入式数据库则不需要）、自动创建表结构的设置，例如使用`mysql`的情况如下：

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.jpa.properties.hibernate.hbm2ddl.auto=create-drop
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
spring.jpa.show-sql = true
```

`spring.jpa.properties.hibernate.hbm2ddl.auto`是hibernate的配置属性，其主要作用是：自动创建、更新、验证数据库表结构。该参数的几种配置如下：

- `create`：每次加载hibernate时都会删除上一次的生成的表，然后根据你的model类再重新来生成新表，哪怕两次没有任何改变也要这样执行，这就是导致数据库表数据丢失的一个重要原因。
- `create-drop`：每次加载hibernate时根据model类生成表，但是sessionFactory一关闭,表就自动删除。
- `update`：最常用的属性，第一次加载hibernate时根据model类会自动建立起表的结构（前提是先建立好数据库），以后加载hibernate时根据model类自动更新表结构，即使表结构改变了但表中的行仍然存在不会删除以前的行。要注意的是当部署到服务器后，表结构是不会被马上建立起来的，是要等应用第一次运行起来后才会。
- `validate`：每次加载hibernate时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。

`spring.jpa.database-platform`，其作用是：指定mysql的存储引擎为：`innodb`

例如使用`oracle`的情况如下：

```properties
#数据库oracle配置
spring.datasource.driver-class-name = oracle.jdbc.driver.OracleDriver
spring.datasource.url = jdbc:oracle:thin:@10.0.6.138:1521:xe
spring.datasource.username = PINGAN_OPENAPI
spring.datasource.password = 123456

spring.jpa.hibernate.ddl-auto = update
spring.jpa.show-sql = true

# Hikari will use the above plus the following to setup connection pooling
spring.datasource.type = com.zaxxer.hikari.HikariDataSource
spring.datasource.hikari.minimum-idle = 5
spring.datasource.hikari.maximum-pool-size = 15
spring.datasource.hikari.auto-commit = true
spring.datasource.hikari.idle-timeout = 30000
spring.datasource.hikari.pool-name = DatebookHikariCP
spring.datasource.hikari.max-lifetime = 1800000
spring.datasource.hikari.connection-timeout = 30000
#mysql或者oracle这里配置不一样
#spring.datasource.hikari.connection-test-query = SELECT 1
spring.datasource.hikari.connection-test-query = SELECT 1  FROM DUAL
```

#### 创建实体

通过ORM框架其会被映射到数据库表中，由于配置了`hibernate.hbm2ddl.auto`，在应用启动的时候框架会自动去数据库中创建对应的表。

```java
package com.sinovoice.hcicloud.dto;


import com.fasterxml.jackson.annotation.JsonView;
import com.sinovoice.hcicloud.validator.MyConstraint;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CacheConcurrencyStrategy;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.GenericGenerator;

import javax.persistence.*;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.Past;
import java.util.Date;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "test_user")
public class User {

    public interface UserSimpleView {};
    public interface UserDetailView extends UserSimpleView {};

    /**
     * 逻辑主键
     */
    @Id
    @GenericGenerator(name = "hibernate-uuid", strategy = "uuid")
    @GeneratedValue(generator = "hibernate-uuid")
    @Column(name = "UUID", length = 32, nullable = false, insertable = true, updatable = false)
    private String id;

    /**
     * 因为设置了 unique 为true，会生成默认唯一索引
     */
    @MyConstraint(message = "这是一个测试")
    @Column(name = "USERNAME", length = 64, nullable = false, unique = true)
    private String username;

    @NotEmpty(message = "密码不能为空")
    @Column(name = "PASSWORD", length = 1000, nullable = false)
    private String password;

    @Past(message = "生日必须是过去的时间")
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "BIRTHDAY", updatable = false)
    @CreationTimestamp
    private Date birthday;

    @JsonView(UserSimpleView.class)
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    @JsonView(UserSimpleView.class)
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    @JsonView(UserDetailView.class)
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @JsonView(UserSimpleView.class)
    public Date getBirthday() {
        return birthday;
    }

    public void setBirthday(Date birthday) {
        this.birthday = birthday;
    }

    @Override
    public String toString() {
        return "User{" +
                "id='" + id + '\'' +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                ", birthday=" + birthday +
                '}';
    }
}
```

#### 创建数据访问接口

下面针对SendSuccessLog实体创建对应的`Repository`接口实现对该实体的数据访问，如下代码：

```java
package com.sinovoice.hcicloud.service.repository;

import com.sinovoice.hcicloud.dto.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

/**
 * @Author: mhn
 * @Date: 2019/6/19 23:11
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Repository
public interface UserRepository extends JpaRepository<User, String> {

    User findByUsername(String name);

    User findByUsernameAndPassword(String name, String password);

    @Query("from User u where u.username=:username")
    User findUser(@Param("username") String username);

    //自定义repository。手写sql
    @Query(value = "update User set username=?1 where id=?4",nativeQuery = true)   //占位符传值形式
    @Modifying
    int updateById(String name,String id);
}
```

在Spring-data-jpa中，只需要编写类似上面这样的接口就可实现数据访问。不再像我们以往编写了接口时候还需要自己编写接口实现类，直接减少了我们的文件清单。

下面对上面的`UserRepository`做一些解释，该接口继承自`JpaRepository`，通过查看`JpaRepository`接口的[API文档](http://docs.spring.io/spring-data/data-jpa/docs/current/api/)，可以看到该接口本身已经实现了创建（save）、更新（save）、删除（delete）、查询（findAll、findOne）等基本操作的函数，因此对于这些基础操作的数据访问就不需要开发者再自己定义。

在我们实际开发中，`JpaRepository`接口定义的接口往往还不够或者性能不够优化，我们需要进一步实现更复杂一些的查询或操作。由于本文重点在spring boot中整合spring-data-jpa，在这里先抛砖引玉简单介绍一下spring-data-jpa中让我们兴奋的功能，后续再单独开篇讲一下spring-data-jpa中的常见使用。

在上例中，我们可以看到下面两个函数：

- `User findByName(String name)`
- `User findByNameAndAge(String name, Integer age)`

它们分别实现了按name查询User实体和按name和age查询User实体，可以看到我们这里没有任何类SQL语句就完成了两个条件查询方法。这就是Spring-data-jpa的一大特性：**通过解析方法名创建查询**。

除了通过解析方法名来创建查询外，它也提供通过使用@Query 注解来创建查询，您只需要编写JPQL语句，并通过类似“:name”来映射@Param指定的参数，就像例子中的第三个findUser函数一样。

#### 定义service层的接口

```java
package com.sinovoice.hcicloud.service;

import com.sinovoice.hcicloud.dto.User;

import java.util.Iterator;

/**
 * @Author: mhn
 * @Date: 2019/6/19 23:20
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
public interface UserServiceJpa {
    /**
     * 删除
     */
    void delete(String id);

    /**
     * 增加
     */
    void insert(User user);

    /**
     * 更新
     */
    int update(User user);

    /**
     * 查询单个
     */
    User selectById(String id);

    /**
     * 查询全部列表
     */
    Iterator<User> selectAll(int pageNum, int pageSize);
}
```



#### 定义service层的实现类

```java
package com.sinovoice.hcicloud.service.impl;

import com.sinovoice.hcicloud.dto.User;
import com.sinovoice.hcicloud.service.UserServiceJpa;
import com.sinovoice.hcicloud.service.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;

import java.util.Iterator;
import java.util.Optional;

/**
 * @Author: mhn
 * @Date: 2019/6/19 23:21
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Service
public class UserServiceJpaImpl implements UserServiceJpa {

    @Autowired
    private UserRepository userRepository;

    /**
     * 删除
     *
     * @param id
     */
    @Override
    public void delete(String id) {
        userRepository.deleteById(id);
    }

    /**
     * 增加
     *
     * @param user
     */
    @Override
    public void insert(User user) {
        userRepository.save(user);
    }

    /**
     * 更新
     *
     * @param user
     */
    @Override
    public int update(User user) {
        userRepository.save(user);
        return 1;
    }

    /**
     * 查询单个
     *
     * @param id
     */
    @Override
    public User selectById(String id) {
        Optional<User> optional = userRepository.findById(id);
        User user = optional.get();
        return user;
    }

    /**
     * 查询全部列表,并做分页
     *  @param pageNum 开始页数
     * @param pageSize 每页显示的数据条数
     */
    @Override
    public Iterator<User> selectAll(int pageNum, int pageSize) {
        //将参数传给这个方法就可以实现物理分页了，非常简单。
        Sort sort = new Sort(Sort.Direction.DESC, "id");
        Pageable pageable = new PageRequest(pageNum, pageSize, sort);
        Page<User> users = userRepository.findAll(pageable);
        Iterator<User> userIterator =  users.iterator();
        return  userIterator;
    }
}
```



#### 单元测试

```java
package com.sinovoice.hcicloud.service;

import com.sinovoice.hcicloud.dto.User;
import com.sinovoice.hcicloud.service.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.time.LocalDateTime;
import java.util.Date;
import java.util.Iterator;

/**
 * 测试的时候，要注释掉 MyConstraintValidator#helloService 依赖，不然会失败
 * @Author: mhn
 * @Date: 2019/6/19 22:49
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Slf4j
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class UserServiceJpaTest {
    @Autowired
    private UserServiceJpa userSerivceJpa;

    @Test
    public void test() throws Exception {
        User user = new User();
        user.setUsername("mhn1");
        user.setPassword("123456");
        user.setBirthday(new Date());
        userSerivceJpa.insert(user);

        User user1 = new User();
        user1.setUsername("mhn2");
        user1.setPassword("123456");
        user1.setBirthday(new Date());
        userSerivceJpa.insert(user1);


        Iterator<User> userIterator = userSerivceJpa.selectAll(1, 10);
        while (userIterator.hasNext()){
            User user2 = userIterator.next();
            userSerivceJpa.delete(user2.getId());
        }
    }
}
```



