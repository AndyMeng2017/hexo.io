---
layout: post
title: Spring Boot系列（二）Spring-data-jpa 事务机制
date: 2019-8-26 19:35:16
categories: blog
tags: [springboot,jpa]
description: spring-data-jpa使用实践


---



## Spring-data-jpa 事务机制

### 什么是事务？

我们在开发企业应用时，对于业务人员的一个操作实际是对数据读写的多步操作的结合。由于数据操作在顺序执行的过程中，任何一步操作都有可能发生异常，异常会导致后续操作无法完成，此时由于业务逻辑并未正确的完成，之前成功操作数据的并不可靠，需要在这种情况下进行回退。

事务的作用就是为了保证用户的每一个操作都是可靠的，事务中的每一步操作都必须成功执行，只要有发生异常就回退到事务开始未进行操作的状态。

事务管理是Spring框架中最为常用的功能之一，我们在使用Spring Boot开发应用时，大部分情况下也都需要使用事务。

### 快速入门

在Spring Boot中，当我们使用了spring-boot-starter-jdbc或spring-boot-starter-data-jpa依赖的时候，框架会自动默认分别注入DataSourceTransactionManager或JpaTransactionManager。所以我们不需要任何额外配置就可以用@Transactional注解进行事务的使用。

<!-- more -->

#### 单元测试

```java
package com.sinovoice.hcicloud.service;

import com.sinovoice.hcicloud.dto.SendSuccessLog;
import com.sinovoice.hcicloud.service.repository.SendSuccessLogRepository;
import com.sinovoice.hcicloud.web.controller.metrics.DemoController;
import lombok.extern.slf4j.Slf4j;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

/**
 * 测试的时候，要注释掉 MyConstraintValidator#helloService 依赖，不然会失败
 *
 * @Author: mhn
 * @Date: 2019/6/19 22:49
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Slf4j
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class SendSuccessLogServiceJpaTest {
    @Autowired
    private SendSuccessLogServiceJpa sendSuccessLogServiceJpa;

    @Test
    public void query() throws Exception {
        SendSuccessLog sendSuccessLog = sendSuccessLogServiceJpa.getSysConfig("8a80805e6ca8b9bd016ca8b9f4ee0000");
        System.err.println(sendSuccessLog.getVersion());
    }

    @Test
    public void update() throws Exception {
        SendSuccessLog sendSuccessLog = sendSuccessLogServiceJpa.getSysConfig("8a80805e6ca8b9bd016ca8b9f4ee0000");
        sendSuccessLog.setRemark1("0");
        System.err.println("开始保存");
        sendSuccessLogServiceJpa.saveSysConfig(sendSuccessLog);
    }

    @Test
    public void update1() throws Exception {
        SendSuccessLog sendSuccessLog = sendSuccessLogServiceJpa.getSysConfig("8a80805e6ca8b9bd016ca8b9f4ee0000");
        sendSuccessLog.setRemark1("1");
        System.err.println("开始保存");
        sendSuccessLogServiceJpa.testSysConfig1(sendSuccessLog);
    }

    @Test
    public void update2() throws Exception {
        SendSuccessLog sendSuccessLog = sendSuccessLogServiceJpa.getSysConfig("8a80805e6ca8b9bd016ca8b9f4ee0000");
        sendSuccessLog.setRemark1("2");
        System.err.println("开始保存");
        sendSuccessLogServiceJpa.testSysConfig2(sendSuccessLog);
    }

    @Test
    public void update4() throws Exception {
        SendSuccessLog sendSuccessLog = sendSuccessLogServiceJpa.getSysConfig("8a80805e6ca8b9bd016ca8b9f4ee0000");
        sendSuccessLog.setRemark1("5");
        System.err.println("开始保存");
        sendSuccessLogServiceJpa.testSysConfig4(sendSuccessLog);
    }

}
```

#### 创建实体

略

#### 创建数据访问接口

```java
package com.sinovoice.hcicloud.service.repository;

import com.sinovoice.hcicloud.dto.SendSuccessLog;
import com.sinovoice.hcicloud.dto.User;
import com.sinovoice.hcicloud.web.aspect.ExceptionRetry;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.stereotype.Repository;

/**
 * @Author: mhn
 * @Date: 2019/6/19 23:11
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Repository
public interface SendSuccessLogRepository extends JpaRepository<SendSuccessLog, String> {

    SendSuccessLog findByBusinessId(String businessId);
}
```

#### 定义service层接口

