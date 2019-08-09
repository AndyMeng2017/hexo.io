---
layout: post
title: 微服务系列（一）分布式配置中心apollo实战
date: 2019-8-9 19:07:36
categories: blog
tags: [apollo,xxl-job,prometheus,metrics,微服务]
photos:
	- gallery/01.jpg
description: 整体部署方案
---

# 部署文档

## 一.整体部署图

![1563972872967](https://upload-images.jianshu.io/upload_images/539247-c805213dd3f09f8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 

主要分为4部分

- 外部服务组件，包括apollo，xxl-job，zookeeper

- 数据交互组件，包括elasticsearch，redis，mysql，oracle，prometheus

- 能力服务组件，包括asr，ta，nlu

- 外部接口服务，包括平安数据拉取上报接口，友邻调用接口

- 用户展示组件，包括kibana，grafana

- 业务组件，包括处理成功单业务


<!-- more -->

## 二.各个组件功能以及部署说明

### apollo安装与部署

#### 1.客户端环境配置

> 本文档只选用其中一种配置方式，其他方式请参见官方文档

##### 1.1 本地缓存环境

- **Mac/Linux**: `/opt/data/{appId}/config-cache`
- **Windows**: `C:\opt\data\{appId}\config-cache`

![1561369137089](https://upload-images.jianshu.io/upload_images/539247-04485f3214daed06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 1.2 可选配置 

- 对于**Mac/Linux**，文件位置为`/opt/settings/server.properties`
- 对于**Windows**，文件位置为`C:\opt\settings\server.properties`

###### 1.2.1 Environment 文件内容形式为：

- `env=DEV`

###### 1.2.2 Cluster集群 文件内容形式为：

- `idc=beijing`

![1561369026561](https://upload-images.jianshu.io/upload_images/539247-c7e475592e1ed0b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 服务端部署

线上结构图如下：

![1563975022206](https://upload-images.jianshu.io/upload_images/539247-099c029673764a71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2.1 导入两张库表文件

###### 2.1.1 修改ApolloPorTalDB

可支持的环境列表`ApolloPorTalDB#ServerConfig`

这里的环境`value`要和`scripts/build.sh`脚本中的`dev_meta`个数保持一致

| key                | value | comment          |
| ------------------ | ----- | ---------------- |
| apollo.portal.envs | dev   | 可支持的环境列表 |

###### 2.1.2 修改ApolloConfigDB

可支持的环境列表`ApolloConfigDB#ServerConfig`

这里的环境`value`要和部署的`config-service`的个数一致，其实就是对外暴露出多个`eureka`服务，保证客户端拉取配置信息高可用

| key                | cluster | value                         | comment                                  |
| ------------------ | ------- | ----------------------------- | ---------------------------------------- |
| eureka.service.url | default | http://localhost:8080/eureka/ | Eureka服务Url，多个service以英文逗号分隔 |

##### 2.2 配置数据库连接信息和meta service地址

- 修改`scripts/build.sh`

```properties
#apollo config db info
apollo_config_db_url=jdbc:mysql://localhost:3306/ApolloConfigDB?useSSL=false&characterEncoding=utf8
apollo_config_db_username=用户名
apollo_config_db_password=密码（如果没有密码，留空即可）

# apollo portal db info
apollo_portal_db_url=jdbc:mysql://localhost:3306/ApolloPortalDB?useSSL=false&characterEncoding=utf8
apollo_portal_db_username=用户名
apollo_portal_db_password=密码（如果没有密码，留空即可）

# 如果某个环境不需要，也可以直接删除对应的配置项
dev_meta=http://1.1.1.1:8080
#fat_meta=http://apollo.fat.xxx.com
#uat_meta=http://apollo.uat.xxx.com
#pro_meta=http://apollo.xxx.com

META_SERVERS_OPTS="-Ddev_meta=$dev_meta -Dfat_meta=$fat_meta -Duat_meta=$uat_meta -Dpro_meta=$pro_meta"
```

##### 2.3 执行编译，打zip包

执行命令`/build.sh`

##### 2.4 部署启动zip包

###### 2.4.1 apollo-configservice-1.4.0-github.zip 

记得需要修改`scripts/startup.sh`文件的启动端口

```shell
# 解压后的结构
PAGLJTDB1:/home/apollo/apollo-configservice-1.4.0-github # ll
total 121080
-rwxr-xr-x 1 apollo users    39055 Jul 17 21:08 apollo-configservice-1.4.0-sources.jar
-rwxr-xr-x 2 apollo users 61896951 Jul 17 21:08 apollo-configservice-1.4.0.jar
-rw-r--r-- 1 apollo users       57 May  3 12:13 apollo-configservice.conf
-rwxr-xr-x 2 apollo users 61896951 Jul 17 21:08 apollo-configservice.jar
-rw-r--r-- 1 apollo users        6 Jul 17 21:16 apollo-configservice_homeapolloapollo-configservice-1.4.0-github.pid
drwxr-xr-x 2 apollo users     4096 Jul 17 21:58 config
drwxr-xr-x 2 apollo users     4096 Jul 17 21:15 scripts

# 查看config目录下的文件
PAGLJTDB1:/home/apollo/apollo-configservice-1.4.0-github # ll config/
total 8
-rw-r--r-- 1 apollo users  30 May  3 12:13 app.properties
-rw-r--r-- 1 apollo users 190 Jul 17 21:16 application-github.properties


# 查看scripts目录下的文件
PAGLJTDB1:/home/apollo/apollo-configservice-1.4.0-github # ll scripts/
total 12
-rwxr-xr-x 1 apollo users  340 May  3 12:13 shutdown.sh
-rwxr-xr-x 1 apollo users 5084 Jul 17 21:15 startup.sh

# 查看config目录下文件的内容
PAGLJTDB1:/home/apollo/apollo-configservice-1.4.0-github/config # cat app.properties 
appId=100003171
jdkVersion=1.8


PAGLJTDB1:/home/apollo/apollo-configservice-1.4.0-github/config # cat application-github.properties 
# DataSource
spring.datasource.url = jdbc:mysql://192.169.51.1:3306/ApolloConfigDB?useSSL=false&characterEncoding=utf8 
spring.datasource.username = root
spring.datasource.password = 123456
```

###### 2.4.2 apollo-adminservice-1.4.0-github.zip

记得需要修改`scripts/startup.sh`文件的启动端口

```shell
# 解压后的结构
PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github # ll
total 113964
-rwxr-xr-x 1 apollo users    26057 Jul 17 21:08 apollo-adminservice-1.4.0-sources.jar
-rwxr-xr-x 2 apollo users 58265253 Jul 17 21:08 apollo-adminservice-1.4.0.jar
-rw-r--r-- 1 apollo users       57 May  3 12:13 apollo-adminservice.conf
-rwxr-xr-x 2 apollo users 58265253 Jul 17 21:08 apollo-adminservice.jar
-rw-r--r-- 1 apollo users        6 Jul 17 21:18 apollo-adminservice_homeapolloapollo-adminservice-1.4.0-github.pid
drwxr-xr-x 2 apollo users     4096 Jul 17 21:18 config
drwxr-xr-x 2 apollo users     4096 Jul 17 21:18 scripts

# 查看config目录下的文件
PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github # ll config/
total 8
-rw-r--r-- 1 apollo users  30 May  3 12:13 app.properties
-rw-r--r-- 1 apollo users 189 Jul 17 21:18 application-github.properties


# 查看scripts目录下的文件
PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github # ll scripts/
total 12
-rwxr-xr-x 1 apollo users  339 May  3 12:13 shutdown.sh
-rwxr-xr-x 1 apollo users 5083 Jul 17 21:18 startup.sh

# 查看config目录下文件的内容
PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github/config # cat app.properties 
appId=100003172
jdkVersion=1.8


PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github/config # 
PAGLJTDB1:/home/apollo/apollo-adminservice-1.4.0-github/config # cat application-github.properties 
# DataSource
spring.datasource.url = jdbc:mysql://192.169.51.1:3306/ApolloConfigDB?useSSL=false&characterEncoding=utf8
spring.datasource.username = root
spring.datasource.password = 123456
```

###### 2.4.3 apollo-portal-1.4.0-github.zip

需要修改`scripts/startup.sh`文件的启动端口

需要修改文件`apollo-env.properties `保证和可支持的环境列表一致

```shell
# 解压后的结构
PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github # ll
total 83656
-rwxr-xr-x 1 apollo users  1140951 Jul 17 21:54 apollo-portal-1.4.0-sources.jar
-rwxr-xr-x 2 apollo users 42197183 Jul 17 21:54 apollo-portal-1.4.0.jar
-rw-r--r-- 1 apollo users       57 May  3 12:13 apollo-portal.conf
-rwxr-xr-x 2 apollo users 42197183 Jul 17 21:54 apollo-portal.jar
-rw-r--r-- 1 apollo users        5 Jul 19 10:57 apollo-portal_homeapolloapollo-portal-1.4.0-github.pid
drwxr-xr-x 2 apollo users     4096 Jul 19 10:57 config
drwxr-xr-x 2 apollo users     4096 Jul 17 21:57 scripts

# 查看config目录下的文件
PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github # ll config/
total 12
-rw-r--r-- 1 apollo users 223 Jul 19 10:57 apollo-env.properties
-rw-r--r-- 1 apollo users  30 May  3 12:13 app.properties
-rw-r--r-- 1 apollo users 190 Jul 17 22:15 application-github.properties


# 查看scripts目录下的文件
PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github # ll scripts/
total 12
-rwxr-xr-x 1 apollo users  333 May  3 12:13 shutdown.sh
-rwxr-xr-x 1 apollo users 5077 Jun 21 18:41 startup.sh

# 查看config目录下文件的内容，这里一定要和编译打包时的meta一致
PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github/config # cat apollo-env.properties 
#local.meta=http://localhost:8080
dev.meta=http://1.1.1.1:8080
#fat.meta=
#uat.meta=
#lpt.meta=${lpt_meta}
#pro.meta=http://192.169.51.1:8050,http://192.169.51.2:8050
#pro.meta=http://apollo-configservice 这种配置不支持，除非走slb



PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github/config # cat app.properties 
appId=100003173
jdkVersion=1.8

PAGLJTDB1:/home/apollo/apollo-portal-1.4.0-github/config # cat application-github.properties 
# DataSource
spring.datasource.url = jdbc:mysql://192.169.51.1:3306/ApolloPortalDB?useSSL=false&characterEncoding=utf8 
spring.datasource.username = root
spring.datasource.password = 123456


```



#### 3.整合springboot

##### Maven Dependency

```xml
<dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-client</artifactId>
    <version>1.4.0</version>
</dependency>
```

##### 3.1 AppId

通过`app.properties`配置文件

- 可以在`classpath:/META-INF/app.properties`指定：`app.id=YOUR-APP-ID`

##### 3.2 Apollo Meta Server

默认情况下，meta server和config service是部署在同一个JVM进程，所以meta server的地址就是config service的地址。

通过`app.properties`配置文件（可以配置为SLB方式）

- 可以在`classpath:/META-INF/app.properties`指定：`apollo.meta=http://config-service-url`

假设要支持多个ip，从0.11.0版本开始支持填入以逗号分隔的多个地址（[PR #1214](https://github.com/ctripcorp/apollo/pull/1214)），如`http://1.1.1.1:8080,http://2.2.2.2:8080`，不过生产环境还是建议使用域名（走slb），因为机器扩容、缩容等都可能导致IP列表的变化。

##### 3.3 配置`app.properties` 文件

```properties
app.id=open-api-vts
apollo.meta=http://localhost:8080
```

##### 3.4 修改 `application.properties`

```properties
# 定义本地服务名称
spring.application.name = open-api-vts

# will inject 'application' namespace in bootstrap phase
apollo.bootstrap.enabled = true
# will inject 'application', 'FX.apollo' and 'application.yml' namespaces in bootstrap phase
#apollo.bootstrap.namespaces = application,FX.apollo,application.yml
apollo.bootstrap.namespaces = application
```

##### 3.5 在apollo的配置页面进行配置

###### 3.5.1 默认portal的端口为：8070，点击创建项目

![1561368193769](https://upload-images.jianshu.io/upload_images/539247-9b34482d8999756f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 3.5.2  这里的部门可以手动配置，`管理员工具 -> 系统参数`可以修改

![1561368251174](https://upload-images.jianshu.io/upload_images/539247-8c671d2e6ef6cf24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的key为：`ApolloPortalDB#ServerConfig` 数据库表中的字段

![1561368441620](https://upload-images.jianshu.io/upload_images/539247-da1e9cb8c48a1607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 3.5.3 新增配置，然后进行发布

![1561368752517](https://upload-images.jianshu.io/upload_images/539247-da929da513a8d1e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### xxl-job安装与部署

#### 1.客户端环境配置

无

#### 2.服务端部署

线上结构图如下：

![1563978460854](https://upload-images.jianshu.io/upload_images/539247-1722785d63176ec0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2.1 修改application.properties

修改启动端口以及数据库配置信息

```
### web
server.port=8082
server.context-path=/xxl-job-admin

……

### xxl-job, datasource
spring.datasource.url=jdbc:mysql://192.169.51.1:3306/xxl-job?Unicode=true&characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
……
```

##### 2.2 修改logback.xml

修改默认的日志文件地址为：`/opt/logs/xxl-job/xxl-job-admin.log`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false" scan="true" scanPeriod="1 seconds">

    <contextName>logback</contextName>
    <property name="log.path" value="/opt/logs/xxl-job/xxl-job-admin.log"/>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
	<appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}.%d{yyyy-MM-dd}.zip</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>%date %level [%thread] %logger{36} [%file : %line] %msg%n
            </pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </root>

</configuration>
```



##### 2.3 修改pom.xml

添加servlet依赖

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>RELEASE</version>
</dependency>
```

##### 2.4 执行编译，打jar包

在根目录下执行，打包命令：`mvn clean package -DskipTests `

##### 2.5 部署启动

后台启动，日志生成在`logback.xml`配置的目录里

`nohup java -jar xxl-job-admin-2.1.0.jar >/dev/null 2>&1 &`

#### 3.整合springboot

##### Maven Dependency

```xml
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>${xxl-job-core.version}</version>
</dependency>
```

##### 3.1 修改application.properties

```properties
#xxl-job
### xxl-job admin address list, such as "http://address" or "http://address01,http://address02"
xxl.job.admin.addresses = http://192.169.51.1:8082/xxl-job-admin
# 这里的appname是页面中的执行器名字
xxl.job.executor.appname = xxl-job-executor-zhw
xxl.job.executor.ip = 10.137.87.168
xxl.job.executor.port = 9999
xxl.job.accessToken = 
# 日志位置是：job executor生成日志的位置
xxl.job.executor.logpath = /opt/logs/xxl-job/jobhandler
### xxl-job log retention days
xxl.job.executor.logretentiondays = -1
```

##### 3.2 XxlJobConfig 配置类

```java
import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.executor.appname}")
    private String appName;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
	
    /**
     * 针对多网卡、容器内部署等情况，可借助 "spring-cloud-commons" 提供的 "InetUtils" 组件灵活定制注册IP；
     *
     *      1、引入依赖：
     *          <dependency>
     *             <groupId>org.springframework.cloud</groupId>
     *             <artifactId>spring-cloud-commons</artifactId>
     *             <version>${version}</version>
     *         </dependency>
     *
     *      2、配置文件，或者容器启动变量
     *          spring.cloud.inetutils.preferred-networks: 'xxx.xxx.xxx.'
     *
     *      3、获取IP
     *          String ip_ = inetUtils.findFirstNonLoopbackHostInfo().getIpAddress();
     */
}

```



##### 3.3 编写jobHandler

```java
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.handler.IJobHandler;
import com.xxl.job.core.handler.annotation.JobHandler;
import com.xxl.job.core.log.XxlJobLogger;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

/**
 * 任务Handler示例（Bean模式）
 *
 * 开发步骤：
 * 1、继承"IJobHandler"：“com.xxl.job.core.handler.IJobHandler”；
 * 2、注册到Spring容器：添加“@Component”注解，被Spring容器扫描为Bean实例；
 * 3、注册到执行器工厂：添加“@JobHandler(value="自定义jobhandler名称")”注解，注解value值对应的是调度中心新建任务的JobHandler属性的值。
 * 4、执行日志：需要通过 "XxlJobLogger.log" 打印执行日志；
 *
 * @author xuxueli 2015-12-19 19:43:36
 */
@Slf4j
@JobHandler(value="demoJobHandler")
@Component
public class DemoJobHandler extends IJobHandler {

    @Override
    public ReturnT<String> execute(String param) throws Exception {
        XxlJobLogger.log("XXL-JOB, Hello World.");

        for (int i = 0; i < 5; i++) {
            XxlJobLogger.log("beat at:" + i);
            TimeUnit.SECONDS.sleep(2);
            log.info("beat at:" + i);
        }
        log.info("定时任务执行完毕");
        return SUCCESS;
    }

}
```

##### 3.4 在xxl-job配置页面进行配置

###### 3.4.1 新增执行器 

AppName与上面配置中的一致

![1563978153968](https://upload-images.jianshu.io/upload_images/539247-c1996d9ce3304823.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 3.4.2 新增任务

![1563978234344](https://upload-images.jianshu.io/upload_images/539247-81e16f681da1c2e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 3.4.3 具体任务的配置

![1563978269858](https://upload-images.jianshu.io/upload_images/539247-ae498e61d0523d1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 3.zookeeper安装与部署

#### 3.1 客户端环境配置

无

#### 3.2 服务端部署

线上结构图如下：

![1564018962177](https://upload-images.jianshu.io/upload_images/539247-7c8285e3eb650ab8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 3.2.1 下载安装包

官网下载安装包：`zookeeper-3.4.10.tar.gz`

##### 3.2.2 修改配置文件

这里修改`zoo.cfg`文件

主要修改 `dataDir`，` dataLogDir`，`clientPort`，以及集群配置项

```properties

# The number of milliseconds of each tick
# 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
# 用来配置Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 5 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒。
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
# 标识Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒。
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
# Zookeeper 保存数据的目录
dataDir=/home/skywalking/zookeeper-3.4.10_data
# Zookeeper 保存日志文件的目录
dataLogDir=/home/skywalking/zookeeper-3.4.10_data/log
# the port at which the clients will connect
# 客户端连接Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
4lw.commands.whitelist=*

# 集群配置项
server.1=10.0.1.227:2888:3888
server.2=10.0.1.227:2889:3889
server.3=10.0.1.227:2890:3890

```

##### 3.2.3 集群部署

在路径`/usr/local/zookeeper/dataDir`创建`myid`文件，内容为 `1` or `2` or `3`

##### 3.2.4 启动

`zkServer.sh start` 启动服务

`zkServer.sh stop` 关闭服务

`jps`命令查看，存在QuorumPeerMain进程，表示Zookeeper已经启动

##### 3.2.5 客户端连接

`./zkCli.sh -server localhost:2182`

#### 3.3 整合springboot

##### Maven Dependency

```xml
<!-- zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>${zookeeper.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

##### 3.3.1 修改`application.properties`

##### 3.3.2 配置类

```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import javax.validation.constraints.NotNull;

/**
 * @author ynz
 * @version 创建时间：2018/7/2
 * @email ynz@myframe.cn
 */
@ConfigurationProperties(prefix="zookeeper")
@Data
@Validated
public class ZookeeperProperties {

	@NotNull(message = "zookeeper服务地址不能为空")
    private String server;
	@NotNull(message = "namespace不能为空")
    private String namespace;
    private String digest;
    private Integer sessionTimeoutMs = 60000;
    private Integer connectionTimeoutMs = 6000;
    private Integer maxRetries = 3;
    private Integer baseSleepTimeMs = 1000;
}
```

##### 3.3.4 启动类

```
import com.sinovoice.hcicloud.common.properties.ZookeeperProperties;
import com.sinovoice.hcicloud.common.zookeeper.ZkClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties({ZookeeperProperties.class})
@ConditionalOnProperty(  //配置文件属性是否为true
        value = {"zookeeper.enabled"},
        matchIfMissing = false
)
public class ZkConfig {

    @Autowired
    ZookeeperProperties zookeeperProperties;

    @Bean(initMethod = "init", destroyMethod = "stop")
    public ZkClient zkClient() {
        ZkClient zkClient = new ZkClient(zookeeperProperties);
        return zkClient;
    }
}
```

##### 3.3.5 客户端集成类

代码太长，略，参见 `ZkClient.java`



### 4.prometheus安装与部署

#### 4.1 客户端环境配置

无

#### 4.2 服务端部署

线上结构图如下：

![1564020968651](https://upload-images.jianshu.io/upload_images/539247-7aa2eb55c9630666.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 4.2.1 下载安装包

官网下载安装包：`prometheus-2.11.1.linux-amd64.tar.gz`

##### 4.2.2 修该配置文件

修改`prometheus.yml`文件

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    scrape_interval: 5s # 重写了全局抓取间隔时间，由15秒重写成5秒

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'open-api-vts-wec'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    metrics_path: '/actuator/prometheus'
    static_configs:
    - targets: ['192.169.51.30:8080']

  - job_name: 'open-api-vts-zhw'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    metrics_path: '/actuator/prometheus'
    static_configs:
    - targets: ['10.137.87.168:8081']
```

##### 4.2.3 启动

```yaml
# 可选参数 --storage.tsdb.path=存储数据的路径，默认路径为./data
./prometheus --config.file=prometheus.yml
```

启动命令 `nohup ./prometheus &`

#### 4.3 整合springboot

##### Maven Dependency

```
        <!--监控-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        <dependency>
            <groupId>io.github.mweirauch</groupId>
            <artifactId>micrometer-jvm-extras</artifactId>
            <version>${micrometer.jvm.extras.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
            <scope>true</scope>
        </dependency>
```

##### 4.3.1 修改`application.properties`

```properties
#监控
management.endpoints.web.exposure.include = *
management.metrics.tags.application = ${spring.application.name}
```



##### 4.3.2 启动类

```java
import com.sinovoice.hcicloud.common.properties.PinganProperties;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;

/**
 * 使用示例：@EnableAsync 在方法上标注 @Async
 *
 * @author menghaonan
 * @date 2019年6月19日15:49:05
 */
@SpringBootApplication
@EnableConfigurationProperties(PinganProperties.class)
@EnableAsync
public class HcicloudApplication {

    /** 启动方式
     // 1.直接运行spring boot的应用主类
     // 2.mvn spring-boot:run
     // 3.java -jar ***.jar
     * @param args
     */
    public static void main(String[] args) {
        SpringApplication.run(HcicloudApplication.class, args);
    }

    @Bean
    MeterRegistryCustomizer<MeterRegistry> configurer(@Value("${spring.application.name}") String applicationName) {
        return (registry) -> registry.config().commonTags("application", applicationName);
    }

}
```



##### 4.3.3 监控项配置

参考`Meter.java`查看有哪些监控项

```
    /**
     * Custom meters may emit metrics like one of these types without implementing
     * the corresponding interface. For example, a heisen-counter like structure
     * will emit the same metric as a {@link Counter} but does not have the same
     * increment-driven API.
     */
    enum Type {
        COUNTER,
        GAUGE,
        LONG_TASK_TIMER,
        TIMER,
        DISTRIBUTION_SUMMARY,
        OTHER;
    }
```



###### 4.3.3.1 `COUNTER` 计数器

> 计数器，只增不减，适用于一些增长类型的统计，例如下单、支付次数、Http请求总量记录等等

`demo`案例：

指标项定义：

```java
@Component
public class DemoMetrics implements MeterBinder {
    // 只增加
    public Counter jobCounter;
    
    @Override
    public void bindTo(MeterRegistry meterRegistry) {
        this.jobCounter = Counter.builder("charles_demo_counter")
                .tags(new String[]{"name", "charles_demo1"})
                .description("pay execute count")
                .register(meterRegistry);
}
```

指标项赋值：

```java
@RestController
@Slf4j
public class DemoController {

    @Autowired
    private DemoMetrics demoMetrics;
    
    @RequestMapping("/pay")
    public void pay() {
        demoMetrics.jobCounter.increment();
    }
}
```



###### 4.3.3.2 `GAUGE`仪表盘

>  当前状态，可增可减，例如：当前的内存使用情况，同时也可以测量上下移动的”计数”，比如队列中的消息数量

`demo`案例：

指标项定义：

```java
@Component
public class DemoMetrics implements MeterBinder {
    // 只增加
    public Map<String, Double> map;
    
    public DemoMetrics() {
        map = new HashMap<>();
        map.put("x", (double) 5);
    }
    
    @Override
    public void bindTo(MeterRegistry meterRegistry) {
        Gauge.builder("charles_job_gauge", map, x -> x.get("x"))
                .tag("name", "gauge_job1")
                .description("XXXXX")
                .register(meterRegistry);
}
```

指标项赋值：

```java
@RestController
@Slf4j
public class DemoController {

    @Autowired
    private DemoMetrics demoMetrics;
    
    @RequestMapping("/createOrder")
    public void createOrder() {
        demoMetrics.map.put("x", Double.valueOf(Math.random() * 10));
    }
}
```



###### 4.3.3.3 `LONG_TASK_TIMER`

> Timer的实现至少记录了发生的事件的数量和这些事件的总耗时
>
> LongTaskTimer适合用于长时间持续运行的事件耗时的记录，例如相对耗时的定时任务
>
> 这里会生成直方图区间来进行统计，实测中发现有69个bucket

这里的截图不太清晰，主要统计到`mhn_demo_controller_longtask_time_seconds_bucket`的区间个数为：69

![1564044175859](https://upload-images.jianshu.io/upload_images/539247-f39e13080b2c7d54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`demo`案例：

指标项定义：

无

指标项赋值：

```java
@RestController
@Slf4j
public class DemoController {

    @Autowired
    private DemoMetrics demoMetrics;
    
     /**
     *  Timer 任务计时器，开启直方图
     *
     * @return
     */
    @Timed(value = "mhn_demo_controller_longtask_time",
            longTask = true,
            histogram = true,
            extraTags = {"name", "controller_thread_test"},
            description = "The time the thread has spent")
    @GetMapping("/taskTimer")
    public String taskTimer() {
        try {
            Thread.sleep(1000 * ((int) Math.random() * 10));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("执行完毕");
        return "SUCCESS";
    }
}
```



> 这里详细描述下Timer直方图的作用

- 过去5分钟内的平均请求持续时间

` rate(mhn_demo_controller_longtask_time_seconds_sum[5m])/rate(mhn_demo_controller_longtask_time_seconds_count[5m])`

- 按照`job`维度，统计过去5分钟内的平均请求持续时间

` sum(rate(mhn_demo_controller_longtask_time_seconds_sum[5m])) by (job)/sum(rate(mhn_demo_controller_longtask_time_seconds_count[5m])) by (job)`

- 统计一个近似`Apdex score`指标的值

> 满意: 这样的响应时间让用户感到很愉快，例如少于3秒钟。
>
> 容忍: 慢了一点，但还可以接受，继续这一应用过程，例如3～12秒。
>
> 失望: 太慢了，受不了了，用户决定放弃这个应用，例如超过12秒。

```
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job)
+
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m])) by (job)
) / 2 / sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

- 聚合每一个水桶的第95%位的值

`histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`



###### 4.3.3.4 `DISTRIBUTION_SUMMARY`

> 使用DistributionSummary测量命中服务器的请求的有效负载大小



##### 4.4 监控指标界面

`http://192.169.51.30:8080//actuator/prometheus`

### 5.grafana安装与部署

#### 5.1 客户端环境配置

无

#### 5.2 服务端部署

##### 5.2.1 下载安装包

官网下载安装包 `grafana-6.2.5.linux-amd64.tar.gz`

##### 5.2.2 启动

启动命令 `nohup ./grafana-server &` 

默认的启动端口为3000，初始的账号密码都为admin，权限是管理员权限。

##### 5.2.3 图解4701

![640-1.jpg](https://upload-images.jianshu.io/upload_images/539247-04d0cc608658345a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_1_.jpg](https://upload-images.jianshu.io/upload_images/539247-54375a765c44bfb6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_2_.jpg](https://upload-images.jianshu.io/upload_images/539247-291a9e0b07ea1cd1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_3_.jpg](https://upload-images.jianshu.io/upload_images/539247-eb37afc2983f8ab7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_4_.jpg](https://upload-images.jianshu.io/upload_images/539247-bdb349a2ff76b0bc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![640-_5_.jpg](https://upload-images.jianshu.io/upload_images/539247-3c5c6a63122358f3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_6_.jpg](https://upload-images.jianshu.io/upload_images/539247-27e833c83ef9af4e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![640-_7_.jpg](https://upload-images.jianshu.io/upload_images/539247-070a578d749fab37.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 6.open-api-vts安装与部署

#### 6.1 本地打包

##### 6.1.1 修改`app.properties`

指定`apollo`的`eureka`的地址

```properties
app.id=open-api-vts
apollo.meta=http://192.169.51.1:8050,http://192.169.51.2:8050
#apollo.meta=http://10.0.0.222:8080

```

##### 6.1.2 打包

打包命令为：`mvn clean package -DskipTests`

这里是打出`zip`包

#### 6.2 线上部署

##### 6.2.1 新建用户

`useradd -m mhn`

自动创建用户的根目录`/home/mhn`

修改密码 `passwd mhn`

密码统一为：`apollo_Pingan123`

##### 6.2.2 搭建jdk环境

修改`.profile`文件

```properties
export JAVA_HOME="/home/apollo/jdk1.8.0_144"
export JRE_HOME=${JAVA_HOME}/jre
export PATH=${JAVA_HOME}/bin:$PATH
export CLASSPATH=.:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export JAVA_HOME JRE_HOME CLASSPATH PATH
```

`source .profile` 刷新生效

##### 6.2.3 新建日志目录

`/opt/data` 配置的缓存文件

`/opt/logs` 日志文件

```shell
# 新建目录
linux-k7mb:/opt # mkdir /opt/data
linux-k7mb:/opt # mkdir /opt/logs

# 修改权限
linux-k7mb:/opt # chmod 777 data
linux-k7mb:/opt # chmod 777 logs
```



##### 6.2.4 修改启动脚本权限

```shell
cd /home/apollo/xqd-ccodszdx/scripts


chmod 755 startup.sh
chmod 755 shutdown.sh
```

##### 6.2.5 修改jar所属用户

```
chown apollo:users open-api-vts-2.1.6.jar

chmod 755 open-api-vts-2.1.6.jar
```

##### 6.2.6 新建录音目录

这里以中兴平台为例：

```shell
mkdir -p  success_voice/xqd-zx
mkdir -p  annoy_voice/xqd-zx
mkdir -p  upload_voice/xqd-zx
```

##### 6.2.7 新建`open-api-vts.conf`文件

>  其实这个文件已经存在在`zip`里

这里创建的`conf`文件名，一定要和要启动的`.jar`的名称相同

```properties
MODE=service
PID_FOLDER=.
LOG_FOLDER=/opt/logs/open-api/
```

##### 6.2.8 修改`application.properties`

修改`apollo.bootstrap.namespaces`指向`apollo`配置的`namespace`

以ccodszdx为例，`application-***`对应到`application-ccodszdx`

```properties
# 定义本地服务名称
spring.application.name = open-api-vts

# will inject 'application' namespace in bootstrap phase
apollo.bootstrap.enabled = true
# put apollo initialization before logging system initialization
apollo.bootstrap.eagerLoad.enabled=true
# will inject 'application', 'FX.apollo' and 'application.yml' namespaces in bootstrap phase
#apollo.bootstrap.namespaces = application,FX.apollo,application.yml
apollo.bootstrap.namespaces = application-***
```

##### 6.2.9 修改启动脚本`startup.sh`

务必修改`SERVER_PORT`和`apollo`里的一致

环境这里要对应到`apollo`可支持的环境列表

```shell
## Adjust server port if necessary
SERVER_PORT=8081
YOUR_ENVIRONMENT=PRO
```

##### 6.2.10 修改线上apollo配置

```properties
# 服务的端口
server.port = 8081

# 数据库mysql配置
spring.datasource.url = jdbc:mysql://192.169.51.1:3306/hcicpm?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull
spring.datasource.username = root
spring.datasource.password = 123456

# redis配置
spring.redis.host = 192.169.50.1
spring.redis.port = 6380

# 修改配置文件的位置
logback.file.log.home = /opt/logs/open-api/open-api-springboot-ccodsz.log

# xxl-job
xxl.job.executor.appname = xxl-job-executor-zhw
xxl.job.executor.ip = 10.137.87.168
xxl.job.executor.port = 9999
xxl.job.executor.logpath = /opt/logs/xxl-job/jobhandler-ccodszdx

#elasticsearch jest配置项
spring.elasticsearch.jest.uris = http://192.169.51.1:9200,http://192.169.51.2:9200,http://192.169.51.4:9200,http://192.169.51.30:9200,http://192.169.51.33:9200

# pingan platform 相关配置 录音上传地址 webservice请求地址
pingan.openapi.success.voicePath = /home/voiceUpload_success
pingan.openapi.success.webserviceUrl = http://10.137.87.34:18080/hwrecord/services/recordWs?wsdl

# pingan platform 相关配置
openapi.platform.success.local = XQD-CCODSZDX
openapi.platform.success.convert.path = /home/convert/decoder-common 7
openapi.platform.success.socket.ip = 10.137.87.218
openapi.platform.success.socket.port = 12000
openapi.platform.success.to.openapi.url = http://192.169.51.3:8080/pingan_openAPI/openApi/receiveCheckResult
openapi.platform.success.trans.count = 50

# es查询相关配置
openapi.elasticsearch.url = http://10.0.6.139:9200

# 平台代码是否开启test
openapi.platform.test = false
```



##### 6.2.11 检查录音权限，以及转码命令权限

无



### 7.elk安装与部署









### 