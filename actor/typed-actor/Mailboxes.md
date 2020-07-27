# Mailboxes (邮箱)

## Introduction

Akka 中每一个actor都有一个邮箱，这是消息在被actor处理之前enqueued 的地方
默认适用的是无限制的邮箱，意味着任何数量的消息都能被actor处理

适用无限制邮箱时，如果消息添加到邮箱的速度比处理消息的速度快，可以会导致应用程序的内存耗尽。
避免这种情况可以适用有界的邮箱，有界邮箱会在邮箱满了的时候将新消息的传递给死信

## Selecting what mailbox is used (选择使用的邮箱)

Selecting a Mailbox Type for an Actor

要给actor指定使用一个特定的邮箱， 使用 MailboxSelector 创建一个 Props 来生成actor

```
context.spawn(childBehavior, "bounded-mailbox-child", MailboxSelector.bounded(100))

val props = MailboxSelector.fromConfig("my-app.my-special-mailbox")
context.spawn(childBehavior, "from-config-mailbox-child", props)
```

`fromConfig` 是从配置中读取一个指定的邮箱类型

```
my-app {
  my-special-mailbox {
    mailbox-type = "akka.dispatch.SingleConsumerOnlyUnboundedMailbox"
  }
}
```


Default Mailbox

没有指定邮箱时，默认使用的是 `SingleConsumerOnlyUnboundedMailbox`


Which Configuration is passed to the Mailbox Type

每个邮箱类型由一个类实现，该类扩展了MailboxType，并接受两个构造函数参数：一个ActorSystem.Settings对象和一个Config
后者是通过名称从 ActorSystem 配置中获取配置对象，通过配置 `mailbox-type` 来重写 ID Key ，并通过添加fall-back 来配置默认的 mailbox configuration section。



## Mailbox Implementations (邮箱的实现)

Akka提供了许多邮箱实现

SingleConsumerOnlyUnboundedMailbox (默认邮箱)

*  默认的邮箱
*  是一个 Multiple-Producer (多生产者) Single-Consumer (单消费者) 队列， 不能和 `BalancingDispatcher` 一起使用
*  没有阻塞
*  没有绑定
*  配置名称: `"akka.dispatch.SingleConsumerOnlyUnboundedMailbox"`


UnboundedMailbox (无界邮箱)

*  由 `java.util.concurrent.ConcurrentLinkedQueue` 支持
*  没有阻塞
*  没有绑定
*  配置名称: `“unbounded”` 或 `"akka.dispatch.UnboundedMailbox"`

NonBlockingBoundedMailbox (没有阻塞有绑定邮箱)

*  是一个非常高效的Multiple-Producer (多生产者) Single-Consumer (单消费者) 队列
*  没有阻塞(将溢出的消息丢弃到死信)
*  有绑定
*  配置名称: `"akka.dispatch.NonBlockingBoundedMailbox"`

UnboundedControlAwareMailbox (没有绑定有控制的邮箱)
是一个扩展 `akka.dispatch.ControlMessage` 优先传递优先级较高的消息

*  由两个 `java.util.concurrent.ConcurrentLinkedQueue` 支持
*  与 `UnboundedStablePriorityMailbox` 对比，对于同等优先级的邮件发送是不确定的
*  没有阻塞
*  没有绑定
*  配置名称: `"akka.dispatch.UnboundedControlAwareMailbox"`

UnboundedPriorityMailbox (无限制优先级邮箱)

*  由 `java.util.concurrent.PriorityBlockingQueue` 支持
*  与 `UnboundedStablePriorityMailbox` 对比，对于同等优先级的邮件发送顺序是不确定的
*  没有阻塞
*  没有绑定
*  配置名称: `"akka.dispatch.UnboundedPriorityMailbox"`


UnboundedStablePriorityMailbox (无界稳定优先级邮箱)

