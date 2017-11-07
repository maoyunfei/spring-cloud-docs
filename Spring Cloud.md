
# 目录
[Spring Cloud Commons](#1)

* [Spring RestTemplate as a Load Balancer Client](#1.1)

* [Retrying Failed Requests](#1.2)

[Spring Cloud Netflix](#2)

* [Service Discovery: Eureka Clients](#2.1)
* [Service Discovery: Eureka Server](#2.2)

# <span id="1">Spring Cloud Commons

## <span id="1.1"> Spring RestTemplate as a Load Balancer Client </span>
通过@LoadBalanced和@Bean修饰可以生成一个具有负载均衡功能的RestTemplate。

```
@Configuration
public class MyConfiguration {
    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

## <span id="1.2">Retrying Failed Requests
RestTemplatede的失败重试,默认是不可用的，如果需要开启，需要设置```spring.cloud.loadbalancer.retry.enabled=true```并且添加Spring Retry依赖。

```
<dependency>
	<groupId>org.springframework.retry</groupId>
	<artifactId>spring-retry</artifactId>
</dependency>
```
具有负载均衡功能的RestTemplate将遵循Ribbon关于重试的配置，如```client.ribbon.MaxAutoRetries```，```client.ribbon.MaxAutoRetriesNextServer```，```client.ribbon.OkToRetryOnAllOperations```。[Ribbon具体的配置](https://github.com/Netflix/ribbon/wiki/Getting-Started#the-properties-file-sample-clientproperties)。

# <span id="2">Spring Cloud Netflix

## <span id="2.1">Service Discovery: Eureka Clients
* 如何引入Eureka Client

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
* 注册到Eurake

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

## <span id="2.2">Service Discovery: Eureka Server

* 如何引入Eureka Server

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```
* 如何运行一个Eureka Server

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







