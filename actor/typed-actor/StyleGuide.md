# Style guide (风格指南)

一个使用函数式风格实现的计数器actor的例子:

```
import akka.actor.typed.Behavior
import akka.actor.typed.scaladsl.ActorContext
import akka.actor.typed.scaladsl.Behaviors

object Counter {
  sealed trait Command
  case object Increment extends Command
  final case class GetValue(replyTo: ActorRef[Value]) extends Command
  final case class Value(n: Int)

  def apply(): Behavior[Command] =
    counter(0)

  private def counter(n: Int): Behavior[Command] =
    Behaviors.receive { (context, message) =>
      message match {
        case Increment =>
          val newValue = n + 1
          context.log.debug("Incremented counter to [{}]", newValue)
          counter(newValue)
        case GetValue(replyTo) =>
          replyTo ! Value(n)
          Behaviors.same
      }
    }
}
```


## Passing around too many parameters

使用函数式风格时，有时候会需要传递很多参数

example:

```
// this works, but previous example is better for structuring more complex behaviors
object Counter {
  sealed trait Command
  case object Increment extends Command
  final case class IncrementRepeatedly(interval: FiniteDuration) extends Command
  final case class GetValue(replyTo: ActorRef[Value]) extends Command
  final case class Value(n: Int)

  def apply(name: String): Behavior[Command] =
    Behaviors.setup { context =>
      Behaviors.withTimers { timers =>
        def counter(n: Int): Behavior[Command] =
          Behaviors.receiveMessage {
            case IncrementRepeatedly(interval) =>
              context.log.debugN(
                "[{}] Starting repeated increments with interval [{}], current count is [{}]",
                name,
                interval,
                n)
              timers.startTimerWithFixedDelay(Increment, interval)
              Behaviors.same
            case Increment =>
              val newValue = n + 1
              context.log.debug2("[{}] Incremented counter to [{}]", name, newValue)
              counter(newValue)
            case GetValue(replyTo) =>
              replyTo ! Value(n)
              Behaviors.same
          }

        counter(0)
      }
    }
}
```

## Behavior factory method

初始化 behaviors 应该通过伴生对象的工厂方法来创建， 这样当实现改变的时候，behavior 的用法不会改变

工厂方法是一个很好检索资源的地方，比如 `Behaviors.withTimers`, `Behaviors.withStash`, `ActorContext`, `Behaviors.setup`

工厂方法命名惯例是apply, 一致的命名使代码的读者更容易找到behavior的 "起点"

Example: 

```
object CountDown {
  sealed trait Command
  case object Down extends Command

  // factory for the initial `Behavior`
  def apply(countDownFrom: Int, notifyWhenZero: ActorRef[Done]): Behavior[Command] =
    new CountDown(notifyWhenZero).counter(countDownFrom)
}

private class CountDown(notifyWhenZero: ActorRef[Done]) {
  import CountDown._

  private def counter(remaining: Int): Behavior[Command] = {
    Behaviors.receiveMessage {
      case Down =>
        if (remaining == 1) {
          notifyWhenZero.tell(Done)
          Behaviors.stopped
        } else
          counter(remaining - 1)
    }
  }

}
```

## Where to define messages

当发送或接收actor消息时，应该以定义它们的actor/behavior的名称作为前缀，以避免歧义

如果几个actor共享同一个消息协议，应该把该协议定义在单独的对象里面

```
object CounterProtocol {
  sealed trait Command

  final case class Increment(delta: Int, replyTo: ActorRef[OperationResult]) extends Command
  final case class Decrement(delta: Int, replyTo: ActorRef[OperationResult]) extends Command

  sealed trait OperationResult
  case object Confirmed extends OperationResult
  final case class Rejected(reason: String)
}
```


## Public versus private messages

有的时候actor有一些消息只用于内部实现，而不是公用的。例如定时器消息或ask或messageAdapter包装的消息
这样的消息应该被声明为私有的，这样它们就不能从actor的外部被访问和发送。注意，它们仍然必须扩展公共Command特性

Example: 

