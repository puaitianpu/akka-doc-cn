
# Actor lifecycle (Actor 生命周期)

## Introduction

Actor 是一个有状态的资源，必须明确地启动和停止

需要注意的是，当不再被引用时，actor 不会自动停止。每一个被创建的Actor也必须显式地销毁。停止一个父Actor也会递归地停止这个父Actor下面的所有子Actor。当ActorSystem被关闭时，所有的Actor也会停止

Note
```
ActorSystem是一个重量级的结构，会分配线程。每个逻辑应用都会创建一个，通常每个JVM进程都有一个
```

## Creating Actors

一个actor 可以创建或产生任意数量的子actor, 子actor又可以产生自己的子actor,从而行程一个actor层次结构。
ActorSystem是顶层的根Actor。父actor承载子actor的生命周期，子actor的寿命不会超过父actor

ActorContext 的作用

*  创建和监督子actor
*  如果被监督的actor永久停止，则监视的其他的成员会收到一个`Terminated(otherActor)` 事件
*  日志记录
*  创建消息适配器
*  和其他actor 进行请求-响应通信
*  访问自己的 ActorRef

如果一个 behavior 需要 ActorContext, 使用 spawn 创建子actor, 或者使用 `context.self`
可以使用Behaviors.setup获得

```
object HelloWorldMain {

  final case class SayHello(name: String)

  def apply(): Behavior[SayHello] =
    Behaviors.setup { context =>
      val greeter = context.spawn(HelloWorld(), "greeter")

      Behaviors.receiveMessage { message =>
        val replyTo = context.spawn(HelloWorldBot(max = 3), message.name)
        greeter ! HelloWorld.Greet(message.name, replyTo)
        Behaviors.same
      }
    }

}
```


ActorContext 的线程安全

ActorContext 中大多方法都是线程不安全的, 使用中有以下注意点

*  不能和 `scala.concurrent.Future` 搭配使用
*  不能在多个actor实例之间共享
*  只能在普通的actor消息处理线程中使用

Guardian Actor
顶层的actor, 用户监护actor. 和ActorSystem一起创建，用户创建ActorSystem定义behavior行为

```
val system: ActorSystem[HelloWorldMain.SayHello] =
  ActorSystem(HelloWorldMain(), "hello")
```
使用场景
```
val systemId: String = ConfigSource.default.at("akka.system.name").loadOrThrow[String]
  val selfUuid: String                      = UUID.randomUUID().toString
  implicit val system: ActorSystem[SpawnProtocol.Command] = ActorSystem(SpawnProtocol(), systemId)
```

对于非常简单的应用 `guardian` 可以包含应用逻辑和handle消息
一旦应用程序处理了超过一个关注点， `guardian` 就应该仅仅只是引导应用程序，生成各种子系统作为子系统并监控它们的生命周期
guardian actor 停止时，也将停止ActorSystem

创建子actor
使用 ActorContent 的 spawn 创建

```
object HelloWorldMain {

  final case class SayHello(name: String)

  def apply(): Behavior[SayHello] =
    Behaviors.setup { context =>
      val greeter = context.spawn(HelloWorld(), "greeter")

      Behaviors.receiveMessage { message =>
        val replyTo = context.spawn(HelloWorldBot(max = 3), message.name)
        greeter ! HelloWorld.Greet(message.name, replyTo)
        Behaviors.same
      }
    }

}
```

在创建 actor 使用 `DispatcherSelector` 指定调度程序， 如果不指定，则使用默认调度程序

```
def apply(): Behavior[SayHello] =
  Behaviors.setup { context =>
    val dispatcherPath = "akka.actor.default-blocking-io-dispatcher"

    val props = DispatcherSelector.fromConfig(dispatcherPath)
    val greeter = context.spawn(HelloWorld(), "greeter", props)

    Behaviors.receiveMessage { message =>
      val replyTo = context.spawn(HelloWorldBot(max = 3), message.name)

      greeter ! HelloWorld.Greet(message.name, replyTo)
      Behaviors.same
    }
  }
``` 

SpawnProtocol
一个预定义的消息协议和行为的实现，可以作为ActorSystem的监护角色(guardian actor)
可以与Behaviors.setup结合使用来启动一些初始任务或角色。
可以发送消息到SpawnProtocol.Spawn的方式从外部启动子Actor


## Stopping Actors

一个actor 可以使用 `Behaviors.stopped` 作为下一个 behavior 来停止自己

子actor在处理完当前消息后，可以通过使用父actor的 ActorContext 的 stop 方法来强制定制，
只有子actor可以通过这种方式被停止

父actor停止时，所有的子actor都会被停止