```java
package com.sinovoice.hcicloud.service;

import com.sinovoice.hcicloud.dto.SendSuccessLog;

/**
 * @Author: mhn
 * @Date: 2019/7/9 18:09
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
public interface SendSuccessLogServiceJpa {

    public SendSuccessLog getSysConfig(String uuid);

    public SendSuccessLog saveSysConfig(SendSuccessLog entity);

    public void testSysConfig1(SendSuccessLog entity) throws Exception;


    public void testSysConfig2(SendSuccessLog entity) throws Exception;
    public void testSysConfig3(SendSuccessLog entity) throws Exception;



    public void testSysConfig4(SendSuccessLog entity) throws Exception;
    public void testSysConfig5(SendSuccessLog entity) throws Exception;

}
```



#### 定义service层接口实现

```java
package com.sinovoice.hcicloud.service.impl;

import com.sinovoice.hcicloud.dto.SendSuccessLog;
import com.sinovoice.hcicloud.service.SendSuccessLogServiceJpa;
import com.sinovoice.hcicloud.service.repository.SendSuccessLogRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.Optional;

/**
 * @Author: mhn
 * @Date: 2019/7/9 18:10
 * @Version 1.0
 * @Software: IntelliJ IDEA
 */
@Service
public class SendSuccessLogServiceJpaImpl implements SendSuccessLogServiceJpa {

    @Autowired
    private SendSuccessLogRepository sendSuccessLogRepository;

    @Override
    public SendSuccessLog getSysConfig(String uuid) {
        Optional<SendSuccessLog> sendSuccessLogOptional = sendSuccessLogRepository.findById(uuid);
        return sendSuccessLogOptional.get();
    }

    @Override
    public SendSuccessLog saveSysConfig(SendSuccessLog entity) {
        if(entity.getOpenApiCreateTime()==null){
            entity.setOpenApiCreateTime(new Date());
        }
        return sendSuccessLogRepository.save(entity);
    }

    /**
     * 不会回滚
     * 由于 @Transactional 注解默认不捕获 检查性异常
     * @param entity
     * @throws Exception
     */
    @Override
    @Transactional
    public void testSysConfig1(SendSuccessLog entity) throws Exception {
        this.saveSysConfig(entity);
        throw new Exception("sysconfig error");
    }

    /**
     * =================================== 一个方法调用另一个方法（一） ================================
     * 方法 testSysConfig2 调用 testSysConfig3 ，因为方法 testSysConfig2 开启了事务，所以这两个方法都在事务中。
     * 不会回滚
     * 方法 testSysConfig2 的注解 @Transactional 默认不捕获 检查性异常
     * @param entity
     * @throws Exception
     */
    @Override
    @Transactional
    public void testSysConfig2(SendSuccessLog entity) throws Exception {
        //事务仍然会被提交
        this.testSysConfig3(entity);
        throw new Exception("sysconfig error");
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void testSysConfig3(SendSuccessLog entity) throws Exception {
        this.saveSysConfig(entity);
    }


    /**
     * =================================== 一个方法调用另一个方法（二） ================================
     * 方法 testSysConfig4 调用 testSysConfig5 ，因为方法 testSysConfig4 没有开启事务，所以这两个方法不在事务中。
     * 不会回滚
     * 方法 testSysConfig4 没有加事务注解，会导致 testSysConfig5 也没在事务中
     * @param entity
     * @throws Exception
     */
    @Override
    public void testSysConfig4(SendSuccessLog entity) throws Exception {
        //事务仍然会被提交
        this.testSysConfig5(entity);
        throw new Exception("sysconfig error");
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void testSysConfig5(SendSuccessLog entity) throws Exception {
        this.saveSysConfig(entity);
        throw new Exception("sysconfig error");
    }

}
```



上述定义了2种类型的事务示例

> 展示了不会回滚的3种情况

##### 一个方法

`testSysConfig1`方法中，添加注解`@Transactional`，出现`Exception`异常不会回滚，因为默认只对`RuntimeException `回滚

##### 两个方法

1.`testSysConfig2`方法中，开启事务注解`@Transactional`，`testSysConfig3`方法中也开启事务注解`@Transactional`，同时捕获`Exception`异常。这里的两个方法会产生事务传递，由于`testSysConfig2`方法开启事务，所以这两个方法处于同一个事务中。但是`testSysConfig2`方法并没有捕获`Exception`异常，这里依然不会回滚

2.`testSysConfig4`方法和`testSysConfig5`方法中都没有开启事务，所以这两个方法不在事务中，不会回滚



