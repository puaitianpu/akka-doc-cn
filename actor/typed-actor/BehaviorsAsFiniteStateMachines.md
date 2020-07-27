# Behaviors as finite state machines (实现有限状态机)

actor 可以用来模拟有限状态机(FSM)

什么是有限状态机？
有限状态机，（英语：Finite-state machine, FSM），又称有限状态自动机，简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型。

有限状态机一般都有以下特点：（1）可以用状态来描述事物，并且任一时刻，事物总是处于一种状态；（2）事物拥有的状态总数是有限的；（3）通过触发事物的某些行为，可以导致事物从一种状态过渡到另一种状态；（4）事物状态变化是有规则的，A状态可以变换到B，B可以变换到C，A却不一定能变换到C；（5）同一种行为，可以将事物从多种状态变成同种状态，但是不能从同种状态变成多种状态


这个例子演示了如何来实现:

*  使用不同的 behaviors 来模拟状态
*  通过将 behavior 表示为一种方法来建立每个状态下的数据存储模型
*  实现状态超时
*  Actor 可以接收的消息类型为FSM可以接收的事件
*  每一个状态都会成为一个不同的 behavior, 在处理完一个消息后，会以 behavior 的形式返回下一个状态



```
object Buncher {

  // FSM event becomes the type of the message Actor supports
  sealed trait Event
  final case class SetTarget(ref: ActorRef[Batch]) extends Event
  final case class Queue(obj: Any) extends Event
  case object Flush extends Event
  private case object Timeout extends Event
}
```


```
object Buncher {

  // FSM event becomes the type of the message Actor supports
  sealed trait Event
  final case class SetTarget(ref: ActorRef[Batch]) extends Event
  final case class Queue(obj: Any) extends Event
  case object Flush extends Event
  private case object Timeout extends Event
}
```


```
object Buncher {
  // states of the FSM represented as behaviors

  // initial state
  def apply(): Behavior[Event] = idle(Uninitialized)

  private def idle(data: Data): Behavior[Event] = Behaviors.receiveMessage[Event] { message =>
    (message, data) match {
      case (SetTarget(ref), Uninitialized) =>
        idle(Todo(ref, Vector.empty))
      case (Queue(obj), t @ Todo(_, v)) =>
        active(t.copy(queue = v :+ obj))
      case _ =>
        Behaviors.unhandled
    }
  }

  private def active(data: Todo): Behavior[Event] =
    Behaviors.withTimers[Event] { timers =>
      // instead of FSM state timeout
      timers.startSingleTimer(Timeout, 1.second)
      Behaviors.receiveMessagePartial {
        case Flush | Timeout =>
          data.target ! Batch(data.queue)
          idle(data.copy(queue = Vector.empty))
        case Queue(obj) =>
          active(data.copy(queue = data.queue :+ obj))
      }
    }

}
```

设置状态超时使用 `Behaviors.whithTimers 以及 startSingleTimer `