```
object Counter {
  sealed trait Command
  case object Increment extends Command
  final case class GetValue(replyTo: ActorRef[Value]) extends Command
  final case class Value(n: Int)

  // Tick is private so can't be sent from the outside
  private case object Tick extends Command

  def apply(name: String, tickInterval: FiniteDuration): Behavior[Command] =
    Behaviors.setup { context =>
      Behaviors.withTimers { timers =>
        timers.startTimerWithFixedDelay(Tick, tickInterval)
        new Counter(name, context).counter(0)
      }
    }
}

class Counter private (name: String, context: ActorContext[Counter.Command]) {
  import Counter._

  private def counter(n: Int): Behavior[Command] =
    Behaviors.receiveMessage {
      case Increment =>
        val newValue = n + 1
        context.log.debug2("[{}] Incremented counter to [{}]", name, newValue)
        counter(newValue)
      case Tick =>
        val newValue = n + 1
        context.log.debug2("[{}] Incremented counter by background tick to [{}]", name, newValue)
        counter(newValue)
      case GetValue(replyTo) =>
        replyTo ! Value(n)
        Behaviors.same
    }
}

```

另一种方法是使用类型层次结构，并缩小公共消息的超级类型，作为与所有行为者消息的超级类型不同的类型。前一种方法是被推荐的，但知道这种替代方法是很好的，因为当使用共享消息协议类时，它可能是有用的，正如在哪里定义消息中所描述的那样。

example: 

```
// above example is preferred, but this is possible and not wrong
object Counter {
  // The type of all public and private messages the Counter actor handles
  sealed trait Message

  /** Counter's public message protocol type. */
  sealed trait Command extends Message
  case object Increment extends Command
  final case class GetValue(replyTo: ActorRef[Value]) extends Command
  final case class Value(n: Int)

  // The type of the Counter actor's internal messages.
  sealed trait PrivateCommand extends Message
  // Tick is a private command so can't be sent to an ActorRef[Command]
  case object Tick extends PrivateCommand

  def apply(name: String, tickInterval: FiniteDuration): Behavior[Command] = {
    Behaviors
      .setup[Counter.Message] { context =>
        Behaviors.withTimers { timers =>
          timers.startTimerWithFixedDelay(Tick, tickInterval)
          new Counter(name, context).counter(0)
        }
      }
      .narrow // note narrow here
  }
}

class Counter private (name: String, context: ActorContext[Counter.Message]) {
  import Counter._

  private def counter(n: Int): Behavior[Message] =
    Behaviors.receiveMessage {
      case Increment =>
        val newValue = n + 1
        context.log.debug2("[{}] Incremented counter to [{}]", name, newValue)
        counter(newValue)
      case Tick =>
        val newValue = n + 1
        context.log.debug2("[{}] Incremented counter by background tick to [{}]", name, newValue)
        counter(newValue)
      case GetValue(replyTo) =>
        replyTo ! Value(n)
        Behaviors.same
    }
}

```

## Partial versus total Function

建议使用 sealed 作为消息传递的命令

```
sealed trait Command
case object Down extends Command
final case class GetValue(replyTo: ActorRef[Value]) extends Command
final case class Value(n: Int)
```
这就是Behaviors.receive，Behaviors.receiveMessage采取Function而不是PartialFunction的主要原因。


## ask versus

当使用 AskPattern 时， 建议使用ask方法而不是 `?`

```
import akka.actor.typed.scaladsl.AskPattern._
import akka.util.Timeout

implicit val timeout: Timeout = Timeout(3.seconds)
val counter: ActorRef[Command] = ???

val result: Future[OperationResult] = counter.ask(replyTo => Increment(delta = 2, replyTo))
```

可以使用 `_` 来代替 replyTo 这样更简洁

```
val result2: Future[OperationResult] = counter.ask(Increment(delta = 2, _))
```

## Nesting setup

当一个actor behaviors 需要 `setup`, `withTimers`, `withStash` 中多个方法时，可以通过嵌套的形式来使用

```
def apply(): Behavior[Command] =
  Behaviors.setup[Command](context =>
    Behaviors.withStash(100)(stash =>
      Behaviors.withTimers { timers =>
        context.log.debug("Starting up")

        // behavior using context, stash and timers ...
      }))
```

## Additional naming conventions

replyTo是消息中ActorRef[Reply]参数的典型名称，应该向其发送回复或确认。

传入到actor的消息通常被称为命令，因此，actor可以处理的所有消息的超类型通常是 `sealed trait Command`

对于事件源行为（EventSourcedBehavior）持久化的事件使用过去式，因为那些代表已经发生的事实，例如Incremented。



