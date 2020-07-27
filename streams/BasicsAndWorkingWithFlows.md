# Basics and working with Flows

## Core concepts

Akka Streams是一个使用有界缓冲空间来处理和传输元素序列的库。后一个属性就是我们所说的有界性，它是Akka Streams的定义特征。
翻译成日常用语，可以表达一个处理实体的链（或者我们后面看到的图）。这些实体中的每一个都独立于其他实体执行（也可能是并发的），同时在任何给定的时间内只缓冲有限数量的元素。这种有界缓冲的属性是与行为体模型的区别之一，在行为体模型中，每个行为体通常都有一个无界的，或者一个有界的，但会掉落的邮箱。Akka Stream处理实体拥有不掉线的有界 "邮箱"。

在我们继续之前，让我们定义一些基本术语，这些术语将在整个文档中使用

### Stream

一个移动和转化数据的执行过程

### Element

元素是流的处理单元。所有的操作都是将元素从上游转换和转移到下游。缓冲区大小总是以元素的数量表示，与元素的实际大小无关

### Back-pressure

流量控制的一种手段，是数据的消费者通知生产者他们当前的可用性，有效地减慢上游生产者的速度，以匹配他们的消费速度。在Akka流的语境中，背压始终被理解为非阻塞和异步

### Non-Blocking

意味着某项操作不会阻碍调用线程的进度，即使需要很长时间才能完成请求的操作

### Graph

对流处理拓扑的描述，定义了流运行时元素流动的路径

### Operator

构建Graph的所有构件的通用名称。操作符的例子有map()、filter()、扩展GraphStages的自定义操作符以及像Merge或Broadcast这样的图形连接。有关内置运算符的完整列表，请参见运算符索引


当我们谈到异步、非阻塞反压时，我们指的是Akka Streams中可用的操作符不会使用阻塞调用，而是使用异步消息传递来相互交换消息。这样它们就可以在不阻塞其线程的情况下减缓一个快速生产者的速度。这是一种线程池友好的设计，因为需要等待的实体（一个快速生产者等待一个慢速消费者）不会阻塞线程，而是可以将其交还给底层线程池进一步使用


## Defining and running streams

线性处理流水线可以用Akka Streams表达，使用以下核心抽象

### Source

一个只有一个输出的操作者，每当下游操作者准备好接收数据元素时，就会发出数据元素

### Sink

一个正好有一个输入的操作者，请求和接受数据元素，可能会拖慢元素的上游生产者

### Flow

有一个输入和输出的运算器，它通过变换流经它的数据元素来连接它的上游和下游

### RunnableGraph

一个两端分别 "连接 "到Source和Sink的Flow，并准备好被 `run()`

可以将一个Flow附加到一个Source上，从而得到一个复合源，也可以将一个Flow预先附加到一个Sink上，从而得到一个新的Sink。当一个流同时拥有源和汇之后，它将被RunnableGraph类型所代表，表明它已经准备好被执行

重要的是要记住，即使通过连接所有的源、汇和不同的操作符来构造RunnableGraph，在它被物化之前，任何数据都不会流经它。物化是分配运行Graph所描述的计算所需的所有资源的过程（在Akka Streams中，这通常涉及启动Actor）。由于Flows是对处理流水线的描述，它们是不可变的、线程安全的、可自由共享的，这意味着例如可以安全地在Actor之间共享和发送它们，让一个Actor准备工作，然后在代码中一些完全不同的地方将其物化

```scala
val source = Source(1 to 10)
val sink = Sink.fold[Int, Int](0)(_ + _)

// connect the Source to the Sink, obtaining a RunnableGraph
val runnable: RunnableGraph[Future[Int]] = source.toMat(sink)(Keep.right)

// materialize the flow and get the value of the FoldSink
val sum: Future[Int] = runnable.run()
```

在运行（物化）RunnableGraph[T]后，我们会得到T类型的物化值。每个流操作符都可以产生一个物化值，用户有责任将它们组合成一个新的类型。在上面的例子中，我们用toMat表示我们要转换源和汇的物化值，我们用方便函数Keep.right表示我们只对汇的物化值感兴趣

