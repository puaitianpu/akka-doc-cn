# Design Principles behind Akka Streams

背后的设计原则

## What shall users of Akka Streams expect?

提供所有必要的工具来表达任何流处理拓扑，对这个领域的所有基本方面（背压、缓冲、转换、故障恢复等）进行建模，并且无论用户构建的是什么，都可以在更大的范围内重复使用


### Akka Streams does not send dropped stream elements to the dead letter office

只提供可以依赖的功能的一个重要后果是，Akka Streams不能确保通过处理拓扑发送的所有对象都会被处理。元素可能因为一些原因而被放弃

*  普通用户代码可以在map(...)操作符中消耗一个元素，并产生一个完全不同的元素作为其结果
*  常见的流操作符有意地丢弃元素，例如：take/drop/filter/conflate/buffer/...
*  流失败将不等处理完成就拆掉流，所有正在运行的元素将被丢弃
*  流的取消将向上游传播，导致上游处理步骤在没有处理完所有输入的情况下被终止


这意味着，向流中发送需要清理的JVM对象，需要用户确保在Akka Streams设施之外进行清理（例如，在超时后或在流输出上观察到它们的结果时进行清理，或者使用其他手段，如最终确定器等）


### Resulting Implementation Considerations

组成性要求部分流拓扑的可重用性，这使得我们采用了提升的方法，将数据流描述为（部分）图，这些图可以作为数据的复合源、流（也就是管道）和汇。这些构件就应是可以自由共享的，能够自由组合成更大的图。因此，这些构件的表示必须是一个不可改变的蓝图，在一个显式的步骤中被具体化，以便启动流处理。那么，由此产生的流处理引擎也是不可改变的，即拥有一个固定的拓扑结构，这个拓扑结构是由蓝图规定的。动态网络需要通过显式使用Reactive Streams接口将不同的引擎插在一起进行建模。

物化过程中往往会创建一些特定的对象，这些对象在处理引擎运行后与处理引擎进行交互时非常有用，例如用于关闭引擎或提取指标。这意味着物化函数会产生一个结果，称为图的物化值


## Interoperation with other Reactive Streams implementations

Akka Streams完全实现了Reactive Streams规范，并与所有其他符合规范的实现互操作。我们选择将Reactive Streams接口从用户级API中完全分离出来，因为我们认为它们是一个不针对终端用户的SPI。为了从Akka流拓扑中获得Publisher或Subscriber，必须使用相应的Sink.asPublisher或Source.asSubscriber元素。

所有由Akka流的默认物质化产生的流处理器都被限制为只有一个Subscriber，额外的Subscribers将被拒绝。原因是使用我们的DSL描述的流拓扑从来没有要求元素的Publisher方面的扇出行为，所有的扇出都是使用Broadcast[T]这样的显式元素来完成的。

这意味着在需要广播行为与其他Reactive Streams实现互操作的地方，必须使用Sink.asPublisher(true)（用于启用扇出支持）

通过www.DeepL.com/Translator（免费版）翻译


### Rationale and benefits from Sink/Source/Flow not directly extending Reactive Streams interfaces

关于Reactive Streams的一个有时被忽略的关键信息是，它们是一个服务提供商接口，这在早期关于规范的一个讨论中得到了深入的解释。Akka Streams是在Reactive Streams的开发过程中设计的，所以它们两者之间都受到了很大的影响。

了解到即使在Reactive规范中，类型最初也曾试图向API的用户隐藏Publisher、Subscriber和其他SPI类型，可能会有所启发。虽然由于这些内部的SPI类型最终会在某些情况下浮现在标准的最终用户面前，所以决定删除API类型，只保留Publisher、Subscriber等SPI类型。

有了这些历史知识和背景，关于标准的目的--作为互操作库的内部细节，我们可以肯定地说，不能真的说这些类型的直接继承关系可以被认为是库之间某种形式的优势或有意义的区别。相反，可以看到，向终端用户暴露这些SPI类型的API不小心泄露了内部实现细节。

作为Akka Streams一部分的Source、Sink和Flow类型的目的是提供流畅的DSL，同时也是运行这些流的 "工厂"。它们在Reactive Streams中的直接对应类型分别是Publisher、Subscriber和Processor。换句话说，Akka Streams运行在计算图的提升表示上，然后按照Reactive Streams规则将其具体化并执行。这也使得Akka Streams可以在物化步骤中执行优化，比如融合和调度器配置。

隐藏Reactive Streams接口的另一个并不明显的收益来自于org.reactivestreams.Subscriber（等）现在已经被包含在Java 9+中，从而成为Java本身的一部分，所以库应该迁移到使用java.util.concurrent.Flow.Subscriber而不是org.reactivestreams.Subscriber。选择暴露和直接扩展Reactive Streams类型的库现在将更难适应JDK9+类型--他们所有扩展Subspber和朋友的类都需要被复制或改变，以扩展完全相同的接口，但来自不同的包。在Akka中，我们只需在被要求时暴露新的类型--从JDK9发布的那天起，我们就已经支持JDK9类型了。

隐藏Reactive Streams接口的另一个，也许是更重要的原因，回到了这个解释的第一点：Reactive Streams是一个SPI的事实，因此很难在临时实现中 "正确"。因此Akka Streams不鼓励使用底层基础架构中难以实现的部分，而是为用户提供了更简单、更类型安全、但更强大的抽象。GraphStages和运算符。当然，通过使用asPublisher或fromSubscriber这样的方法，仍然可以（而且很容易）接受或获得流操作符的Reactive Streams（或JDK+ Flow）表示


## What shall users of streaming libraries expect?

我们希望库能建立在Akka Streams之上，事实上Akka HTTP就是这样一个例子，它本身就存在于Akka项目中。为了让用户能够从上面描述的Akka Streams的原则中获利，我们制定了以下规则

*  库应向他们的用户提供可重用的部件，即暴露出返回操作符的工厂，允许完全的组成性
*  库可以选择性地和额外地提供消耗和物化操作符的设施

第一条规则背后的理由是，如果不同的库只接受操作符，并期望将它们具体化，那么组成性就会被破坏：同时使用其中的两个操作符是不可能的，因为具体化只能发生一次。因此，一个库的功能必须被表达为可以由用户在库的控制之外进行物化。

第二条规则允许库为普通情况下额外提供很好的糖，一个例子是Akka HTTP API提供了一个handleWith方法来方便物化

注
这样做的一个重要后果是，一个可重用的流描述不能绑定到 "活 "资源上，任何对这些资源的连接或分配都必须推迟到物化的时候。活 "资源的例子是已经存在的TCP连接、多播Publisher等；如果TickSource的定时器是在实现时才创建的，则不属于这一类（我们的实现就是这样）。

与此不同的例外情况需要有充分的理由并仔细记录


### Resulting Implementation Constraints

Akka Streams必须使一个库能够用不可改变的蓝图来表达任何流处理实用程序。最常见的构件是

*  Source: 输出流
*  Sink: 输入流
*  Flow: 输入 - 输出
*  BidiFlow: 两个输入，两个输出
*  Graph: 一个流处理拓扑结构



