# cat部署及接入

## 服务端部署

CAT服务端部署主要步骤为：

1. 下载源码，构建
2. 初始化SQL & 添加服务端配置及目录权限
3. 部署到tomcat
4. 验证及客户端路由配置

### 1、构建

github地址：https://github.com/dianping/cat

具体介绍可以查看官网文档，下载速度过慢可以使用码云导入，码云地址：https://gitee.com/github_import/cat?_from=gitee_search

源码使用maven工程导入，之后在主目录执行 mvn clean install -DskipTests

将cat-home模块下打包的cat-alpha的war包取出来，改名为cat.war

搞不定依赖的话，也可以直接下载：http://unidal.org/nexus/service/local/repositories/releases/content/com/dianping/cat/cat-home/3.0.0/cat-home-3.0.0.war

### 2、初始化SQL & 服务器配置

源码docker目录下，有部署需要的配置文件

<img src="C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200528170131394.png" alt="image-20200528170131394" style="zoom: 50%;" />

（1）client.xml

CAT客户端初始化时的路由列表，用于上报数据，该文件必须放在客户端应用机器的/data/appdatas/cat目录下，如果客户端本地运行，则windows系统需要在idea运行环境的盘符下的该目录，例如我的为C:\data\appdatas\cat

```xml
<config mode="client">
    <servers>
        <!--可配置多个服务器（cat服务器集群）详见官方文档-->
        <server ip="127.0.0.1" port="2280" http-port="8080"/>
    </servers>
</config>
```

（2）datasources.xml

cat服务端，mysql连接信息，主要修改地址及用户密码

```xml
<data-sources>
    <data-source id="cat">
        <maximum-pool-size>3</maximum-pool-size>
        <connection-timeout>1s</connection-timeout>
        <idle-timeout>10m</idle-timeout>
        <statement-cache-size>1000</statement-cache-size>
        <properties>
            <driver>com.mysql.jdbc.Driver</driver>
            <url><![CDATA[jdbc:mysql://47.115.116.245:13606/cat]]></url>
            <user>dev</user>
            <password>CW9!TlN6hmPI#23Q</password>
            <connectionProperties><![CDATA[useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&socketTimeout=120000]]></connectionProperties>
        </properties>
    </data-source>
</data-sources>
```

（3）服务器目录权限

不管是服务端还是客户端，都需要具有/data/appdatas/cat和/data/applogs/cat的读写权限，其中服务端需要将client.xml和datasources.xml放在/data/appdatas/cat下；客户端需要client.xml

（4）数据库初始化

需要在对应mysql实例新建一个名称为cat的库，运行 script/CatApplication.sql 的sql文件

### 3、部署到tomcat

搞定tomcat环境后，启动tomcat，放在webapps目录下即可

注：windows较坑的一点是，服务器部署在tomcat，则服务端需要的目录，是tomcat所在的盘符，如我放在D盘，则服务器配置需要放在D:\data\appdatas\cat，而idea启动客户端，我的idae运行环境在C盘，则客户端配置需要放在C:\data\appdatas\cat下

### 4、路由配置

服务端启动后，可通过 http://${服务器ip}:${tomcat端口}/cat/s/config?op=routerConfigUpdate 打开cat的服务器配置，默认账号密码都为admin，若该页面成功打开，则说明服务器启动成功。

