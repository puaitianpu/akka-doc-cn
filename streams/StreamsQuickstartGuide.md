# Streams Quickstart Guide

要使用Akka Streams, 添加以下依赖:

```scala
val AkkaVersion = "2.6.8"
libraryDependencies += "com.typesafe.akka" %% "akka-stream" % AkkaVersion
```


## First steps

一个 stream 通常是从一个 source 开始的

import: 

```scala
import akka.stream._
import akka.stream.scaladsl._
```

```scala
import akka.{ Done, NotUsed }
import akka.actor.ActorSystem
import akka.util.ByteString
import scala.concurrent._
import scala.concurrent.duration._
import java.nio.file.Paths
```

implicit ActorSystem

```scala
object Main extends App {
  implicit val system = ActorSystem("QuickStart")
  // Code here
}
```

一个简单的流， 数字从 1 到 100：

```scala
val source: Source[Int, NotUsed] = Source(1 to 100)
```

Source类型的参数化有两种类型：第一种是该源发出的元素类型，第二种是 "materialized value"，允许运行该源产生一些辅助值（例如网络源可以提供绑定端口或对等体地址的信息）。在没有产生辅助信息的情况下，则使用akka.NotUsed类型。一个简单的整数范围就属于这一类--运行我们的流会产生一个NotUsed。

在创建了这个源之后，意味着我们有了如何发出前100个自然数的描述，但这个源还没有被激活。为了得到这些数字，我们必须运行它

```scala
source.runForeach(i => println(i))
```

runForeach在流完成时会返回一个Future[Done]

```scala
val done: Future[Done] = source.runForeach(i => println(i))

implicit val ec = system.dispatcher
done.onComplete(_ => system.terminate())
```

Akka Streams 的好处是， Source 是对想要运行的东西的描述，可以重复使用，融入到一个更大的设计中。

```scala
val factorials = source.scan(BigInt(1))((acc, next) => acc * next)

val result: Future[IOResult] =
  factorials.map(num => ByteString(s"$num\n")).runWith(FileIO.toPath(Paths.get("factorials.txt")))
```


### Browser-embedded example

```scala
import akka.NotUsed
import akka.actor.ActorSystem
import akka.stream.scaladsl._

final case class Author(handle: String)

final case class Hashtag(name: String)

final case class Tweet(author: Author, timestamp: Long, body: String) {
  def hashtags: Set[Hashtag] =
    body
      .split(" ")
      .collect {
        case t if t.startsWith("#") => Hashtag(t.replaceAll("[^#\\w]", ""))
      }
      .toSet
}

val akkaTag = Hashtag("#akka")

val tweets: Source[Tweet, NotUsed] = Source(
  Tweet(Author("rolandkuhn"), System.currentTimeMillis, "#akka rocks!") ::
  Tweet(Author("patriknw"), System.currentTimeMillis, "#akka !") ::
  Tweet(Author("bantonsson"), System.currentTimeMillis, "#akka !") ::
  Tweet(Author("drewhk"), System.currentTimeMillis, "#akka !") ::
  Tweet(Author("ktosopl"), System.currentTimeMillis, "#akka on the rocks!") ::
  Tweet(Author("mmartynas"), System.currentTimeMillis, "wow #akka !") ::
  Tweet(Author("akkateam"), System.currentTimeMillis, "#akka rocks!") ::
  Tweet(Author("bananaman"), System.currentTimeMillis, "#bananas rock!") ::
  Tweet(Author("appleman"), System.currentTimeMillis, "#apples rock!") ::
  Tweet(Author("drama"), System.currentTimeMillis, "we compared #apples to #oranges!") ::
  Nil)

  implicit val system = ActorSystem("reactive-tweets")

  tweets
    .map(_.hashtags) // Get all sets of hashtags ...
    .reduce(_ ++ _) // ... and reduce them to a single set, removing duplicates across all tweets
    .mapConcat(identity) // Flatten the set of hashtags to a stream of hashtags
    .map(_.name.toUpperCase) // Convert all hashtags to upper case
    .runWith(Sink.foreach(println)) // Attach the Flow to a Sink that will finally print the hashtags
```


## Reusable Pieces