一个actor被停止时，会收到一个PostStop 信号，可以使用该信号去处理一下清理资源的操作
也可以给 `Behaviors.stopped` 指定一个回调函数作为参数，以便优雅的在停止时处理PostStop信号
这样可以在突然停止时应用不同的动作。

```
import akka.actor.typed.Behavior
import akka.actor.typed.scaladsl.Behaviors
import akka.actor.typed.{ ActorSystem, PostStop }


object MasterControlProgram {
  sealed trait Command
  final case class SpawnJob(name: String) extends Command
  case object GracefulShutdown extends Command

  // Predefined cleanup operation
  def cleanup(log: Logger): Unit = log.info("Cleaning up!")

  def apply(): Behavior[Command] = {
    Behaviors
      .receive[Command] { (context, message) =>
        message match {
          case SpawnJob(jobName) =>
            context.log.info("Spawning job {}!", jobName)
            context.spawn(Job(jobName), name = jobName)
            Behaviors.same
          case GracefulShutdown =>
            context.log.info("Initiating graceful shutdown...")
            // perform graceful stop, executing cleanup before final system termination
            // behavior executing cleanup is passed as a parameter to Actor.stopped
            Behaviors.stopped { () =>
              cleanup(context.system.log)
            }
        }
      }
      .receiveSignal {
        case (context, PostStop) =>
          context.log.info("Master Control Program stopped")
          Behaviors.same
      }
  }
}

object Job {
  sealed trait Command

  def apply(name: String): Behavior[Command] = {
    Behaviors.receiveSignal[Command] {
      case (context, PostStop) =>
        context.log.info("Worker {} stopped", name)
        Behaviors.same
    }
  }
}

import MasterControlProgram._

val system: ActorSystem[Command] = ActorSystem(MasterControlProgram(), "B7700")

system ! SpawnJob("a")
system ! SpawnJob("b")

Thread.sleep(100)

// gracefully stop the system
system ! GracefulShutdown

Thread.sleep(100)

Await.result(system.whenTerminated, 3.seconds)
``` 

PreRestart 是重启的信号，在actor重启时发出，重启时不会发出PostStop

## Watching Actors

watch 用于监督非父子级别的actor，父actor默认拥有子actor的监视权
watch 可以用于非关联的actor之间监督。
被监督的actor停止的时，监督的actor会收到一个Terminated消息

```
object MasterControlProgram {
  sealed trait Command
  final case class SpawnJob(name: String) extends Command

  def apply(): Behavior[Command] = {
    Behaviors
      .receive[Command] { (context, message) =>
        message match {
          case SpawnJob(jobName) =>
            context.log.info("Spawning job {}!", jobName)
            val job = context.spawn(Job(jobName), name = jobName)
            context.watch(job)
            Behaviors.same
        }
      }
      .receiveSignal {
        case (context, Terminated(ref)) =>
          context.log.info("Job stopped: {}", ref.path.name)
          Behaviors.same
      }
  }
}
```

 watchWith 是另一种 watch, 它可以指定一个自定义消息而不是Terminated信号来监控
 这样可以包含一些额外的信息
 
```
object MasterControlProgram {
  sealed trait Command
  final case class SpawnJob(name: String, replyToWhenDone: ActorRef[JobDone]) extends Command
  final case class JobDone(name: String)
  private final case class JobTerminated(name: String, replyToWhenDone: ActorRef[JobDone]) extends Command

  def apply(): Behavior[Command] = {
    Behaviors.receive { (context, message) =>
      message match {
        case SpawnJob(jobName, replyToWhenDone) =>
          context.log.info("Spawning job {}!", jobName)
          val job = context.spawn(Job(jobName), name = jobName)
          context.watchWith(job, JobTerminated(jobName, replyToWhenDone))
          Behaviors.same
        case JobTerminated(jobName, replyToWhenDone) =>
          context.log.info("Job stopped: {}", jobName)
          replyToWhenDone ! JobDone(jobName)
          Behaviors.same
      }
    }
  }
}
```

需要注意的是，终止消息的生成与注册和终止发生的顺序无关。特别是，即使被监视的演员在注册时已经被终止，监视的演员也会收到终止消息。

多次注册不一定会导致多条消息的生成，但不能保证只收到一条这样的消息：如果终止监视行为者已经生成了消息并排队，而在这条消息被处理之前又进行了一次注册，那么第二条消息就会被排队，因为注册监视一个已经终止的行为者会导致立即生成终止的消息。

也可以使用context.unwatch(target)来取消监视另一个actor的活泼度的注册。即使被终止的消息已经在邮箱中被排队，这也是有效的；在调用unwatch之后，不会再处理该角色的终止消息。

当被监视的actor在一个已经从Cluster中移除的节点上时，终止的消息也会被发送。
