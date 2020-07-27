# Routers (路由)

## Introduction

在某种场景下，将相同的消息类型的actor放在一组行为体下是很有用的，这样消息可以并行处理
一个actor一次只能处理一个消息

router 路由器本身是一个behavior， 它被spawned成一个正在运行的actor.
然后将任何一个发送到它的消息转发给路由集中的一个最终接收的actor

Akka Typed中包含两种路由器: 池路由器和组路由器


## Pool Router (池路由器)

池路由器是routee Behavior创建的，并以改行为生成一些子的behavior

如果一个子行为被停止，池路由器就会把它从它的路由集合中删除。当最后一个子代停止时，路由器本身也会停止。为了使一个有弹性的路由器能够处理故障，必须对routetee Behavior进行监督。

Configuring Dispatchers

路由器本身是作为一个actor被生成的，所以它所使用的调度器可以在生成时进行配置

```
// make sure workers use the default blocking IO dispatcher
val blockingPool = pool.withRouteeProps(routeeProps = DispatcherSelector.blocking())
// spawn head router using the same executor as the parent
val blockingRouter = ctx.spawn(blockingPool, "blocking-pool", DispatcherSelector.sameAsParent())
```

## Group Router (组路由器)

组路由器是用ServiceKey创建的，并使用Receptionist来发现该key的可用actor，
并将消息路由到当前已知的key的注册actor之一。

由于使用了接待员，这意味着群组路由器是开箱即用的群组感知。路由器向集群中任何可到达的节点上的注册actor发送消息。
如果没有可到达的actor存在，路由器将回退并将消息路由给标记为不可到达的节点上的actor。

使用了接收者也就意味着路由集最终是一致的，当群组路由器启动时，它所知道的路由集立即是空的，
直到它看到了接收者的列表，它就会把收到的消息存储起来，一旦收到接收者的列表就会转发。

当路由器收到来自接待员的列表，并且注册的演员集为空时，路由器将放弃他们（将他们发布到事件流中，称为akka.actor.Dropped）。


## Routing strategies (路由策略)

有三种不同的策略用于选择消息转发到哪个路由，可以在生成消息之前从路由器中选择。

Round Robin

在路由集上旋转，确保如果有n个路由，那么对于通过路由器发送的n条消息，每个行为者被转发一条消息。

只要路由集保持相对稳定，round robin就能提供公平的路由，每个可用的路由者都能得到相同数量的消息，但如果路由集变化很大，则可能不公平。

这是池路由器的默认值，因为希望路由池保持不变。

这个策略可以使用一个可选的参数preferLocalRoutees。如果 preferLocalRoutees 为真且本地路由确实存在，路由器将只使用位于本地角色系统中的路由。这个参数的默认值是false。

Random

当消息通过路由器发送时，随机选择一个路由。

这是群组路由器的默认值，因为随着节点加入和离开群组，路由组会发生变化。

该策略可以使用一个可选的参数preferLocalRoutees。如果 preferLocalRoutees 为真且本地路由确实存在，路由器将只使用位于本地角色系统中的路由。该参数的默认值为false。


Consistent Hashing

使用一致散列来根据发送的消息选择一个routee

必须定义路由器的hashMapping来将收到的消息映射到它们的一致哈希键上。这使得这个决定对发送者来说是透明的。

一致哈希使得具有相同哈希值的报文被映射到相同的route，只要route的集合保持不变。当路由集发生变化时，一致散列试图确保但不保证具有相同散列值的消息被路由到相同的路由。

## Routers and performance (路由性能)

如果路由共享资源，资源将决定增加actor的数量是否会带来更高的吞吐量或更快。
例如，如果routees是CPU绑定的角色，那么创建更多的routees并不会带来更好的性能，因为有更多的线程来执行这些角色。

由于路由器本身就是一个actor，并且有一个邮箱，这就意味着消息会按顺序被路由到路由，在那里它可以被并行处理（取决于调度器的可用线程）。
在高吞吐量的使用情况下，顺序路由可能会成为一个瓶颈。Akka Typed并没有为此提供一个优化的工具。


