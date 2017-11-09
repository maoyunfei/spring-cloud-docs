# Client Side Load Balancer: Ribbon
[原文链接](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-ribbon)

Ribbon是一个客户端负载均衡器，它可以让您对HTTP和TCP客户端的行为有很大的控制权。 Feign已经使用Ribbon，所以如果您使用的是@FeignClient，那么这个部分也适用。

Ribbon中一个重要的概念是named client。Spring Cloud使用RibbonClientConfiguration根据需要为每个named client创建一个新的集合作为ApplicationContext，这包含（除其他外）ILoadBalancer，RestClient和ServerListFilter。

## 1. 如何引入Ribbon
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

## 2. 自定义Ribbon Client

你可以使用`<client>.ribbon.*`属性来配置ribbon client。

Spring Cloud还允许你通过使用`@RibbonClient`声明其他配置（在`RibbonClientConfiguration`上）来完全控制客户端。例：

```
@Configuration
@RibbonClient(name = "foo", configuration = FooConfiguration.class)
public class TestConfiguration {
}
```
在这种情况下，ribbon client由已经在`RibbonClientConfiguration`中的组件和`FooConfiguration`中的任何组件（后者通常会覆盖前者）组成。(备注：使用`RibbonClientConfiguration`中的Bean和自定义的`FooConfiguration`中的Bean来配置ribbon client, `FooConfiguration`中的Bean会覆盖`RibbonClientConfiguration`中的Bean)

**注意：** 上面的`FooConfiguration`必须用`@Configuration`，但是注意它不能在应用上下文被`@ComponentScan`扫描到，否则它将被所有`@RibbonClient`所共享。如果你使用`@ComponentScan`或者`@SpringBootApplication`,你需要避免它被包括在内(例如：把它放在一个单独的，不重叠的包或者在`@ComponentScan`中明确指定要扫描的包)。(备注：我是在`src/main/java`下新建一个package,将自定义的RibbonConfiguration配置Bean放在这个包下)

### Spring Cloud Netflix默认给ribbon提供以下的bean(`BeanType` beanName: `ClassName`):
* `IClientConfig` ribbonClientConfig: `DefaultClientConfigImpl`
* `IRule` ribbonRule: `ZoneAvoidanceRule`
* `IPing` ribbonPing: `NoOpPing`
* `ServerList<Server>` ribbonServerList: `ConfigurationBasedServerList`
* `ServerListFilter<Server>` ribbonServerListFilter: `ZonePreferenceServerListFilter`
* `ILoadBalancer` ribbonLoadBalancer: `ZoneAwareLoadBalancer`
* `ServerListUpdater` ribbonServerListUpdater: `PollingServerListUpdater`

创建这些类型的bean并将其放置在`@RibbonClient`配置Bean（例如上面的`FooConfiguration`）中，可以覆盖所描述的每个bean。例：

```
@Configuration
public class FooConfiguration {
    @Bean
    public IPing ribbonPing(IClientConfig config) {
        return new PingUrl();
    }
}
```
这将用`PingUrl`代替`NoOpPing`。

## 3. 使用properties来自定义Ribbon Client
Spring Cloud Netflix现在支持使用properties来定制Ribbon client，以便与[Ribbon文档](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers#components-of-load-balancer)兼容。

这使你可以在不同的环境启动时更改行为。

支持的属性如下所列，并应以`<clientName>.ribbon`为前缀：

* `NFLoadBalancerClassName`: should implement `ILoadBalancer`
* `NFLoadBalancerRuleClassName`: should implement `IRule`
* `NFLoadBalancerPingClassName`: should implement `IPing`
* `NIWSServerListClassName`: should implement `ServerList`
* `NIWSServerListFilterClassName`: should implement `ServerListFilter`

**提示：**在这些属性中定义的类优先于使用`@RibbonClient(configuration=MyRibbonConfig.class)`定义的bean和Spring Cloud Netflix提供的默认类。

要为一个名为`users`的服务设置`IRule`，可以如下设置：

```
users:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```
## 4. Ribbon和Eureka一起使用
当Eureka和Ribbon一起使用(例如，二者都在classpath), `ribbonServerList`被`DiscoveryEnabledNIWSServerList`的一个扩展覆盖了，该扩展的server list来自于Eureka。同时用`NIWSDiscoveryPing`替代`IPing`,通过Eureka来判断服务状态是否为UP。默认安装的ServerList是一个DomainExtractingServerList，这样做的目的是在不使用AWS AMI metadata(这是Netflix所依赖的)的情况下为负载均衡器提供物理metadata。默认情况下，server list将使用实例metadata中提供的“zone”信息构建（所以在远程客户端上设置`eureka.instance.metadataMap.zone`）,如果没有设置zone，可以使用服务器hostname的域名作为zone的代理（如果设置了标志`approximateZoneFromHostname`）。一旦zone信息可用，就可以在`ServerListFilter`中使用。默认情况下，它将用于定位与client位于同一个zone的server，因为默认值是`ZonePreferenceServerListFilter`。默认地client的zone的确定方式与远程实例相同，即通过`eureka.instance.metadataMap.zone`。

**提示：**如果没有设置zone数据，则根据client配置（而不是实例配置）进行猜测。我们把`eureka.client.availabilityZones`(这是一个从region名称到zone列表的map)，并取出实例所在region的第一个zone（即`eureka.client.region`，默认为“us-east-1“，为了与本地Netflix的兼容性）。

## 5. Ribbon不和Eureka一起使用
Eureka是一个远程服务发现的一个简便实现，所以你不需要在client端硬编码url，但是如果你不喜欢使用eureka，Ribbon和Feign仍然很合适。假设你已经为“stores”声明了@RibbonClient, 并且没有使用eureka。Ribbon client默认使用一个配置的server list,你可以像这样提供配置：

**application.yml**

```
stores:
  ribbon:
    listOfServers: example.com,google.com
```
## 6. 在Ribbon中禁用Eureka
设置属性`ribbon.eureka.enabled = false`将明确禁止在Ribbon中使用Eureka。

**application.yml**

```
ribbon:
  eureka:
   enabled: false
```
## 7. 直接使用Ribbon API
你可以直接使用`LoadBalancerClient`，例如：

```
public class MyClass {
    @Autowired
    private LoadBalancerClient loadBalancer;

    public void doStuff() {
        ServiceInstance instance = loadBalancer.choose("stores");
        URI storesUri = URI.create(String.format("http://%s:%s", instance.getHost(), instance.getPort()));
        // ... do something with the URI
    }
}
```
## 8. Ribbon的缓存配置
每个named client的Ribbon都有一个Spring Cloud维护的对应子应用程序上下文,这个应用程序的上下文是当对named client第一次请求时懒加载的。可以将此延迟加载行为更改为在启动时立即加载这些子应用程序上下文，通过指定Ribbon client的名称来配置。

**application.yml**

```
ribbon:
  eager-load:
    enabled: true
    clients: client1, client2, client3
```