对于常用的Spring的@Transactional的总结如下：

- 异常在A方法内抛出，则A方法就得加注解
- 多个方法嵌套调用，如果都有 @Transactional 注解，则产生事务传递，默认 Propagation.REQUIRED
- 如果注解上只写 @Transactional  默认只对 RuntimeException 回滚，而非 Exception 进行回滚
- 如果要对 checked Exceptions 进行回滚，则需要 @Transactional(rollbackFor = Exception.class)





### 事务详解

指定不同的事务管理器之后，还能对事务进行隔离级别和传播行为的控制，下面分别详细解释：

#### 事务管理器

##### Spring-data-jpa支持

新增对第一数据源的JPA配置，注意两处注释的地方，用于指定数据源对应的`Entity`实体和`Repository`定义位置，用`@Primary`区分主数据源。

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactoryPrimary",
        transactionManagerRef="transactionManagerPrimary",
        basePackages= { "com.didispace.domain.p" }) //设置Repository所在位置
public class PrimaryConfig {

    @Autowired @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    @Primary
    @Bean(name = "entityManagerPrimary")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }

    @Primary
    @Bean(name = "entityManagerFactoryPrimary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(primaryDataSource)
                .properties(getVendorProperties(primaryDataSource))
                .packages("com.didispace.domain.p") //设置实体类所在位置
                .persistenceUnit("primaryPersistenceUnit")
                .build();
    }

    @Autowired
    private JpaProperties jpaProperties;

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Primary
    @Bean(name = "transactionManagerPrimary")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }

}
```

新增对第二数据源的JPA配置，内容与第一数据源类似，具体如下：

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactorySecondary",
        transactionManagerRef="transactionManagerSecondary",
        basePackages= { "com.didispace.domain.s" }) //设置Repository所在位置
public class SecondaryConfig {

    @Autowired @Qualifier("secondaryDataSource")
    private DataSource secondaryDataSource;

    @Bean(name = "entityManagerSecondary")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactorySecondary(builder).getObject().createEntityManager();
    }

    @Bean(name = "entityManagerFactorySecondary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactorySecondary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(secondaryDataSource)
                .properties(getVendorProperties(secondaryDataSource))
                .packages("com.didispace.domain.s") //设置实体类所在位置
                .persistenceUnit("secondaryPersistenceUnit")
                .build();
    }

    @Autowired
    private JpaProperties jpaProperties;

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Bean(name = "transactionManagerSecondary")
    PlatformTransactionManager transactionManagerSecondary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactorySecondary(builder).getObject());
    }

}
```

完成了以上配置之后，主数据源的实体和数据访问对象位于：`com.didispace.domain.p`，次数据源的实体和数据访问接口位于：`com.didispace.domain.s`。

分别在这两个package下创建各自的实体和数据访问接口

- 主数据源下，创建User实体和对应的Repository接口

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer age;

    public User(){}

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    // 省略getter、setter

}
public interface UserRepository extends JpaRepository<User, Long> {

}
```

- 从数据源下，创建Message实体和对应的Repository接口

```java
@Entity
public class Message {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String content;

    public Message(){}

    public Message(String name, String content) {
        this.name = name;
        this.content = content;
    }

    // 省略getter、setter

}
public interface MessageRepository extends JpaRepository<Message, Long> {

}
```

接下来通过测试用例来验证使用这两个针对不同数据源的配置进行数据操作。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {

	@Autowired
	private UserRepository userRepository;
	@Autowired
	private MessageRepository messageRepository;

	@Test
	public void test() throws Exception {

		userRepository.save(new User("aaa", 10));
		userRepository.save(new User("bbb", 20));
		userRepository.save(new User("ccc", 30));
		userRepository.save(new User("ddd", 40));
		userRepository.save(new User("eee", 50));

		Assert.assertEquals(5, userRepository.findAll().size());

		messageRepository.save(new Message("o1", "aaaaaaaaaa"));
		messageRepository.save(new Message("o2", "bbbbbbbbbb"));
		messageRepository.save(new Message("o3", "cccccccccc"));

		Assert.assertEquals(3, messageRepository.findAll().size());

	}

}
```

#### 隔离级别

隔离级别是指若干个并发的事务之间的隔离程度，与我们开发时候主要相关的场景包括：脏读取、重复读、幻读。

我们可以看`org.springframework.transaction.annotation.Isolation`枚举类中定义了五个表示隔离级别的值：

```
public enum Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);
}
```