在我们的例子中，FoldSink将一个类型为Future的值具体化，它将代表流上折叠过程的结果。一般来说，一个流可以暴露多个物化值，但只对流中的Source或Sink的值感兴趣是很常见的。出于这个原因，有一个名为runWith()的方便方法可用于Sink、Source或Flow，分别需要一个提供的Source(为了运行Sink)、Sink(为了运行Source)或同时需要一个Source和Sink(为了运行Flow，因为它还没有附加)

```scala
val source = Source(1 to 10)
val sink = Sink.fold[Int, Int](0)(_ + _)

// materialize the flow, getting the Sinks materialized value
val sum: Future[Int] = source.runWith(sink)
```

值得指出的是，由于运算符是不可变的，连接它们会返回一个新的运算符，而不是修改现有的实例，所以在构造长流的时候，要记得把新的值赋给一个变量或运行它

```scala
val source = Source(1 to 10)
source.map(_ => 0) // has no effect on source, since it's immutable
source.runWith(Sink.fold(0)(_ + _)) // 55

val zeroes = source.map(_ => 0) // returns new Source[Int], with `map()` appended
zeroes.runWith(Sink.fold(0)(_ + _)) // 0
```

在上面的例子中，我们使用了runWith方法，它既可以将流物质化，也可以返回给定sink或source的物质化值。

由于一个流可以被多次物化，所以每一次这样的物化也会重新计算物化值，通常会导致每次返回不同的值。在下面的例子中，我们创建了两个运行中的流的物化实例，我们在runnable变量中描述了这两个实例。两个物化都给了我们一个与地图不同的Future，尽管我们使用了相同的sink

```scala
// connect the Source to the Sink, obtaining a RunnableGraph
val sink = Sink.fold[Int, Int](0)(_ + _)
val runnable: RunnableGraph[Future[Int]] =
  Source(1 to 10).toMat(sink)(Keep.right)

// get the materialized value of the FoldSink
val sum1: Future[Int] = runnable.run()
val sum2: Future[Int] = runnable.run()

// sum1 and sum2 are different Futures!
```

### Defining sources, sinks and flows

```scala
// Create a source from an Iterable
Source(List(1, 2, 3))

// Create a source from a Future
Source.fromFuture(Future.successful("Hello Streams!"))

// Create a source from a single element
Source.single("only one element")

// an empty source
Source.empty

// Sink that folds over the stream and returns a Future
// of the final result as its materialized value
Sink.fold[Int, Int](0)(_ + _)

// Sink that returns a Future as its materialized value,
// containing the first element of the stream
Sink.head

// A Sink that consumes a stream without doing anything with the elements
Sink.ignore

// A Sink that executes a side-effecting call for every element of the stream
Sink.foreach[String](println(_))
```

```scala
// Explicitly creating and wiring up a Source, Sink and Flow
Source(1 to 6).via(Flow[Int].map(_ * 2)).to(Sink.foreach(println(_)))

// Starting from a Source
val source = Source(1 to 6).map(_ * 2)
source.to(Sink.foreach(println(_)))

// Starting from a Sink
val sink: Sink[Int, NotUsed] = Flow[Int].map(_ * 2).to(Sink.foreach(println(_)))
Source(1 to 6).to(sink)

// Broadcast to a sink inline
val otherSink: Sink[Int, NotUsed] =
  Flow[Int].alsoTo(Sink.foreach(println(_))).to(Sink.ignore)
Source(1 to 6).to(otherSink)
```

### Illegal stream elements

根据Reactive Streams规范（规则2.13），Akka Streams不允许将null作为元素通过流。如果你想模拟一个值的缺失概念，我们建议使用scala.Option或scala.util.Either


## Back-pressure explained

Akka Streams实现了一个由Reactive Streams规范标准化的异步非阻塞背压协议，Akka是该规范的创始成员之一。

该库的用户不必编写任何显式的背压处理代码--它是内置的，并由所有提供的Akka Streams操作符自动处理。然而，可以添加显式的缓冲区操作符，其溢出策略可以影响流的行为。这在复杂的处理图中尤其重要，因为这些图甚至可能包含循环（必须以非常特殊的方式处理，如图循环、有效性和死锁中所解释的）。

背压协议是根据下游Subscriber能够接收和缓冲的元素数量来定义的，称为需求。数据源在Reactive Streams术语中被称为Publisher，在Akka Streams中被实现为Source，保证它发出的元素永远不会超过任何给定Subscriber的接收总需求


