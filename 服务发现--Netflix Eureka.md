
# 目录

### [Service Discovery: Eureka Clients](#1.1)

### [Service Discovery: Eureka Server](#1.2)


# <span id="1">Spring Cloud Netflix

## <span id="1.1">Eureka Clients
[原文链接](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#_service_discovery_eureka_clients)

### 如何引入Eureka Client

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
### 注册到Eurake

当一个client注册到Eureka，它提供自己的meta-data，例如host,port,health indicator URL,home page等。Eureka接受心跳信息从属于一个服务的每个实例。如果心跳在一个配置的时间内失败，实例将从注册中心移除。

```
@EnableEurekaClient
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```
application.yml

```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```
* Authenticating with the Eureka Server

在```eureka.client.serviceUrl.defaultZone``` URLs中加上认证信息，如```http://user:password@localhost:8761/eureka```。
* Why is it so Slow to Register a Service?

一个实例涉及和注册中心的周期性的心跳，默认周期为30s。一个服务不被客户端发现直到实例，服务端和客户端都在它们本地缓存有了相同的metadata。你可以用```eureka.instance.leaseRenewalIntervalInSeconds```来改变周期，这会加速client和其他server的连接进程。在生产环境最好遵守默认配置，因为在server有一些关于续约周期的内部计算。

## <span id="1.2">Eureka Server
[原文链接](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-eureka-server)

### 如何引入Eureka Server

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```
### 如何运行一个Eureka Server

```
@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }
}
```
Eureka Server有一个UI主页来查看注册的服务信息，```/eureka/```。

### Standalone Mode
在单机模式下，更喜欢关闭client端的行为，如`registerWithEureka`，`fetchRegistry`，所以它不会试图去到达它的peers。

*application.yml (Standalone Eureka Server)*

```
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
注意`serverUrl`指向本地实例的host。

### Peer Awareness
Eureka可以变得高可用通过运行多个实例并让它们相互注册。事实上，这是默认的行为，所以我们只需要给peer添加一个有效的`serviceUrl`。

*application.yml (Two Peer Aware Eureka Servers)*

```
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2/eureka/

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1/eureka/
```
你可以添加多个peers到一个系统，只要它们互相至少有一边连接，它们将互相同步注册信息。如果peers存在物理分区，该系统原则上可能存在裂脑问题。
### Prefer IP Address
通常，Eureka更喜欢暴露它的IP地址而不是它的hostname，设置`eureka.instance.preferIpAddress`为`true`,当注册时，它将使用它的IP地址而不是hostname。



