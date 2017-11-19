# 1. Hystrix Clients
[原文链接](http://cloud.spring.io/spring-cloud-static/Finchley.M2/#_circuit_breaker_hystrix_clients)

Netflix创建了一个实现了[circuit breaker模式](https://martinfowler.com/bliki/CircuitBreaker.html)的叫做[Hystrix](https://github.com/Netflix/Hystrix)的库。在一个微服务架构中通常有多层的服务调用，如下图。

<img src="http://cloud.spring.io/spring-cloud-static/Finchley.M2/images/HystrixGraph.png" style="zoom:50%" />*Microservice Graph*

一个底层的服务失败可以导致级联的直到用户的失败。在一个由 `metrics.rollingStats.timeInMilliseconds`(默认10秒)定义的默认窗口内，当请求一个指定的服务次数大于`circuitBreaker.requestVolumeThreshold`(默认20)并且失败率大于`circuitBreaker.errorThresholdPercentage`(默认50%)时，断路打开，请求不会发出。在发生错误或者短路时，开发者可以提供fallback。

<img src="http://cloud.spring.io/spring-cloud-static/Finchley.M2/images/HystrixFallback.png" style="zoom:50%" />*Hystrix fallback prevents cascading failures*

断路阻止了级联失败，并且允许高负载或者失败的服务有时间去恢复。fallback可以是另一个Hystrix保护的调用，静态数据或者空值。fallback可能是链式的，导致第一个fallback做的一些业务调用又回退到静态数据。

## 1.1 如何引入Hystrix

```
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
```
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
}
```
`@HystrixCommand`由一个名为“javanica”的Netflix contrib库提供。Spring Cloud自动将包含该注释的Spring bean包装在连接到Hystrix断路器的代理中。断路器计算何时打开和关闭电路，以及在发生故障时该怎么办。

要配置`@HystrixCommand`，您可以使用带有`@HystrixProperty`注释列表的`commandProperties`属性。[这里](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica#configuration)查看细节。[Hystrix properties](https://github.com/Netflix/Hystrix/wiki/Configuration)。

## 1.2 传播安全上下文或者使用Spring Scopes
如果你想传播一些线程本地上下文到@HystrixCommand中，用默认声明是不起作用的，因为它在一个线程池中执行命令。当调用者使用一些配置或者直接在注解中让它去使用一个不同的隔离策略，你可以切换Hystrix去使用同一个线程。例如：

```
@HystrixCommand(fallbackMethod = "stubMyService",
    commandProperties = {
      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
    }
)
...
```
如果使用`@SessionScope`或`@RequestScope`，则同样适用。你将知道何时需要执行此操作，因为一个运行时异常表示无法找到该scope内的上下文。

你也可以设置`hystrix.shareSecurityContext`属性为`true`。这样会自动配置一个Hystrix并发策略插件，它将会把`SecurityContext`从你的主线程传递到Hystrix命令使用的线程。Hystrix不支持注册多个hystrix并发策略，所以可以通过声明你自己的`HystrixConcurrencyStrategy` bean来扩展。Spring cloud将在你的spring上下文中查找并把它封装进它自己的插件。

## 1.3 健康指标
连接断路器的状态也暴露在应用程序的`/health`端点中。

```
{
    "hystrix": {
        "openCircuitBreakers": [
            "StoreIntegration::getStoresByLocationLink"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
```

## 1.4 Hystrix指标流
添加spring-boot-starter-actuator依赖来启用Hystrix指标流。这将暴露管理端点`/hystrix.stream`。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

# 2. Hystrix Dashboard
[原文链接](http://cloud.spring.io/spring-cloud-static/Finchley.M2/#_circuit_breaker_hystrix_dashboard)

Hystrix的主要优点之一就是它收集的关于每个HystrixCommand的指标集合。Hystrix仪表板以高效的方式显示每个断路器的运行状况。

<img src="http://cloud.spring.io/spring-cloud-static/Finchley.M2/images/Hystrix.png" style="zoom:30%" />*Hystrix Dashboard*

# 3. Hystrix超时和Ribbon Client
[原文链接](http://cloud.spring.io/spring-cloud-static/Finchley.M2/#_hystrix_timeouts_and_ribbon_clients)

当使用Hystrix命令包装Ribbon client，你需要确保配置的Hystrix超时时间大于配置的Ribbon超时时间，包括任何潜在的重试。例如，如果你的ribbon连接超时是1秒，ribbon client可能重试3次，然后Hystrix超时应该略大于3秒。

# 3.1 如何引入Hystrix Dashboard

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-netflix-dashboard</artifactId>
</dependency>
```
运行Hystrix Dashboard使用`@EnableHystrixDashboard`注释Spring Boot主类。然后访问`/hystrix`并将仪表板指向Hystrix客户端应用程序中的单个实例的`/hystrix.stream`端点。

**提示：** 连接到使用HTTPS的`/hystrix.stream`端点时，服务器使用的证书必须由JVM信任。如果证书不可信，你必须将证书导入到JVM中，以便Hystrix仪表板能成功连接到流终端。

## 3.2 Turbine
从单个实例来看，Hystrix数据在整个系统的健康状况方面并不是很有用。[Turbine](https://github.com/Netflix/Turbine)是一个应用程序，它将所有相关的`/hystrix.stream`端点汇总到一个用于Hystrix仪表板的组合`/turbine.stream`中。单个实例通过Eureka找到。运行Turbine与使用@`EnableTurbine`注释注释主类一样简单(例如，在classpath引入spring-cloud-starter-netflix-turbine)。来自[Turbine 1 wiki](https://github.com/Netflix/Turbine/wiki/Configuration-(1.x))的文档配置属性都适用。唯一的区别是`turbine.instanceUrlSuffix`不需要预先添加端口，因为这是自动处理的，除非`turbine.instanceInsertPort=false`。

**提示：** 默认情况下，Turbine在注册实例上查找`/hystrix.stream`端点是通过在Eureka中查找其`homePageUrl`条目，然后将`/hystrix.stream`附加到上面。这意味着如果`spring-boot-actuator`在自己的端口上运行（这是默认的），对`/hystrix.stream`的调用将失败。要使Turbine在正确的端口找到Hystrix流，你需要将`management.port`添加到实例的metadata：

```
eureka:
  instance:
    metadata-map:
      management.port: ${management.port:8081}
```
配置`turbine.appConfig`是Turbine用于查找实例的Eureka中注册的serviceId的列表。Turbine stream然后在Hystrix仪表板中使用一个类似如下的url：`http://my.turbine.sever:8080/turbine.stream?cluster=<CLUSTERNAME>`(cluster参数可以被省略，如果名称是“default”)。`cluster`参数必须与`turbine.aggregator.clusterConfig`中的条目匹配。从Eureka返回的值是大写，因此，如果有一个名为“customers”的应用程序在Eureka注册，我们预计这个例子将起作用：

```
turbine:
  aggregator:
    clusterConfig: CUSTOMERS
  appConfig: customers
```
`clusterName`可以通过`turb.clusterNameExpression`中的SPEL表达式来定制，指定`InstanceInfo`的一个实例。默认值是`appName`，这意味着Eureka serviceId最终作为集群key(即customers的InstanceInfo具有“CUSTOMERS”的`appName`)。另一个示例是`turb.clusterNameExpression=aSGName`，它将从AWS ASG名称获取集群名称。另一个例子：

```
turbine:
  aggregator:
    clusterConfig: SYSTEM,USER
  appConfig: customers,stores,ui,admin
  clusterNameExpression: metadata['cluster']
```
在这种情况下，来自4个服务的集群名称将从其metadata映射中提取出来，预期包含“SYSTEM”和“USER”的值。

要为所有应用程序使用“default”集群，你需要一个字符串文字表达式(使用单引号，如果使用YAML，则使用双引号进行转义):

```
turbine:
  appConfig: customers,stores
  clusterNameExpression: "'default'"
```
Spring Cloud提供了一个`spring-cloud-starter-netflix-turbine`，它拥有运行Turbine服务器所需的所有依赖。只需创建一个Spring Boot应用程序并使用`@EnableTurbine`对其进行注释。

**提示：** 默认情况下，Spring Cloud允许Turbine使用主机和端口来允许每个主机，每个集群使用多个进程。如果你希望Turbine中内置的本机Netflix行为不允许每个主机，每个集群（实例id的key是主机名）都有多个进程，那么请设置属性`turbine.combineHostPort=false`。
## 3.3 Turbine Stream
在某些环境下（例如在PaaS设置中），从所有分布式Hystrix命令中提取指标的传统Turbine模型不起作用。在这种情况下，你可能希望让你的Hystrix命令将度量标准推送到Turbine，Spring Cloud通过消息传递来实现。你需要在客户端上执行的操作是添加依赖关系到你选择的`spring-cloud-netflix-hystrix-stream`和`spring-cloud-starter-stream-*`(有关broker和如何配置客户端凭据的详细信息，请参阅Spring Cloud Stream文档，但它应该为本地broker开箱即用)。

在server端只需创建一个Spring Boot应用程序并使用`@EnableTurbineStream`对其进行注释，默认情况下它将在8989端口(将Hystrix仪表板指向该端口，任何路径)运行。你可以使用`server.port`或`turbine.stream.port`来自定义端口。如果在classpath中也有`spring-boot-starter-web`和`spring-boot-starter-actuator`，那么你可以通过提供一个不同的`management.port`，在单独的端口(默认情况下使用Tomcat)打开Actuator端点。

然后你可以将Hystrix仪表板指向Turbine Stream Server，而不是单独的Hystrix流。如果Turbine Stream在myhost上的8989端口上运行，则将http:// myhost:8989放在Hystrix仪表板的流输入字段中。电路将以它们各自的serviceId为前缀，接着是一个点，然后是电路名称。

Spring Cloud提供了一个`spring-cloud-starter-netflix-turbine-stream`，它拥有运行Turbine Stream server所需的所有依赖关系，只需添加你选择的Stream绑定程序，例如：`spring-cloud-starter-stream-rabbit`。你需要Java 8来运行应用程序，因为它是基于Netty的。