*  将 `java.util.concurrent.PriorityBlockingQueue` 包装在 `akka.util.PriorityQueueStabilizer` 中实现的
*  对于同等优先级的消息，FIFO顺序会被保留--与UnboundedPriorityMailbox形成对比
*  没有阻塞
*  没有绑定
*  配置名称: `"akka.dispatch.UnboundedStablePriorityMailbox"`




其他有界邮箱的实现，如果达到了容量，将阻断发件人，并配置了非零的消息推送超时时间。
注意：如果达到了容量，并且配置了非零的消息推送-超时时间，则会阻止发件人。

以下邮箱只能在消息推送超时时间为零时使用。

BoundedMailbox (绑定邮箱)

*  由 `java.util.concurrent.LinkedBlockingQueue` 支持
*  设置的邮箱推送消息超时时间不为0时生效
*  有绑定
*  配置名称: `“bounded”` 或 `"akka.dispatch.BoundedMailbox"`

BoundedPriorityMailbox (绑定优先级邮箱)

*  将 `java.util.PriorityQueue` 包装在 `akka.util.BoundedBlockingQueue` 中实现的
*  同等优先级的消息发送顺序是不确定的--与BoundedStablePriorityMailbox形成对比
*  设置的邮箱推送消息超时时间不为0时生效
*  有绑定
*  配置名称: `"akka.dispatch.BoundedPriorityMailbox"`
 
BoundedStablePriorityMailbox (绑定稳定优先级邮箱)

*  将 `java.util.PriorityQueue` 包装在 `akka.util.PriorityQueueStabilizer` 中实现的
*  对于同等优先级的消息，FIFO顺序会被保留--与BoundedPriorityMailbox形成对比
*  设置的邮箱推送消息超时时间不为0时生效
*  有绑定
*  配置名称: `"akka.dispatch.BoundedStablePriorityMailbox"`

BoundedControlAwareMailbox (有绑定有控制的邮箱)

*  扩展 `akka.dispatch.ControlMessage` 优先传递优先级较高的消息
*  由两个 `java.util.concurrent.ConcurrentLinkedQueue` 支持, 如果容量已达到，则在enqueue上阻塞。
*  设置的邮箱推送消息超时时间不为0时生效
*  有绑定
*  配置名称: `"akka.dispatch.BoundedControlAwareMailbox"`


## Custom Mailbox type (自定义邮箱)

例子: 

```
// Marker trait used for mailbox requirements mapping
trait MyUnboundedMessageQueueSemantics
```

```
import akka.actor.ActorRef
import akka.actor.ActorSystem
import akka.dispatch.Envelope
import akka.dispatch.MailboxType
import akka.dispatch.MessageQueue
import akka.dispatch.ProducesMessageQueue
import com.typesafe.config.Config
import java.util.concurrent.ConcurrentLinkedQueue
import scala.Option

object MyUnboundedMailbox {
  // This is the MessageQueue implementation
  class MyMessageQueue extends MessageQueue with MyUnboundedMessageQueueSemantics {

    private final val queue = new ConcurrentLinkedQueue[Envelope]()

    // these should be implemented; queue used as example
    def enqueue(receiver: ActorRef, handle: Envelope): Unit =
      queue.offer(handle)
    def dequeue(): Envelope = queue.poll()
    def numberOfMessages: Int = queue.size
    def hasMessages: Boolean = !queue.isEmpty
    def cleanUp(owner: ActorRef, deadLetters: MessageQueue): Unit = {
      while (hasMessages) {
        deadLetters.enqueue(owner, dequeue())
      }
    }
  }
}

// This is the Mailbox implementation
class MyUnboundedMailbox extends MailboxType with ProducesMessageQueue[MyUnboundedMailbox.MyMessageQueue] {

  import MyUnboundedMailbox._

  // This constructor signature must exist, it will be called by Akka
  def this(settings: ActorSystem.Settings, config: Config) = {
    // put your initialization code here
    this()
  }

  // The create method is called to create the MessageQueue
  final override def create(owner: Option[ActorRef], system: Option[ActorSystem]): MessageQueue =
    new MyMessageQueue()
}
```


