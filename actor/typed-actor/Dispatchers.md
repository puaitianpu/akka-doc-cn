# Dispatchers (调度器)

## Introduction

Akka MessageDispatcher是让Akka Actors "tick "的东西，
可以说它是机器的引擎。所有的MessageDispatcher实现也是一个ExecutionContext，这意味着它们可以被用来执行任意代码，例如Future。

## Default dispatcher

每一个 ActorSystem 系统都会有一个默认的调度器，在没有为Actor配置其他任何东西的情况下，会使用这个调度器
默认调度器可以配置，默认是一个Dispatcher，通过`akka.actor.default-dispatcher.executor`配置。
如果没有选择执行者，则会选择一个 "fork-join-executor"，在大多数情况下，这个执行者的性能非常好。

## Internal dispatcher (内部调度器)

为了保护由各种Akka模块产生的内部Actor，默认使用一个单独的内部调度器。
内部调度器可以通过设置akka.actor.internal-dispatcher进行细化调整，也可以通过将akka.actor.internal-dispatcher设为别名，用另一个调度器代替。


## Looking up a Dispatcher (查看调度器)

Dispatchers实现了ExecutionContext接口，因此可以调用 lookup 并返回Future

```
// for use with Futures, Scheduler, etc.
import akka.actor.typed.DispatcherSelector
implicit val executionContext = context.system.dispatchers.lookup(DispatcherSelector.fromConfig("my-dispatcher"))
```


## Selecting a dispatcher (选择调度器)

在没有指定自定义调度器的情况下，所有生成的演员都会使用一个默认的调度器。
这适用于所有不阻塞的actor。actor中的阻塞需要仔细管理。

要选择一个调度器，请使用`DispatcherSelector`创建一个 `Props` 实例来生成actor

```
import akka.actor.typed.DispatcherSelector

context.spawn(yourBehavior, "DefaultDispatcher")
context.spawn(yourBehavior, "ExplicitDefaultDispatcher", DispatcherSelector.default())
context.spawn(yourBehavior, "BlockingDispatcher", DispatcherSelector.blocking())
context.spawn(yourBehavior, "ParentDispatcher", DispatcherSelector.sameAsParent())
context.spawn(yourBehavior, "DispatcherFromConfig", DispatcherSelector.fromConfig("your-dispatcher"))
```

`DispatcherSelector` 有一些实用的方法

*  DispatcherSelector.default用来查找默认的调度器
*  DispatcherSelector.blocking可用于执行阻塞的actor, 例如不支持Future的传统数据库API
*  DispatcherSelector.sameAsParent实用与父actor相同的调度器


最后一个例子展示了如何从配置中加载一个自定义的调度器，并依赖于你application.conf中的内容

```
your-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 32
  }
  throughput = 1
}
```

## Types of dispatchers (调度器的类型)

有两种不同类型的调度器

Dispatcher

这是一个基于事件的调度器，将一组Actor绑定到线程池。如果没有指定其他的调度器，则使用默认的调度器。

*  可共享性, 没有限制
*  邮箱，任意，每个actor创建一个
*  使用案例, 默认调度器， Bulkheading(分流)
*  驱动：java.util.concurrent.ExecutorService， 使用 "executor "指定使用 "fork-join-executor"、"thread-pool-executor "或完全限定的类名的akka.dispatcher.ExecutorServiceConfigurator实现。

PinnedDispatcher

这个调度器为每个使用它的actor奉献一个唯一的线程；即每个actor将有自己的线程池，池中只有一个线程。

可共享性。无
信箱。任意，每个演员创建一个
使用案例。隔离墙


*  不可共享
*  邮箱，任意，每个actor创建一个
*  使用案例， Bulkheading(隔离墙)
*  驱动方式, 任何akka.dispatch.ThreadPoolExecutorConfigurator. 默认情况下是一个 "线程池执行器"。
  
这里是一个Fork Join Pool dispatcher的配置示例。

```
my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

## Dispatcher aliases (调度器别名)

当查询调度器时，如果给定的设置包含一个字符串而不是调度器配置块，则查询会将其视为别名，并跟随该字符串到调度器配置的备用位置。如果调度器配置既通过别名引用，又通过绝对路径引用，那么将只使用一个调度器，并在两个id之间共享。

配置 internal-dispatcher 为 default-dispatcher 的别名。

```
akka.actor.internal-dispatcher = akka.actor.default-dispatcher