Akka Streams的一个很好的部分--也是其他流库没有提供的--就是不仅源可以像蓝图一样重用，其他所有元素也可以重用。我们可以把写文件的Sink，预置必要的处理步骤，从传入的字符串中获取ByteString元素，并把它也包装成一个可重用的作品。由于编写这些流的语言总是从左到右流动的（就像纯英语一样），我们需要一个像源码一样的起点，但有一个 "开放 "的输入。在Akka Streams中，这叫做流

```scala
def lineSink(filename: String): Sink[String, Future[IOResult]] =
  Flow[String].map(s => ByteString(s + "\n")).toMat(FileIO.toPath(Paths.get(filename)))(Keep.right)
```

从字符串流开始，我们将每一个字符串转换为ByteString，然后馈送给已经知道的文件写入Sink.由此产生的蓝图是一个Sink[String, Future[IOResult]]，这意味着接受字符串的输入，当物质化时，将创建类型为Future[IOResult]的辅助信息。由此产生的蓝图是一个Sink[String，Future[IOResult]]，这意味着它接受字符串作为它的输入，当物化时，它将创建类型为Future[IOResult]的辅助信息（当对Source或Flow进行链式操作时，辅助信息的类型称为 "物化值"--由最左边的起点给出；由于我们要保留FileIO.toPath sink所提供的功能，我们需要说Keep.right）。

我们可以使用我们刚刚创建的新的闪亮的Sink，将它附加到我们的factorials源--经过一个小的调整，将数字变成字符串

`factorials.map(_.toString).runWith(lineSink("factorial2.txt"))`


## Time-Based Processing

在开始看一个更复杂的例子之前，我们先来探讨一下Akka Streams可以做的流式性质。从factorials源开始，我们通过将流与另一个流压缩在一起进行转换，这个流由一个发出0到100的数字的Source表示：factorials源发出的第一个数字是0的阶乘，第二个是1的阶乘，以此类推。我们将这两者结合起来，形成像 "3! = 6"

```scala
factorials
  .zipWith(Source(0 to 100))((num, idx) => s"$idx! = $num")
  .throttle(1, 1.second)
  .runForeach(println)
```

到目前为止，所有的操作都是与时间无关的，并且可以以同样的方式在严格的元素集合上执行。下一行表明，我们实际上是在处理可以以一定速度流动的流：我们使用 `throttle` 将流减速到每秒1个元素。

如果你运行这个程序，你会看到每秒打印一行。不过，有一个不是马上就能看到的方面值得一提：如果你尝试着把流设置成每个产生10亿个数字，那么你会注意到你的JVM不会因为OutOfMemoryError而崩溃，尽管你也会注意到运行流是在后台异步发生的（这就是辅助信息作为Future提供的原因，以后再说）。使这一工作的秘密是Akka流隐式地实现了无孔不入的流控制，所有的操作符都尊重背压。这使得节流操作者可以向其所有上游数据源发出信号，表明它只能以一定的速率接受元素--当传入速率高于每秒一个时，节流操作者将向上游断言背压。

这就是Akka Streams的全部内容--轻描淡写的说，有几十个源和汇，还有很多流转换操作符可以选择，也可以参考操作符索引



## Reactive Tweets

流处理的一个典型用例是消耗一个实时数据流，我们想从中提取或聚合一些其他数据。在这个例子中，我们将考虑消耗一个推文流，并从中提取有关Akka的信息。

我们还将考虑所有非阻塞流解决方案所固有的问题。"如果用户的速度太慢，无法消耗实时数据流怎么办？"。传统的解决方案往往是对元素进行缓冲，但这可能--而且通常会导致最终的缓冲区溢出和这类系统的不稳定。相反，Akka Streams依赖于内部背压信号，允许控制在这种情况下应该发生什么

```scala
final case class Author(handle: String)

final case class Hashtag(name: String)

final case class Tweet(author: Author, timestamp: Long, body: String) {
  def hashtags: Set[Hashtag] =
    body
      .split(" ")
      .collect {
        case t if t.startsWith("#") => Hashtag(t.replaceAll("[^#\\w]", ""))
      }
      .toSet
}

val akkaTag = Hashtag("#akka")
```

## Transforming and consuming simple streams

我们将看到的示例应用程序是一个简单的Twitter feed流，我们希望从中提取某些信息，例如找到所有关于#akka的twitter用户的twitter把手。

为了准备我们的环境，我们要创建一个ActorSystem，它将负责运行我们将要创建的流

`implicit val system = ActorSystem("reactive-tweets")`