- `DEFAULT`：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是：`READ_COMMITTED`。
- `READ_UNCOMMITTED`：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读和不可重复读，因此很少使用该隔离级别。
- `READ_COMMITTED`：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
- `REPEATABLE_READ`：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。即使在多次查询之间有新增的数据满足该查询，这些新增的记录也会被忽略。该级别可以防止脏读和不可重复读。
- `SERIALIZABLE`：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

指定方法：通过使用`isolation`属性设置，例如：

```
@Transactional(isolation = Isolation.DEFAULT)
```

#### 传播行为

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。

我们可以看`org.springframework.transaction.annotation.Propagation`枚举类中定义了6个表示传播行为的枚举值：

```
public enum Propagation {
    REQUIRED(0),
    SUPPORTS(1),
    MANDATORY(2),
    REQUIRES_NEW(3),
    NOT_SUPPORTED(4),
    NEVER(5),
    NESTED(6);
}
```

- `REQUIRED`：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- `SUPPORTS`：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- `MANDATORY`：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
- `REQUIRES_NEW`：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- `NOT_SUPPORTED`：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- `NEVER`：以非事务方式运行，如果当前存在事务，则抛出异常。
- `NESTED`：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于`REQUIRED`。

指定方法：通过使用`propagation`属性设置，例如：

```
@Transactional(propagation = Propagation.REQUIRED)
```



##### 事务配置示例

Spring默认情况下会对运行期例外(RunTimeException)，即uncheck异常，进行事务回滚。

如果遇到checked异常就不回滚。
如何改变默认规则：

- 让checked例外也回滚：在整个方法前加上 @Transactional(rollbackFor=Exception.class)

- 让unchecked例外不回滚： @Transactional(notRollbackFor=RunTimeException.class)

- 不需要事务管理的(只查询的)方法：@Transactional(propagation=Propagation.NOT_SUPPORTED)

##### 注意事项

1. 在需要事务管理的地方加@Transactional 注解。@Transactional 注解可以被应用于接口定义和接口方法、类定义和类的 public 方法上 。

2. @Transactional 注解只能应用到 public 可见度的方法上 。 如果你在 protected、private 或者 package-visible 的方法上使用 @Transactional 注解，它也不会报错， 但是这个被注解的方法将不会展示已配置的事务设置。
3. 通过 元素的 "proxy-target-class" 属性值来控制是基于接口的还是基于类的代理被创 建。 如果 "proxy-target-class" 属值被设置为 "true"，那么基于类的代理将起作用（这时需要CGLIB库cglib.jar在CLASSPATH中）。如果 "proxy-target-class" 属值被设置为 "false" 或者这个属性被省略，那么标准的JDK基于接口的代理将起作用。注解@Transactional cglib与java动态代理最大区别是代理目标对象不用实现接口, 那么注解要是写到接口方法上，要是使用cglib代理，这是注解事物就失效了，为了保持兼容注解最好都写到实现类方法上。

### 异常详解

> 那么什么是检查型异常什么又是非检查型异常呢？

最简单的判断点有两个：

1. 继承自runtimeexception或error的是非检查型异常，而继承自exception的则是检查型异常（当然，runtimeexception本身也是exception的子类）。

2. 对非检查型类异常可以不用捕获，而检查型异常则必须用try语句块进行处理或者把异常交给上级方法处理总之就是必须写代码处理它。所以必须在service捕获异常，然后再次抛出，这样事务方才起效。

#### 源码分析

![1566810895.png](https://upload-images.jianshu.io/upload_images/539247-37d0da9acf154478.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

	private static final String ID_MUST_NOT_BE_NULL = "The given id must not be null!";
	
	…………
   	
    @Transactional
	public void deleteById(ID id) {
		Assert.notNull(id, ID_MUST_NOT_BE_NULL);
		delete(findById(id).orElseThrow(() -> new EmptyResultDataAccessException(
				String.format("No %s entity with id %s exists!", entityInformation.getJavaType(), id), 1)));
	}
    	
    
    @Override
	public T getOne(ID id) {
		Assert.notNull(id, ID_MUST_NOT_BE_NULL);
		return em.getReference(getDomainClass(), id);
	}
    
```

`SimpleJpaRepository`在类上添加注解`@Transactional(readOnly = true)`，默认`get*`开头的方法都是只读模式，不能更新，删除，而`save*`,`delete*`等方法都再次添加注解`@Transactional`，`readOnly`默认值为`false`



以上有部分内容参考[Spring Boot多数据源配置与使用](<http://blog.didispace.com/springbootmultidatasource/>)