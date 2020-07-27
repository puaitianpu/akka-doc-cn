# Learning Akka Typed from Classic

## Package names

Akka Typed中包名的约定是在对应的Akka经典包名中加上typed.scaladsl和typed.javadsl，
scaladsl和javadsl是将Scala和Java API分开


## Actor definition

classic actor 的定义是通过继承 `akka.actor.Actor`

typed actor 的定义是继承 `akka.actor.typed.scaladsl.AbstractBehavior`

在typed中，也可以通过函数来定义一个actor，而不是扩展一个类。这就是所谓的函数式。

Classic HelloWorld actor:

```
import akka.actor.Actor
import akka.actor.ActorLogging
import akka.actor.Props

object HelloWorld {
  final case class Greet(whom: String)
  final case class Greeted(whom: String)

  def props(): Props =
    Props(new HelloWorld)
}

class HelloWorld extends Actor with ActorLogging {
  import HelloWorld._

  override def receive: Receive = {
    case Greet(whom) =>
      log.info("Hello {}!", whom)
      sender() ! Greeted(whom)
  }
}

```

Typed HelloWorld actor:

```
import akka.actor.typed.ActorRef
import akka.actor.typed.Behavior
import akka.actor.typed.scaladsl.AbstractBehavior
import akka.actor.typed.scaladsl.ActorContext
import akka.actor.typed.scaladsl.Behaviors

object HelloWorld {
  final case class Greet(whom: String, replyTo: ActorRef[Greeted])
  final case class Greeted(whom: String, from: ActorRef[Greet])

  def apply(): Behavior[HelloWorld.Greet] =
    Behaviors.setup(context => new HelloWorld(context))
}

class HelloWorld(context: ActorContext[HelloWorld.Greet]) extends AbstractBehavior[HelloWorld.Greet](context) {
  import HelloWorld._

  override def onMessage(message: Greet): Behavior[Greet] = {
    context.log.info("Hello {}!", message.whom)
    message.replyTo ! Greeted(message.whom, context.self)
    this
  }
}

```

为什么叫Behavior而不是Actor？

在Typed中，Behavior定义了如何处理输入的消息。在处理完一个消息后，可能会返回一个不同的Behavior来处理下一个消息。
这意味着一个actor是以一个初始Behavior开始的，并可能在其生命周期中改变Behavior。这将在关于become的章节中详细描述。

请注意，Behavior有一个类型参数，描述它可以处理的消息类型。对于classic actor 来说，这个信息并没有被明确定义。


## actorOf and Props

classic actor 是通过 `ActorContext` 或 `ActorSystem` 的 `actorOf` 方法来启动的

typed 中对应的方法是 `akka.actor.typed.scaladsl.ActorContext` 中的 `spawn`

在`akka.actor.typed.scaladsl.ActorSystem` 中没有用于创建顶层actor 的`spawn`方法。
取而代之的是，有一个由 user guardian Behavior 定义的顶层actor，它是在启动ActorSystem时给出的。
其他actor是作为该user guardian Behavior的子actor或actor层次结构中其他actor的子代启动的。这一点在 ActorSystem 中有更多解释。

actorOf方法需要一个akka.actor.Props参数，它就像一个创建actor实例的工厂，在重新启动actor时创建新的实例时也会用到它。Props还可以定义额外的属性，比如为actor使用哪个dispatcher。

在 typed 中，spawn 方法直接从给定的 Behavior 创建一个 actor，而不需要使用 Props 工厂。然而，它接受一个可选的akka.actor.typed.Props来指定Actor元数据。
当使用面向对象风格的类扩展AbstractBehavior时，工厂方面则通过Behaviors.setup来定义。对于函数风格，通常不需要工厂。

额外的属性，例如为actor使用哪个dispatcher，仍然可以通过spawn方法中可选的akka.actor.typed.Props参数给出。

actorOf的name参数是可选的，如果没有定义，actor将有一个生成的名字。在Typed中的对应是通过spwnAnonymous方法实现的。