Note

Reactive Streams规范用Publisher和Subscriber来定义其协议。这些类型并不意味着是面向用户的API，而是作为不同Reactive Streams实现的低级构件。

Akka Streams将这些概念实现为Source、Flow（在Reactive Streams中被称为Processor）和Sink，而不直接暴露Reactive Streams接口。如果你需要与其他Reactive Stream库集成，请阅读Integrating with Reactive Streams

Reactive Streams的背压工作模式可以通俗地描述为 "动态推/拉模式"，因为它将根据下游是否能够应对上游的生产速度，在基于推和拉的背压模式之间切换。

为了进一步说明这一点，让我们考虑这两种问题情况以及背压协议如何处理它们


### Slow Publisher, fast Subscriber

这是一种幸福的情况--在这种情况下，我们不需要减慢Publisher的速度。然而信令速率很少是恒定的，可能会在任何时间点发生变化，突然间就会出现Subscriber比Publisher慢的情况。为了防止这些情况的发生，在这种情况下，背压协议仍然必须启用，但是我们不希望因为启用了这个安全网而付出高昂的代价。

Reactive Streams协议通过异步地从Subscriber向Publisher发出Request(n:Int)信号来解决这个问题。该协议保证Publisher发出的信号永远不会超过信号需求的元素。然而由于Subscriber目前的速度较快，它将以更高的速率来信号化这些Request消息（也可能会将需求批量化--在一个Request信号中请求多个元素）。这意味着，Publisher应该永远不必等待（被反压）发布其传入的元素。

正如我们所看到的，在这种情况下，我们有效地在所谓的推送模式下操作，因为Publisher可以以最快的速度继续生产元素，因为待定的需求将在它发出元素的同时及时回收


### Fast Publisher, slow Subscriber

当需要对Publisher进行反压时，就会出现这种情况，因为Subscriber无法满足其上游想要发出数据元素的速度。

由于Publisher不允许发出比Subscriber发出的待定需求更多的元素信号，所以它必须采用以下策略之一来遵守这种反压

*  不产生元素，如果能够控制其生产速度
*  试着以有界的方式缓冲元素，直到有更多的需求信号出现
*  下降元素，直到发出更多的需求信号
*  如果无法应用上述任何一种策略，则拆流


## Stream Materialization

当在Akka Streams中构造流和图时，把它们看作是准备一个蓝图，一个执行计划。流的具体化是将一个流描述（RunnableGraph）并分配其运行所需的所有必要资源的过程。在Akka流的情况下，这通常意味着启动为处理提供动力的Actor，但并不限于此--它也可能意味着打开文件或套接字连接等--取决于流的需求。

