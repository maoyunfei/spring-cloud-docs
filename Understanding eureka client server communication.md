# Understanding eureka client server communication
[原文链接](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)

## About Instance Statuses

默认的，Eureka client开始状态是 **STARTING**，这为了在实例能够提供服务之前，给它做应用初始化的时间。之后应用可以加入到可提供服务中通过将状态变更为 **UP**。

`ApplicationInfoManager.getInstance().setInstanceStatus(InstanceStatus.UP)`

应用也可以注册健康检查的callback，这可以选择性地将实例状态变为 **DOWN**。

在Neflix中，还有一个 **OUT_ OF_ SERVICE** 状态,表明该实例不可提供服务中。

## Eureka Client Operations

Eureka client首先尝试去和相同zone的Eureka Server连接，如果它不能发现服务端，它将转向其他zone。

### Eureka client通过以下方式和服务端交互

* **Register**

> Eurek Client向Eureka server注册运行实例的信息。注册发生在第一次心跳(在30秒之后)。

* **Renew**

> Eureka client需要通过每30秒发送心跳来“续约”。“续约”信号告诉Eureka server该实例仍然是可用的。如果server在90秒没有收到“续约”，他将从注册列表移除该实例。不去改变“续约”周期是明智的，因为server使用这个信息去判断在client和server之间的通信是否有普遍的问题。(备注：例如网络分区问题)

* **Fetch Registry**

> Eureka client从server获取注册信息并缓存在本地。之后，client使用这个信息去发现其他的服务。注册信息被周期性的更新(每30秒)，通过获取上一个读取周期和当前读取周期之间的增量更新。增量信息在server中保持较长时间（约3分钟），因此增量提取可能会再次返回相同的实例。Eureka client会自动处理重复的信息。

> 获取增量之后，Eureka client和server通过比较server返回的实例数量来比对信息，如果由于某些原因信息不匹配，整个注册信息将重新提取。Eureka server缓存压缩的增量payload、整个注册表，以及每个应用程序相同的未压缩信息。payload支持JSON和XML格式。Eureka client通过jersey apache client获取压缩的JSON格式的信息。

* **Cancel**

> Eureka client在shutdown时给Eureka server发送一个cancel请求。Eureka server将从服务的实例注册信息中移除该实例，这有效得将实例从负载中移除。(备注：通过linux命令 kill -15应用程序将收到shutdown通知，kill -9则收不到通知)

> 当Eureka client shutdown时，应用应该保证去调用以下方法。
```
DiscoveryManager.getInstance().shutdownComponent()
```

* **Time Lag**

> Eureka client的所有操作会花费一定时间反映到Eureka server和其他的clients。这是因为Eureka server上有payload的缓存，它定期刷新以获取新的信息。Eureka client也会定期刷新增量信息。因此，这可能花费长达2分钟的时间去把变更传播到所有的Eureka client。

* **Communication mechanism**

> 默认的，Eureka client使用Jersey,XStream技术和JSON 格式的payload去和Eureka Server交流。你也可以使用你选择的机制来覆盖默认的。