## ActorRef
`akka.actor.ActorRef`与`akka.actor.typed.ActorRef`有对应关系。
不同的是，后者有一个类型参数，描述了actor可以处理哪些消息。这个信息并不是为经典actor定义的，
你可以向 classic actor ActorRef发送任何类型的消息，即使这个actor可能不理解它。

## ActorSystem

`akka.actor.ActorSystem` 和 `akka.actor.typed.ActorSystem`有对应关系。
不同的是，在Typed中创建一个ActorSystem时，你给它一个Behavior，它将被用作顶层的actor，也被称为user guardian。

应用程序的其他角色是由user guardian创建的，同时执行Akka组件的初始化，如集群共享。
相反，在classic actor 的ActorSystem中，这种初始化通常是从 "外部 "执行的。

classic actor ActorSystem的actorOf方法通常用于创建几个（或许多）顶层Actor。Typed中的ActorSystem没有这种能力。

## become
classic actor 可以通过 ActorContext的become 来改变它的消息处理behavior
在typed 中，这是通过消息处理后返回一个新的Behavior来实现的。返回的Behavior将用于下一个收到的消息。

在Typed中没有对应的unbecome。相反，你必须明确地跟踪并返回 "前一个 "Behavior。

## sender
Typed中没有sender()。相反，你必须显式地在消息中包含一个代表发送者的ActorRef--或者说代表在哪里发送回复。

Typed中没有隐式sender的原因是，在编译时不可能知道sender ActorRef[T]的类型。
在消息中显式地定义这个类型也会好很多，因为消息协议所期望的东西会变得更加清晰。

## parent

Typed中没有parent。相反，你必须在构造Behavior时显式地包含父类的ActorRef作为参数。

在Typed中没有父函数的原因是，如果在Behavior中没有额外的类型参数，就不可能在编译时知道父函数ActorRef[T]的类型。
为了测试的目的，也最好把父类传递进来，因为它可以在测试中被探针替换或被支管出来。

## Supervision
classic 和 typed 之间的一个重要区别是，在 typed 中，如果抛出了异常，并且没有定义监督策略，那么默认情况下actor会被停止。相反，在classic中，默认情况下，演员会被重新启动。

在classic中，子演员的监督策略是通过覆盖父演员的 supervisorStrategy 方法来定义的。

在Typed中，监督者策略是通过用Behaviors.supervise包装子actor的Behavior来定义的。

classic的BackoffSupervisor在Typed中作为普通的SupervisorStrategy.restartWithBackoff支持。

Typed中不支持SupervisorStrategy.Escalate，但可以通过层次结构向上实现类似的Bubble故障描述。

## Lifecycle hooks

classic actor 有 preStart、preRestart、postRestart和postStop等方法，这些方法可以被重写，以作用于演员生命周期的变化。
Typed中对应的PreRestart和PostStop信号信息支持这一点。没有PreStart和PostRestart信号，因为这样的操作可以从Behaviors.setup或AbstractBehavior类的构造函数中完成

请注意，在classic中，当actor被重新启动时，postStop生命周期钩子也会被调用。
在Typed中不是这样的，只有PreRestart信号被发出。如果你需要在重启和停止时都进行资源清理，你必须对PreRestart和PostStop都进行清理。


## watch

watch和Terminated消息差不多，只是在Typed中增加了一些功能。

Terminated在Typed中是一个信号，因为它与Behavior的声明消息类型不同。

Typed中ActorContext的watchWith方法可以用来代替Terminated信号发送消息。

在观察子actor时，可以通过ChildFailed信号查看子actor是自愿终止还是由于失败，ChildFailed信号是Terminated的一个子类。

## Stopping
classic actor 可以用ActorContext或ActorSystem的stop方法停止。
在Typed中，一个actor通过返回Behaviors.stoped来停止自己。ActorContext中也有一个stop方法，但它只能用于停止直接的子actor，而不是任何任意的actor。

Typed中不支持PoisonPill。相反，如果你需要请求一个actor停止，你应该定义一个actor能理解的消息，并让它在收到这个消息时返回Behaviors.stop。

