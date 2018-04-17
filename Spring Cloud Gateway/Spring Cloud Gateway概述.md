# Spring Cloud Gateway概述

Spring Cloud Gateway项目提供了一个构建在Spring生态之上的API网关，包括：Spring 5，Spring Boot 2和Project Reactor。Spring Cloud Gateway旨在提供一种简单而高效的统一API路由管理方式，并为他们提供网关基本功能，例如：安全性，监控/指标和弹性等。

## 源码概览

目前最新的Spring Cloud Gateway版本是2.0.0.M9。

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_1.jpeg?raw=true"  width = "70%" height = "70%" align=center />

下面将按照包的顺序介绍主要类的功能和特点。

### (一) 'actuate' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_2.jpeg?raw=true"  width = "50%" height = "50%" align=center />

* **`GatewayControllerEndpoint`**: 提供查询、管理网关路由、过滤器等信息的API端点。使用了Spring Boot 2.0的`@RestControllerEndpoint`注解。

### (二) 'config' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_3.jpeg?raw=true"  width = "70%" height = "70%" align=center />

Spring Cloud Gateway自动配置相关的类。

四个自动配置类的初始化顺序如下：

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_10.png?raw=true"  width = "70%" height = "70%" align=center />

* **`GatewayClassPathWarningAutoConfiguration`**：Spring Cloud Gateway 2.x 基于 Spring WebFlux 实现。用于检查项目是否正确导入`spring-boot-starter-webflux`依赖，而不是错误导入`spring-boot-starter-web`依赖。
* **`GatewayLoadBalancerClientAutoConfiguration`**：实例化`LoadBalancerClientFilter`。
* **`GatewayRedisAutoConfiguration`**：实例化`RedisRateLimiter`。`RequestRateLimiterGatewayFilterFactory`基于`RedisRateLimiter`实现网关的限流功能。
* **`GatewayAutoConfiguration`**：Spring Cloud Gateway核心配置类，实例化以下组件。
	* NettyConfiguration
	* GlobalFilter
	* FilteringWebHandler
	* GatewayProperties
	* PrefixPathGatewayFilterFactory
	* RoutePredicateFactory
	* RouteDefinitionLocator
	* RouteLocator
	* RoutePredicateHandlerMapping
	* GatewayControllerEndpoint

	组件关系交互如下图 ：
	
![](https://github.com/maoyunfei/static-sources/blob/master/gateway_11.jpeg?raw=true)

### (三) 'discovery' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_4.jpeg?raw=true"  width = "50%" height = "50%" align=center />

* **`DiscoveryClientRouteDefinitionLocator`**：基于服务发现的路由定义。
* **`GatewayDiscoveryClientAutoConfiguration`**：实例化`DiscoveryClientRouteDefinitionLocator`。

### (四) 'event' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_5.jpeg?raw=true"  width = "50%" height = "50%" align=center />

事件消息定义。

### (五) 'filter' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_6.jpeg?raw=true"  width = "50%" height = "50%" align=center />

各种全局过滤器和过滤器工厂类。

**GlobalFilter**会作用到所有的Route上,默认加载顺序如下。

1. NettyWriteResponseFilter
2. WebClientWriteResponseFilter
3. RouteToRequestUrlFilter
4. LoadBalancerClientFilter
5. ForwardRoutingFilter
6. NettyRoutingFilter
7. WebClientHttpRoutingFilter
8. WebsocketRoutingFilter

### (六) 'handler' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_7.jpeg?raw=true"  width = "50%" height = "50%" align=center />

* 内置路由断言工厂：
	* `AfterRoutePredicateFactory`
	* `BeforeRoutePredicateFactory`
	* `BetweenRoutePredicateFactory`
	* `CookieRoutePredicateFactory`
	* `HeaderRoutePredicateFactory`
	* `HostRoutePredicateFactory`
	* `MethodRoutePredicateFactory`
	* `PathRoutePredicateFactory`
	* `QueryRoutePredicateFactory`
	* `RemoteAddrRoutePredicateFactory`
	* `WeightRoutePredicateFactory`
* `FilteringWebHandler`：内置`DefaultGatewayFilterChain`，通过过滤器链处理`ServerWebExchange`。
* `RoutePredicateHandlerMapping`：对请求做路由断言处理，将请求映射到对应的处理器。

Spring Cloud Gateway工作流程如下：

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_12.jpeg?raw=true"  width = "50%" height = "50%" align=center />

### (七) 'route' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_8.jpeg?raw=true"  width = "50%" height = "50%" align=center />

Route、RouteDefinition、RouteLocator、RouteDefinitionLocator、RouteLocatorBuilder等类的定义。

### (八) 'support' package

<img src="https://github.com/maoyunfei/static-sources/blob/master/gateway_9.jpeg?raw=true"  width = "50%" height = "50%" align=center />

一些工具类的定义。

## 内置过滤器功能介绍
接下来两天更新。

## 主要配置属性介绍
接下来两天更新。

## 网关管理端口
接下来两天更新。

