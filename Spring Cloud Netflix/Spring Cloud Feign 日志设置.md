# Spring Cloud Feign 日志设置

由于`FeignClient`封装了restful请求，我们很难看出发出的请求和收到的响应具体是什么。

为此我们可以做一些设置来打印出相应的日志。

## 日志配置

* 第一步，编写`FeignClient`的configuration类。

```java
@Configuration
public class FeignLogConfiguration {
  @Bean
  Logger.Level feignLoggerLevel() {
    return Logger.Level.BASIC;
  }
}
```
```java
@FeignClient(name = "microservice-provider-user", configuration = FeignLogConfiguration.class)
public interface UserFeignClient {
  @RequestMapping(value = "/{id}", method = RequestMethod.GET)
  public User findById(@PathVariable("id") Long id);
}
```
`Feign`的日志级别枚举如下：
![](https://github.com/maoyunfei/static-sources/blob/master/feign.jpeg?raw=true)

* 第二步，配置`FeignClient`所在包的日志级别为`DEBUG`

```yml
# application.yml
logging:
  level:
    com.cloud.study.feign: DEBUG
```

如果你不是在`application.yml`中配置的日志级别，而是使用`logback-spring.xml`，同理，在`logback-spring.xml`中做相应配置：

```xml
# logback-spring.xml
<logger name="com.cloud.study.feign" level="DEBUG" additivity="false">
    <appender-ref ref="Console"/>
</logger>
```

## 日志输出

当你配置完以上之后，每次`FeignClient`的调用都会打印出详细日志，具体如下：

![](https://github.com/maoyunfei/static-sources/blob/master/feign_2.jpeg?raw=true)


