# Understanding Eureka Peer to Peer Communication
[原文链接](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication)

Eureka client尝试去和相同zone的Eureka Server通信。如果相同zone的server不存在或者通信有问题，client就会转到其他zone的server。

一旦服务器开始接收流量，在服务器上执行的所有操作都将被复制到服务器所知道的所有对等节点。如果某个操作由于某种原因而失败，那么该信息将在服务器之间下一次心跳时核对后复制。

当Eureka server恢复，它尝试从邻居节点获取所有实例的注册信息。如果从一个节点获取信息存在问题，在它放弃之前，它将尝试所有的对等节点。如果Eureka server能够成功获取所有实例信息，则会根据该信息设置应该接收的“续约”阈值。如果任何时候，“续约”低于设置的该阈值百分比(在15分钟内低于85%),Eureka server停止过期实例来保护当前实例的注册信息。

在Neflix,上面的保护称为“自我保护”模式，主要用在一组client和Eureka server之间存在网络分区的情况下的保护。在这种场景下，Eureka server尝试去保护已经拥有的信息。如果发生大规模的故障，在这种情况下，可能会导致client获得已经不存在的实例。client必须确保它们对于返回不存在或不响应的实例的Eureka server具有弹性。在这些情况下，最好的保护是快速超时并尝试其他服务器。

在这种情况下，如果Eureka server无法从邻居节点获取注册表信息，则会等待几分钟（5分钟），以便客户端可以注册其信息。server尽量不提供部分信息给client，而是通过将流量倾斜到仅一组实例并会导致容量问题。

如此处所述，Eureka server使用在Eureka client和server之间使用的相同机制相互通信。

## What happens during network outages between Peers?
### 在peers之间失去网络通信的情况下，下列事情将发生

* peers之间的心跳复制可能会失败，并且Eureka server检测到这种情况然后进入自我保护模式来保护当前状态。
* 注册可能发生在孤立的Eureka server上，有些client可能会反映新的注册信息，而其他client可能不会。(备注：由于孤立的Eureka server无法与其他server共享注册信息)
* 在网络连接恢复到稳定状态后，情况会自动更正。当peers能够正常通信时，注册信息会自动被传输到没有这些信息的Eureka server上。(备注：即网络恢复后，Eureka server之间会自动同步共享注册信息)

最重要的是，在网络中断期间，Eureka server尝试尽可能地具有弹性，但在此期间，client可能会有不同的server视图。(备注：Eureka server存在网络分区时，多个server之间无法同步注册信息，导致每个server上的信息可能不同，所以client可能会看到不同的server视图)




