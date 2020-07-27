# Actor discovery (actor 发现)

## Obtaining Actor references (获取actor的引用)

获取Actor references 有两种方式: 使用创建actor和使用Receptionist发现actor

## Receptionist (接待员)

当一个actor需要被另一个actor发现，但是无法在接收到的message中加入对它的引用时，可以使用Receptionist
它支持本地(local)和集群(cluster)
可以在本地Receptionist实例中注册具体的actors, 这些actor可以在每个节点上发现，Receptionist的API也是基于actor消息
在集群的情况下，actor引用的注册表会自动分发到所有的其他节点。
可以使用注册时使用的key来查找，查询返回是一个Listing,包含了该key注册的actor引用的Set,多个actor可以被注册到同一个key

注册表是动态的。在系统的生命周期中可以注册新的角色。当已注册的角色被停止、手动取消注册或其所在的节点从群集中移除时，条目将被删除。为了方便这种动态方面，您还可以使用Receptionist.Subscribe消息订阅变化。它将向订阅者发送Listing消息，先是在订阅时发送一组条目，然后每当某个键的条目发生变化时，就会发送。

```context.system.receptionist ! Receptionist.Register(PingServiceKey, context.self)```

`context.system.receptionist ! Receptionist.Subscribe(PingService.PingServiceKey, context.self)`

```
Behaviors.receiveMessagePartial[Receptionist.Listing] {
          case PingService.PingServiceKey.Listing(listings) =>
            listings.foreach(ps => context.spawnAnonymous(Pinger(ps)))
            Behaviors.same
        }
```

`context.system.receptionist ! Receptionist.Find(PingService.PingServiceKey, listingResponseAdapter)`

Receptionist.Deregister 命令用于取消注册，将删除关联并通知所有的订阅者

`context.system.receptionist ! Receptionist.Deregister(PingService.PingServiceKey, context.self)`

## Cluster Receptionist

Receptionist 也可以使用在Cluster中，一个注册到Receptionist的actor会出现在集群其他节点的Receptionist中

Receptionist 的状态是通过分布式数据传播的，每个节点能范围到每个ServiceKey的同一组actor

集群Receptionist的订阅和查找只会列出节点和访问的注册actor, 通过 Listing.allServiceInstances 可以获的完成的actor列表，包括不可访问的actor
所有发送到另一个节点上的actor的消息和从另一个节点上的actor返回的消息必须是可序列化的