修改客户端路由配置，配置详解可看官网：[https://github.com/dianping/cat/wiki/global#%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%B7%AF%E7%94%B1](https://github.com/dianping/cat/wiki/global#客户端路由)

```xml
<router-config backup-server="127.0.0.1" backup-server-port="2280">
   <default-server id="127.0.0.1" weight="1.0" port="2280" enable="true"/>
   <network-policy id="default" title="默认" block="false" server-group="default_group">
   </network-policy>
   <server-group id="default_group" title="default-group">
      <group-server id="127.0.0.1"/>
   </server-group>
   <domain id="cat">
      <group id="default">
         <server id="127.0.0.1" port="2280" weight="1.0"/>
      </group>
   </domain>
</router-config>
```

此处为本地的配置地址，部署的话修改ip即可，端口不允许修改！

## 客户端接入

ygp-utils项目下，cat-combination模块，提供接入cat实现，通过拦截器等实现cat对日志、DAO、Dubbo、URL等监控

### 包依赖

```
<dependency>
    <groupId>com.ygp</groupId>
    <artifactId>cat-combination</artifactId>
    <version>${ygp.cat.combination.version}</version>
</dependency>
```

目前可用版本：1.0.0-SNAPSHOT

### Dubbo

com.ygp.cat.dubbo包提供dubbo接入cat支持 
（1）com.ygp.cat.dubbo.AppNameAppendFilter 
消费者记录AppName 
使用：导入pom包即可
（2）com.ygp.cat.dubbo.CatClientFilter 记录rpc接口方法签名和入参 
使用：默认不使用，需要在dubbo-config配置该filter 
（3）com.ygp.cat.dubbo.CatTransaction 
rpc接口记录Cross事件、Remote事件，异常时记录Event，并记录入参（整合自CatClientFilter） 
使用：导入pom包即可，默认接入，通过注解形式注入到Dubbo的Filter链 
（4）com.ygp.cat.dubbo.DubboCat CatTransaction的开关，可通过该类静态方法设置

### 日志

#### log4j2

com.ygp.cat.log.Log4j2Appender 通过log4j2插件实现日志接入cat 
使用：需要在日志配置中，添加该Appender 
例子：
```xml
<CatAppender name="CatAppender"/>
<Root level="debug">
      <AppenderRef ref="Console"/>
      <AppenderRef ref="CatAppender"/>
    </Root>
```

#### log4j

Cat提供log4j的接入实现：com.dianping.cat.log4j.CatAppender

### DAO

#### mybatis

com.ygp.cat.mybatis.CatSelectInterceptor 
查询语句埋点，通过mybatis拦截器实现，记录DAO事件 
使用：需要配置在mybatis的拦截器中 
例子：

```java
  @Configuration
  public class MybatisPlusConfig {
    @Bean
    @Order(1)
    public CatSelectInterceptor dataFilterInterceptor() {
      return new CatSelectInterceptor();
    }
  }
```

\```

\```

### URL

com.ygp.cat.url.CatFilterConfigure 配置cat对URL的监控 
实现为com.dianping.cat.servlet.CatFilter，通过servlet的拦截器实现 
使用：引入pom即可

## 使用方法

## 原理及架构

todo

## 告警配置

通过cat监控各种事件的数量统计，配置告警规则，当触发告警时，可由CAT发送HTTP请求，由该接口转发到企业微信等

官方文档：https://github.com/dianping/cat/wiki/alarm

#### 应用配置

1. 项目名称
2. 邮件和号码会作为告警名单，支持多个，以小写逗号（,）隔开
   1. 邮件：支持Email、weixin告警，使用企业微信告警，可直接配置为人名
   2. 电话：支持sms告警

![image-20200529174151185](C:\Users\EDZ\AppData\Roaming\Typora\typora-user-images\image-20200529174151185.png)

#### 告警规则配置

- Transaction/Event：统计事务和事件的总数/失败数
- 异常告警：配置特定Exception的告警，可针对重要业务，记录业务异常
- 心跳告警：可配置应用心跳相关的告警，例如GC次数、负载、磁盘空间等的监控

#### 告警策略

全局配置，各告警事件对应的error/warn类型的告警发送配置、间隔时间，可指定应用Group

```xml
<?xml version="1.0" encoding="utf-8"?>
<alert-policy>
   <type id="Transaction">
      <group id="default">
         <level id="warning" send="weixin" suspendMinute="5"/>
         <level id="error" send="weixin" suspendMinute="5"/>
      </group>
       <group id="指定应用group">
         <level id="warning" send="weixin,sms,可多种send逗号隔开" suspendMinute="5"/>
         <level id="error" send="weixin" suspendMinute="5"/>
      </group>
   </type>
   <type id="Event">
      <group id="default">
         <level id="warning" send="weixin" suspendMinute="5"/>
         <level id="error" send="weixin" suspendMinute="1"/>
      </group>
   </type>
   <type id="Exception">
      <group id="default">
         <level id="warning" send="weixin" suspendMinute="5"/>
         <level id="error" send="weixin" suspendMinute="1"/>
      </group>
   </type>
</alert-policy>

```

#### 默认告警人

配置各告警类型的默认告警人，只要符合类型，都会加上该默认告警人

注：weixin类型的告警人标签为<weixin>

```xml
<alert-config>
   <receiver id="Transaction" enable="true">
      <email>testUser1@test.com</email>
      <phone>12345678901</phone>
      <weixin>我是微信</weixin>
   </receiver>
</alert-config>
```

#### 告警服务端

配置监听告警的服务端，支持三种类型，weixin/sms/email，可配置多种类型，在告警策略中配置send

- url：监听接口url
- 入参：入参及对应的取值，取值用${xx}取
- 成功返回值：监听接口成功接收返回值，例如200，只需要返回值中包含字符串“200”即可

```xml
<?xml version="1.0" encoding="utf-8"?>
<sender-config>
   <sender id="weixin" url="http://192.168.0.122:8083/alarmtest/send" type="post" successCode="200" batchSend="false">
      <par id="domain=${domain}"/>
      <par id="email=${receiver}"/>
      <par id="title=${title}"/>
      <par id="content=${content}"/>
      <par id="type=${type}"/>
   </sender>
</sender-config>
```

#### 服务端角色配置

需要将cat集群的其中一台服务器，设置为告警及发送角色，修改后需要重启服务器

```xml
<?xml version="1.0" encoding="utf-8"?>
<server-config>
   <server id="本机ip">
      <properties>
         <property name="job-machine" value="true"/>
         <property name="send-machine" value="true"/>
         <property name="alarm-machine" value="true"/>
         <!-- <property name="remote-servers" value="监听服务器ip:监听端口"/> -->
      </properties>
   </server>
</server-config>

```

注：官方文档中说明， remote-servers为定义HTTP服务列表，远程监听端同步更新服务端信息即取此值，我的理解为该配置只是监听服务器需要更新CAT服务器值才需要