让我们假设我们有一个现成的推文流。在Akka中，这表示为Source[Out，M]

`val tweets: Source[Tweet, NotUsed]`

Streams 是从 Source[Out, M1] 开始的。然后元素经过Flow[In, Out, M2]到最后被 Sink[In, M3] 消耗

对于使用过Scala Collections库的人来说，这些操作应该很熟悉，但是它们操作的是流而不是数据的集合（这是一个非常重要的区别，因为有些操作只有在流中才有意义，反之亦然）

```scala
val authors: Source[Author, NotUsed] =
  tweets.filter(_.hashtags.contains(akkaTag)).map(_.author)
```

最后，为了实现和运行流计算，我们需要将Flow附加到一个Sink上，让Flow运行。最简单的方法是在Source上调用runWith(sink)。为了方便，我们预定义了一些常用的Sink，并作为Sink同伴对象上的方法收集起来

`authors.runWith(Sink.foreach(println))`

`authors.runForeach(println)`

Materializing and running 需要 `implicit val system = ActorSystem("name")` 或者 显示传递: `.runWith(sink)(system)`

```scala
implicit val system = ActorSystem("reactive-tweets")

val authors: Source[Author, NotUsed] =
  tweets.filter(_.hashtags.contains(akkaTag)).map(_.author)

authors.runWith(Sink.foreach(println))
``` 


## Flattening sequences in streams

在上一节中，我们处理的是元素的1:1关系，这是最常见的情况，但有时我们可能想从一个元素映射到多个元素，并接收一个 "扁平化 "的流，类似于flatMap在Scala Collections上的工作。为了从我们的tweets流中得到一个扁平化的标签流，我们可以使用mapConcat操作符

`val hashtags: Source[Hashtag, NotUsed] = tweets.mapConcat(_.hashtags.toList)`


由于flatMap与for-comprehensions和monadic composition很接近，所以有意识地避免使用这个名字。它存在问题的原因有两个：首先，由于存在死锁的风险，在有界流处理中，通过连通的方式进行平坦化处理通常是不可取的（合并是首选策略），其次，单项法则对于我们的平坦地图的实现是不成立的（由于有效性问题）。

请注意，mapConcat要求提供的函数返回一个可迭代的（f: Out => immutable.Iterable[T]，而flatMap则必须对流进行一路操作


## Broadcasting a stream

在 Akka Streams 中，Elements 使用 “fan-out” or "fan-in" 这种结构叫做 "junctions"

`Broadcast` 将元素从输入端口发送到所有的输出端口

Akka Streams有意将线性流结构（Flows）和非线性、分支结构（Graphs）分开，以便为这两种情况提供最方便的API。Graphs可以表达任意复杂的流设置，代价是读起来不像集合变换那么熟悉。

使用 GraphDSL 构建 Graphs: 

```scala
val writeAuthors: Sink[Author, NotUsed] = ???
val writeHashtags: Sink[Hashtag, NotUsed] = ???
val g = RunnableGraph.fromGraph(GraphDSL.create() { implicit b =>
  import GraphDSL.Implicits._

  val bcast = b.add(Broadcast[Tweet](2))
  tweets ~> bcast.in
  bcast.out(0) ~> Flow[Tweet].map(_.author) ~> writeAuthors
  bcast.out(1) ~> Flow[Tweet].mapConcat(_.hashtags.toList) ~> writeHashtags
  ClosedShape
})
g.run()
```

在 `GraphDSL` 中使用 implicit graph builder `b` 来构建一个图结构， `~>` 边缘操作符("connect" or "via" or "to") 来连接图形结构
使用这个操作符需要 `import GraphDSL.Implicits._`

`GraphDSL.create`  returns a Graph(图)， `ClosedShape` 代表这个图是一个完全连接的图形或者说是"closed(封闭的)"。没有输入和输出。
RunnableGraph.fromGraph 说明 图是可以运行的， 通过调用 `run` 方法执行

Graph和RunnableGraph都是不可变的、线程安全的、可自由共享的

## Back-pressure in action

Akka流的主要优势之一是它们总是将背压信息从流汇（Subscribers）传播到它们的源（Publishers）。这不是一个可选的功能，并且在任何时候都是启用的