## ActorSelection
Typed中不支持ActorSelection。相反，Receptionist应该用于通过注册的键来寻找演员。

ActorSelection可以用于发送消息到一个没有目标ActorRef的路径。需要注意的是，可以使用群组路由器来实现。

## ask

classic 的ask模式为响应返回一个Future。

对应的ask存在于Typed中，当请求者本身不是actor时，它是好的。它位于akka.actor.typed.scaladsl.AskPattern中。

当请求者是一个actor时，最好使用Typed中ActorContext的ask方法。它的好处是不必将运行在不同线程上的Future回调与actor代码混合。


## pileTo

pipeTo在actor中通常与ask一起使用，typed 中 ActorContext 的 ask 方法消除了对 pipeTo 的需求。
然后，与其他返回Future的API交互时，仍然需要使用它将消息发送给actor.
为此，在Typed中的ActorContext中有一个 pipeToSelf方法。

## ActorContext.children

ActorContext有方法children和child来检索Typed和Classic中启动的子actor的ActorRef。

返回的ActorRef的类型是未知的，因为不同的actor返回的类型不一样。
因此，当目的是向他们发送消息时，这不是一个有用的查找子actor的方法。

与其通过ActorContext查找子actor，建议使用特定于应用程序的集合来记录子actor，
比如Map[String，ActorRef[Child.Command]] 

```
object Parent {
  sealed trait Command
  case class DelegateToChild(name: String, message: Child.Command) extends Command
  private case class ChildTerminated(name: String) extends Command

  def apply(): Behavior[Command] = {
    def updated(children: Map[String, ActorRef[Child.Command]]): Behavior[Command] = {
      Behaviors.receive { (context, command) =>
        command match {
          case DelegateToChild(name, childCommand) =>
            children.get(name) match {
              case Some(ref) =>
                ref ! childCommand
                Behaviors.same
              case None =>
                val ref = context.spawn(Child(), name)
                context.watchWith(ref, ChildTerminated(name))
                ref ! childCommand
                updated(children + (name -> ref))
            }

          case ChildTerminated(name) =>
            updated(children - name)
        }
      }
    }

    updated(Map.empty)
  }
}
```


## Remote deployment

typed 不支持在远程节点上启动一个actor--也就是所谓的远程部署


这个功能不鼓励使用，因为它经常导致节点之间的紧密耦合和不理想的失败处理。
例如，如果父actor的节点崩溃，所有远程部署的子actor都会被拖垮。
有时可以希望这样，但很多时候是在不知不觉中使用的。这可以通过其他手段来实现，比如使用watch


## Routers
typed 中的路由器对比classic 的形式更为简化

分组路由器的目的地是在Receptionist中注册的，这使得它们具有Cluster意识，也比classic的分组路由器更加动态。

Pool routers只适用于Typed中的本地角色目的地，因为不支持远程部署

## FSM

对于classic actor， 有明确的支持来构建有限状态机。
typed 中不需要支持，因为用Behavior就直接能实现了

## Timers

在classic actor 中，使用Timers来获取消息的延迟和定期调度。
typed 中，可以通过Behaviors.withTimers获取类似的功能

## Stash

在classic actor中，可以使用Stash来获得消息的存储。
在typed 中，可以通过Behaviors.withStash获得类似的功能

## PersistentActor
classic 的`PersistentActor`对应的是`akka.persistence.typed.scaladsl.EventSourcedBehavior`
Typed API的指导性更强，便于事件源的最佳实践。它还与Cluster Sharding有更紧密的集成。


## Asynchronous Testing

异步测试的测试套件比较类似



## Synchronous Testing
classic 和 typed 有不同的测试套件用于同步测试。

Typed中的行为可以被隔离测试，而不必被打包成一个actor。因此，测试可以完全同步运行，而不必担心超时和虚假失败。

BehaviorTestKit提供了一种很好的方式来以确定性的方式对Behavior进行单元测试，但它也有一些需要注意的限制。类似的局限性也存在于经典行为体的同步测试中。