物化是在所谓的 "终端操作 "时触发的。最值得注意的是，这包括定义在Source和Flow元素上的各种形式的run()和runWith()方法，以及少量用于与著名的sink一起运行的特殊语法，如runForeach(el => ...)（是runWith(Sink.foreach(el => ...)的别名）。

物化是由ActorSystem全局Materializer在物化线程上同步进行的。实际的流处理是由在流材料化过程中启动的actors处理的，它们将在它们被配置为运行的线程池上运行--默认情况下，这些线程池是在ActorSystem配置中设置的dispatcher，或者是作为正在材料化的流的属性提供的


### Operator Fusion

默认情况下，Akka Streams将融合流操作符。这意味着流或流的处理步骤可以在同一个Actor内执行，并且有两个后果

*  由于避免了异步消息传递的开销，从一个操作符到下一个操作符之间传递元素的速度要快很多
*  融合流操作符不会相互并行运行，这意味着每个融合部分最多只使用一个CPU核

为了允许并行处理，你必须在你的流和操作符中手动插入异步边界，方法是在Source、Sink和Flow上添加Attributes.asyncBoundary，操作符应以异步方式与图形的下游通信

`Source(List(1, 2, 3)).map(_ + 1).async.map(_ * 2).to(Sink.ignore)`

在这个例子中，我们在流中创建了两个区域，这两个区域将分别在一个Actor中执行--假设整数的加法和乘法是一个极其昂贵的操作，这将导致性能的提升，因为两个CPU可以并行地处理这些任务。需要注意的是，异步边界并不是流中元素异步传递的单一地方（就像在其他流库中一样），而是属性总是通过将信息添加到已经构建到这一点的流图中来工作

![avatar](img/asyncBoundary.png)

这意味着，红色气泡内的所有内容将由一个行为者执行，而红色气泡外的所有内容则由另一个行为者执行。这个方案可以连续应用，总是有一个这样的边界包围着以前的边界，加上此后增加的所有操作者

### Combining materialized values

由于Akka Streams中的每个操作符在被物化后都可以提供一个物化值，所以当我们把这些操作符插在一起时，有必要以某种方式表达这些值应该如何组成一个最终值。为此，许多操作符方法都有一些变体，它们会接受一个额外的参数，即一个函数，这个函数将被用来组合产生的值。下面的例子中说明了使用这些组合器的一些例子

```scala
// A source that can be signalled explicitly from the outside
val source: Source[Int, Promise[Option[Int]]] = Source.maybe[Int]

// A flow that internally throttles elements to 1/second, and returns a Cancellable
// which can be used to shut down the stream
val flow: Flow[Int, Int, Cancellable] = throttler

// A sink that returns the first element of a stream in the returned Future
val sink: Sink[Int, Future[Int]] = Sink.head[Int]

// By default, the materialized value of the leftmost stage is preserved
val r1: RunnableGraph[Promise[Option[Int]]] = source.via(flow).to(sink)

// Simple selection of materialized values by using Keep.right
val r2: RunnableGraph[Cancellable] = source.viaMat(flow)(Keep.right).to(sink)
val r3: RunnableGraph[Future[Int]] = source.via(flow).toMat(sink)(Keep.right)

// Using runWith will always give the materialized values of the stages added
// by runWith() itself
val r4: Future[Int] = source.via(flow).runWith(sink)
val r5: Promise[Option[Int]] = flow.to(sink).runWith(source)
val r6: (Promise[Option[Int]], Future[Int]) = flow.runWith(source, sink)

// Using more complex combinations
val r7: RunnableGraph[(Promise[Option[Int]], Cancellable)] =
  source.viaMat(flow)(Keep.both).to(sink)

val r8: RunnableGraph[(Promise[Option[Int]], Future[Int])] =
  source.via(flow).toMat(sink)(Keep.both)

val r9: RunnableGraph[((Promise[Option[Int]], Cancellable), Future[Int])] =
  source.viaMat(flow)(Keep.both).toMat(sink)(Keep.both)

val r10: RunnableGraph[(Cancellable, Future[Int])] =
  source.viaMat(flow)(Keep.right).toMat(sink)(Keep.both)

// It is also possible to map over the materialized values. In r9 we had a
// doubly nested pair, but we want to flatten it out
val r11: RunnableGraph[(Promise[Option[Int]], Cancellable, Future[Int])] =
  r9.mapMaterializedValue {
    case ((promise, cancellable), future) =>
      (promise, cancellable, future)
  }

// Now we can use pattern matching to get the resulting materialized values
val (promise, cancellable, future) = r11.run()

// Type inference works as expected
promise.success(None)
cancellable.cancel()
future.map(_ + 3)

// The result of r11 can be also achieved by using the Graph API
val r12: RunnableGraph[(Promise[Option[Int]], Cancellable, Future[Int])] =
  RunnableGraph.fromGraph(GraphDSL.create(source, flow, sink)((_, _, _)) { implicit builder => (src, f, dst) =>
    import GraphDSL.Implicits._
    src ~> f ~> dst
    ClosedShape
  })
```

### Source pre-materialization

在某些情况下，你需要在Source被连接到图形的其他部分之前，先得到一个Source的具体化值。这在 "物化值驱动 "的Source的情况下特别有用，比如Source.queue, Source.actorRef或Source.maybe。

通过在一个Source上使用preMaterialize操作符，你可以获得它的物化值和另一个Source。后者可以用来消耗原Source的消息。请注意，这可以被物化多次

```scala
val matValuePoweredSource =
  Source.actorRef[String](bufferSize = 100, overflowStrategy = OverflowStrategy.fail)

val (actorRef, source) = matValuePoweredSource.preMaterialize()

actorRef ! "Hello!"

// pass source around for materialization
source.runWith(Sink.foreach(println))
```


## Stream ordering

在Akka流中，几乎所有的计算运算符都保留了元素的输入顺序。这意味着，如果输入{IA1,IA2,...,IAN}。"原因 "输出{OA1,OA2,...,OAk}和输入{IB1,IB2,...,IBm}。"cause "输出{OB1,OB2,...,OBl}，所有的IAi发生在所有IBi之前，那么OAi发生在OBi之前。

这个属性甚至被如mapAsync这样的异步操作所维护，然而存在一个无序的版本，叫做mapAsyncUnordered，它不保留这个排序。

然而，在处理多个输入流的 Junctions 中（例如 Merge），一般来说，对于不同输入端口的元素，输出顺序是不定义的。也就是说，一个类似于merge的操作可能会在发出Bi之前发出Ai，而决定发出元素的顺序是由其内部逻辑决定的。然而，像Zip这样的特殊元素确实保证了它们的输出顺序，因为每个输出元素都取决于所有上游元素已经被信号化--因此在Zipping的情况下，顺序是由这个属性定义的。

如果你发现自己在扇形图中需要对输出元素的顺序进行精细控制，可以考虑使用MergePreferred、MergePrioritized或GraphStage--它能让你完全控制合并的执行方式


## Actor Materializer Lifecycle

Materializer是一个组件，它负责将流蓝图转化为运行的流，并发出 "物质化的值"。ActorSystem范围内的Materializer是由Akka Extension SystemMaterializer提供的，通过在范围内有一个隐含的ActorSystem，这种方式不需要担心Materializer，除非有特殊要求。

可能需要自定义Materializer实例的用例是，当所有在Actor中物化的流应该与Actor生命周期绑定，并在Actor停止或崩溃时停止。

处理流和Actor的一个重要方面是理解Materializer的生命周期。Materializer被绑定到它所创建的ActorRefFactory的生命周期，在实践中，这个ActorRefFactory将是一个ActorSystem或ActorContext（当Materializer被创建在一个Actor中时）。

将它绑定到ActorSystem上应该被替换为使用Akka 2.6以上的系统材质器。

当由系统材质器运行时，流会一直运行到ActorSystem被关闭。当材质器在流媒体运行完成之前被关闭时，它们将被突然终止。这与通常的终止流的方式有些不同，通常的方式是取消/完成流。流的生命周期是这样绑定到物化器上的，以防止泄漏，在正常的操作中，你不应该依赖这种机制，而应该使用KillSwitch或正常的完成信号来管理流的生命周期。

如果我们看下面的例子，我们在一个Actor中创建Materializer

```scala
final class RunWithMyself extends Actor {
  implicit val mat = Materializer(context)

  Source.maybe.runWith(Sink.onComplete {
    case Success(done) => println(s"Completed: $done")
    case Failure(ex)   => println(s"Failed: ${ex.getMessage}")
  })

  def receive = {
    case "boom" =>
      context.stop(self) // will also terminate the stream
  }
}
```

在上面的例子中，我们使用ActorContext来创建材质器。这就把它的生命周期绑定到了周围的Actor上。换句话说，虽然我们在那里启动的流在正常情况下会永远运行，但如果我们停止Actor，它也会终止该流。我们已经将流的生命周期与周围Actor的生命周期进行了绑定。如果流与Actor密切相关，这是一个非常有用的技术，例如，当Actor代表一个用户或其他实体时，我们使用创建的流持续查询--当Actor已经终止时，保持流的生命力是没有意义的。流的终止将由流发出 "突然终止异常 "的信号。

你也可以通过显式调用shutdown()来导致一个Materializer关闭，结果是突然终止它当时运行的所有流。

然而，有时你可能想显式地创建一个流，这个流会比actor的寿命长。例如，你正在使用Akka流将一些大型数据流推送给外部服务。你可能想急切地停止Actor，因为它已经履行了所有的职责

```scala
final class RunForever(implicit val mat: Materializer) extends Actor {

  Source.maybe.runWith(Sink.onComplete {
    case Success(done) => println(s"Completed: $done")
    case Failure(ex)   => println(s"Failed: ${ex.getMessage}")
  })

  def receive = {
    case "boom" =>
      context.stop(self) // will NOT terminate the stream (it's bound to the system!)
  }
}
```