像这样的应用（不使用Akka Streams）经常面临的一个典型问题是，它们无法足够快地处理传入的数据，无论是暂时的还是设计的，它们会开始缓冲传入的数据，直到没有更多的空间来缓冲，从而导致OutOfMemoryError s或其他服务响应能力的严重下降。使用Akka Streams，缓冲可以而且必须明确地处理。例如，如果我们只对 "最近的推文感兴趣，缓冲区为10个元素"，可以用缓冲区元素来表示
`tweets.buffer(10, OverflowStrategy.dropHead).map(slowComputation).runWith(Sink.ignore)`

缓冲区元素需要一个显式的和必需的OverflowStrategy，它定义了当缓冲区满了的时候收到另一个元素时应该如何反应。提供的策略包括丢弃最老的元素（dropHead）、丢弃整个缓冲区、信号错误等。一定要选择最适合你的用例的策略


## Materialized values

到目前为止，我们只是使用Flows处理数据，并将其消耗到某种外部Sink中--无论是通过打印值还是将其存储在某个外部系统中。然而有时候我们可能会对一些可以从物化处理管道中获得的值感兴趣。例如，我们想知道我们处理了多少条推文。虽然这个问题在无限的推文流的情况下，给出的答案并不那么明显（在流式环境中回答这个问题的一种方法是创建一个计数流，描述为 "到目前为止，我们已经处理了N条推文"），但一般来说，处理有限的流，并得出一个不错的结果，比如元素的总计数


首先，我们用Sink.fold来写这样一个元素计数器

```scala
val count: Flow[Tweet, Int, NotUsed] = Flow[Tweet].map(_ => 1)

val sumSink: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)

val counterGraph: RunnableGraph[Future[Int]] =
  tweets.via(count).toMat(sumSink)(Keep.right)

val sum: Future[Int] = counterGraph.run()

sum.foreach(c => println(s"Total tweets processed: $c"))
```

首先，我们准备一个可重用的Flow，它将把每个传入的tweet变为一个值为1的整数。我们将使用它来将这些与Sink.fold结合起来，Sink.fold将对流中的所有Int元素进行求和，并将其结果作为Future[Int]提供。接下来我们用via连接微博流来计数。最后我们用toMat将Flow连接到之前准备好的Sink上。

还记得Source[+Out，+Mat]、Flow[-In，+Out，+Mat]和Sink[-In，+Mat]上那些神秘的Mat类型参数吗？它们代表了这些处理部件在实体化时返回的值的类型。当你把这些链在一起时，你可以显式地组合它们的物化值。在我们的例子中，我们使用了Keep.right预定义函数，它告诉实现只关心当前附加在右边的运算符的物化类型。sumSink的物化类型是Future[Int]，由于使用了Keep.right，生成的RunnableGraph的类型参数也是Future[Int]。

这一步还没有将处理管道具体化，它只是准备了Flow的描述，现在Flow已经连接到Sink上，因此可以进行Runnable()，这一点由其类型表示。RunnableGraph[Future[Int]]。接下来我们调用run()，它将Flow具体化并运行。在RunnableGraph[T]上调用run()返回的值是T类型。在我们的例子中，这个类型是Future[Int]，当完成后，它将包含我们的tweets流的总长度。在流失败的情况下，这个future会以Failure来完成

一个RunnableGraph可以被重复使用和多次物化，因为它只是流的 "蓝图"。这意味着，如果我们将一个流进行物化，例如一个在一分钟内消耗了一个实时推文流的流，这两个物化的值将是不同的，如这个例子所示：

```scala
val sumSink = Sink.fold[Int, Int](0)(_ + _)
val counterRunnableGraph: RunnableGraph[Future[Int]] =
  tweetsInMinuteFromNow.filter(_.hashtags contains akkaTag).map(t => 1).toMat(sumSink)(Keep.right)

// materialize the stream once in the morning
val morningTweetsCount: Future[Int] = counterRunnableGraph.run()
// and once in the evening, reusing the flow
val eveningTweetsCount: Future[Int] = counterRunnableGraph.run()
```

Akka Streams中的许多元素都提供了物化值，这些物化值可以用来获得计算结果，也可以用来引导这些元

`val sum: Future[Int] = tweets.map(t => 1).runWith(sumSink)`


### Note

runWith()是一个方便的方法，它能自动忽略除了runWith()本身附加的操作符以外的任何其他操作符的物化值。在上面的例子中，它转化为使用Keep.right作为物化值的组合器