```

## Blocking Needs Careful Management (阻塞管理)

在某些情况下，做阻塞操作是不可避免的，即让一个线程在不确定的时间内进入睡眠状态，等待外部事件的发生。例如遗留的RDBMS驱动或消息API，其根本原因通常是（网络）I/O发生在掩盖之下。

Problem: 屏蔽默认调度器
像这样简单地将阻塞调用添加到你的actor消息处理中是有问题的
```
import akka.actor.typed.Behavior
import akka.actor.typed.scaladsl.Behaviors

object BlockingActor {
  def apply(): Behavior[Int] =
    Behaviors.receiveMessage { i =>
      // DO NOT DO THIS HERE: this is an example of incorrect code,
      // better alternatives are described further on.

      //block for 5 seconds, representing blocking I/O, etc
      Thread.sleep(5000)
      println(s"Blocking operation finished: $i")
      Behaviors.same
    }
}
```
无需任何进一步的配置，默认的调度器就会将这个角色和其他角色一起运行。当所有角色的消息处理都是非阻塞的时候，这种方式非常高效。然而，当所有可用的线程都被阻塞时，同一调度器上的所有角色都会对线程感到饥渴，无法处理传入的消息


Note
如果可能的话，也应该避免阻塞式的API。尽量寻找或构建反应式API，这样可以最大限度地减少阻塞，或者转移到专门的调度器上。

在 `application.conf` 专门用于阻塞行为的调度器应该配置如下

```
my-blocking-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 16
  }
  throughput = 1
}
```

基于线程池执行器的调度器可以让我们限制它所承载的线程数量，这样我们就可以获得对系统可能使用的最大阻塞线程数量的严格控制。

具体的大小应该根据你期望在这个调度器上运行的工作负载进行微调。

每当必须进行阻塞时，就使用上面配置的调度器而不是默认的调度器。


Available solutions to blocking operations
适用解决"阻塞问题"有以下建议:

*  在"Future" 中进行阻塞调用，确保在任何时间点上对此类调用的数量有一个上限，（提交无限制数量的此类任务将耗尽你的内存或线程限制）
*  在Future内做阻塞调用，提供一个线程池，其线程数量的上限适合应用程序运行的硬件
*  将一个线程专门用于管理一组阻塞资源（例如驱动多个通道的NIO选择器），并在事件发生时作为执行者消息进行调度
*  在由路由器管理的一个actor（或一组actor）内进行阻塞调用，确保配置一个专门用于此目的的线程池或足够大的线程池。


## More dispatcher configuration examples (更多的调度器配置例子)

Fixed pool size
配置具有固定线程池大小的调度器，例如用于执行阻塞IO的actor

```
blocking-io-dispatcher {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 32
  }
  throughput = 1
}
```

Cores
另一个根据核心数使用线程池的例子（例如对于CPU绑定的任务）

```
my-thread-pool-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "thread-pool-executor"
  # Configuration for the thread pool
  thread-pool-executor {
    # minimum number of threads to cap factor-based core number to
    core-pool-size-min = 2
    # No of core threads ... ceil(available processors * factor)
    core-pool-size-factor = 2.0
    # maximum number of threads to cap factor-based number to
    core-pool-size-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

Pinned
每个被配置为使用Pinned Dispatcher的actor都有一个单独的线程

配置一个PinnedDispatcher

```
my-pinned-dispatcher {
  executor = "thread-pool-executor"
  type = PinnedDispatcher
}
```

请注意，按照上述my-thread-pool-dispatcher例子的线程池-执行者配置是不适用的。这是因为在使用PinnedDispatcher时，每个actor都会有自己的线程池，而这个池只有一个线程。

需要注意的是，并不能保证一直使用同一个线程，因为核心池超时用于PinnedDispatcher，以在演员闲置的情况下降低资源使用率。如果要一直使用同一个线程，你需要在PinnedDispatcher的配置中添加thread-pool-executor.allow-core-timeout=off。


Thread shutdown timeout

fork-join-executor和thread-pool-executor都可以在不用线程时关闭线程。如果希望让线程保持更长时间的活力，可以调整一些超时设置

```
my-dispatcher-with-timeouts {
  type = Dispatcher
  executor = "thread-pool-executor"
  thread-pool-executor {
    fixed-pool-size = 16
    # Keep alive time for threads
    keep-alive-time = 60s
    # Allow core threads to time out
    allow-core-timeout = off
  }
  # How long time the dispatcher will wait for new actors until it shuts down
  shutdown-timeout = 60s
}
```

