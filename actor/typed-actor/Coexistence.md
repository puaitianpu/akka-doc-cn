# Coexistence (共存)

## Introduction

本节描述的是 classic (经典的actor) 和 typed 共存

有两种类型的ActorSystem: `akka.actor.ActorSystem` 和 `akka.actor.typed.ActorSystem`

目前，typed actor 是在 classic actor 基础上实现的，将来可能会改掉

Typed 和 classic 可以通过以下方式交互

*  classic actor 可以创建 typed actor
*  typed actor 和 class actor 之间可以相互发送消息
*  typed actor 和 class actor 之间可以相互生成子actor并监督
*  typed actor 和 class actor 之间可以相互检测
*  classic actor system 可以转化成 typed actor system


在例子中，akka.actor包被别名为 `classic`
`import akka.{ actor => classic }`


## Classic to typed

在共存的同时，你的应用程序可能仍然有一个classic ActorSystem。
可以将其转换为typed ActorSystem，这样新的代码和迁移的部分就不会依赖于classic系统。

```
// adds support for actors to a classic actor system and context
import akka.actor.typed.scaladsl.adapter._

val system = akka.actor.ActorSystem("ClassicToTypedSystem")
val typedSystem: ActorSystem[Nothing] = system.toTyped
```

如果有一个这样新建的typed actor， 要任何 watch 或者发送消息到 classic actor

```
object Typed {
  sealed trait Command
  final case class Ping(replyTo: ActorRef[Pong.type]) extends Command
  case object Pong

  def apply(): Behavior[Command] =
    Behaviors.receive { (context, message) =>
      message match {
        case Ping(replyTo) =>
          context.log.info(s"${context.self} got Ping from $replyTo")
          // replyTo is a classic actor that has been converted for coexistence
          replyTo ! Pong
          Behaviors.same
      }
    }
}
```

创建一个 classic actor 
`val classicActor = system.actorOf(Classic.props())`

然后，它可以创建一个typed actor。 watch 和 发送消息给它

```
class Classic extends classic.Actor with ActorLogging {
  // context.spawn is an implicit extension method
  val second: ActorRef[Typed.Command] =
    context.spawn(Typed(), "second")

  // context.watch is an implicit extension method
  context.watch(second)

  // self can be used as the `replyTo` parameter here because
  // there is an implicit conversion from akka.actor.ActorRef to
  // akka.actor.typed.ActorRef
  // An equal alternative would be `self.toTyped`
  second ! Typed.Ping(self)

  override def receive = {
    case Typed.Pong =>
      log.info(s"$self got Pong from ${sender()}")
      // context.stop is an implicit extension method
      context.stop(second)
    case classic.Terminated(ref) =>
      log.info(s"$self observed termination of $ref")
      context.stop(self)
  }
}
``` 

这个操作需要导入
```
// adds support for actors to a classic actor system and context
import akka.actor.typed.scaladsl.adapter._
```


## Typed to classic

以下例子展示从typed actor 创建 classic actor并watch 和 发送消息

```
bject Classic {
  def props(): classic.Props = classic.Props(new Classic)
}
class Classic extends classic.Actor {
  override def receive = {
    case Typed.Ping(replyTo) =>
      replyTo ! Typed.Pong
  }
}
```

创建一个 typed actor 的 actor 系统
```
val system = classic.ActorSystem("TypedWatchingClassic")
val typed = system.spawn(Typed.behavior, "Typed")
```

使用 typed actor 创建 classic actor, watch 和发送消息并且回应响应

```
object Typed {
  final case class Ping(replyTo: akka.actor.typed.ActorRef[Pong.type])
  sealed trait Command
  case object Pong extends Command

  val behavior: Behavior[Command] =
    Behaviors.setup { context =>
      // context.actorOf is an implicit extension method
      val classic = context.actorOf(Classic.props(), "second")

      // context.watch is an implicit extension method
      context.watch(classic)

      // illustrating how to pass sender, toClassic is an implicit extension method
      classic.tell(Typed.Ping(context.self), context.self.toClassic)

      Behaviors
        .receivePartial[Command] {
          case (context, Pong) =>
            // it's not possible to get the sender, that must be sent in message
            // context.stop is an implicit extension method
            context.stop(classic)
            Behaviors.same
        }
        .receiveSignal {
          case (_, akka.actor.typed.Terminated(_)) =>
            Behaviors.stopped
        }
    }
}
```

Note 
typed actor 向 classic actor 发送消息时，不能直接 sender 。
需要显示的使用 ActorContext[T].self


## Supervision

classic actor 的默认监督行为是重启， typed actor 默认监督行为是停止
当两者结合使用的时候，默认的监督行为是根据子actor的类型来决定的