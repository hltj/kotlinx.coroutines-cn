<!--- TEST_NAME ChannelsGuideTest -->

**目录**

<!--- TOC -->

* [通道](#通道)
  * [通道基础](#通道基础)
  * [关闭与迭代通道](#关闭与迭代通道)
  * [构建通道生产者](#构建通道生产者)
  * [管道](#管道)
  * [使用管道的素数](#使用管道的素数)
  * [扇出](#扇出)
  * [扇入](#扇入)
  * [带缓冲的通道](#带缓冲的通道)
  * [通道是公平的](#通道是公平的)
  * [计时器通道](#计时器通道)

<!--- END -->

## 通道

延期的值提供了一种便捷的方法使单个值在多个协程之间进行相互传输。
通道提供了一种在流中传输值的方法。

### 通道基础

一个 [Channel] 是一个和 `BlockingQueue` 非常相似的概念。其中一个不同是<!--
-->它代替了阻塞的 `put` 操作并提供了挂起的 [send][SendChannel.send]，还替代了<!--
-->阻塞的 `take` 操作并提供了挂起的 [receive][ReceiveChannel.receive]。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        // 这里可能是消耗大量 CPU 运算的异步逻辑，我们将仅仅做 5 次整数的平方并发送
        for (x in 1..5) channel.send(x * x)
    }
    // 这里我们打印了 5 次被接收的整数：
    repeat(5) { println(channel.receive()) }
    println("Done!")
//sampleEnd
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-01.kt)获取完整代码。

这段代码的输出如下：

```text
1
4
9
16
25
Done!
```

<!--- TEST -->

### 关闭与迭代通道

和队列不同，一个通道可以通过被关闭来表明没有更多的元素将会进入通道。
在接收者中可以定期的使用 `for` 循环来从通道中<!--
-->接收元素。
 
从概念上来说，一个 [close][SendChannel.close] 操作就像向通道发送了一个特殊的关闭指令。
这个迭代停止就说明关闭指令已经被接收了。所以这里保证<!--
-->所有先前发送出去的元素都在通道关闭前被接收到。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // 我们结束发送
    }
    // 这里我们使用 `for` 循环来打印所有被接收到的元素（直到通道被关闭）
    for (y in channel) println(y)
    println("Done!")
//sampleEnd
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-02.kt)获取完整代码。

<!--- TEST 
1
4
9
16
25
Done!
-->

### 构建通道生产者

协程生成一系列元素的模式很常见。
这是 _生产者——消费者_ 模式的一部分，并且经常能在并发的代码中看到它。
你可以将生产者抽象成一个函数，并且使通道作为它的参数，但这与<!--
-->必须从函数中返回结果的常识相违悖。

这里有一个名为 [produce] 的便捷的协程构建器，可以很容易的在生产者端正确工作，
并且我们使用扩展函数 [consumeEach] 在消费者端替代 `for` 循环：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
//sampleStart
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
//sampleEnd
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-03.kt)获取完整代码。

<!--- TEST 
1
4
9
16
25
Done!
-->

### 管道

管道是一种一个协程在流中开始生产可能无穷多个元素的模式：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // 在流中开始从 1 生产无穷多个整数
}
```

</div>

并且另一个或多个协程开始消费这些流，做一些操作，并生产了一些额外的结果。
在下面的例子中，对这些数字仅仅做了平方操作：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

</div>

主要的代码启动并连接了整个管道：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val numbers = produceNumbers() // 从 1 开始生成整数
    val squares = square(numbers) // 整数求平方
    repeat(5) {
        println(squares.receive()) // 输出前五个
    }
    println("Done!") // 至此已完成
    coroutineContext.cancelChildren() // 取消子协程
//sampleEnd
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // 从 1 开始的无限的整数流
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-04.kt)获取完整代码。

<!--- TEST 
1
4
9
16
25
Done!
-->

> 所有创建了协程的函数被定义在了 [CoroutineScope] 的扩展上，
所以我们可以依靠[结构化并发](https://kotlinlang.org/docs/reference/coroutines/composing-suspending-functions.html#structured-concurrency-with-async)来确保<!--
-->没有常驻在我们的应用程序中的全局协程。

### 使用管道的素数

让我们来展示一个极端的例子——在协程中使用一个管道来生成<!--
-->素数。我们开启了一个数字的无限序列。

<div class="sample" markdown="1" theme="idea" data-highlight-only>
 
```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // 开启了一个无限的整数流
}
```

</div>

在下面的管道阶段中过滤了来源于流中的数字，删除了所有<!--
-->可以被给定素数整除的数字。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

</div>

现在我们开启了一个从 2 开始的数字流管道，从当前的通道中取一个素数，
并为每一个我们发现的素数启动一个流水线阶段：
 
```
numbersFrom(2) -> filter(2) -> filter(3) -> filter(5) -> filter(7) ……
``` 
 
下面的例子打印了前十个素数，
在主线程的上下文中运行整个管道。直到所有的协程在<!--
-->该主协程 [runBlocking] 的作用域中被启动完成。
我们不必使用一个显式的列表来保存所有被我们已经启动的协程。
我们使用 [cancelChildren][kotlin.coroutines.CoroutineContext.cancelChildren]
扩展函数在我们打印了前十个素数以后<!--
-->来取消所有的子协程。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    var cur = numbersFrom(2)
    repeat(10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // 取消所有的子协程来让主协程结束
//sampleEnd    
}

fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // 从 start 开始过滤整数流
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-05.kt)获取完整代码。

这段代码的输出如下：

```text
2
3
5
7
11
13
17
19
23
29
```

<!--- TEST -->

注意，你可以在标准库中使用
[`iterator`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/iterator.html)
协程构建器来构建一个相似的管道。
使用 `iterator` 替换 `produce`、`yield` 替换 `send`、`next` 替换 `receive`、
`Iterator` 替换 `ReceiveChannel` 来摆脱协程作用域，你将不再需要 `runBlocking`。
然而，如上所示，如果你在 [Dispatchers.Default] 上下文中运行它，使用通道的管道的好处在于<!--
-->它可以充分利用多核心 CPU。

不过，这是一种非常不切实际的寻找素数的方法。在实践中，管道调用了另外的一些<!--
-->挂起中的调用（就像异步调用远程服务）并且这些管道不能<!--
-->内置使用 `sequence`/`iterator`，因为它们不被允许随意的挂起，不像
`produce` 是完全异步的。
 
### 扇出

多个协程也许会接收相同的管道，在它们之间进行分布式工作。
让我们启动一个定期产生整数的生产者协程
（每秒十个数字）：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // 从 1 开始
    while (true) {
        send(x++) // 产生下一个数字
        delay(100) // 等待 0.1 秒
    }
}
```

</div>

接下来我们可以得到几个处理器协程。在这个示例中，它们只是打印它们的 id 和<!--
-->接收到的数字：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

现在让我们启动五个处理器协程并让它们工作将近一秒。看看发生了什么：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950)
    producer.cancel() // 取消协程生产者从而将它们全部杀死
//sampleEnd
}

fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // 产生下一个数字
        delay(100) // 等待 0.1 秒
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-06.kt)获取完整代码。

该输出将类似于如下所示，尽管接收<!--
-->每个特定整数的处理器 id 可能会不同：

```
Processor #2 received 1
Processor #4 received 2
Processor #0 received 3
Processor #1 received 4
Processor #3 received 5
Processor #2 received 6
Processor #4 received 7
Processor #0 received 8
Processor #1 received 9
Processor #3 received 10
```

<!--- TEST lines.size == 10 && lines.withIndex().all { (i, line) -> line.startsWith("Processor #") && line.endsWith(" received ${i + 1}") } -->

注意，取消生产者协程将关闭它的通道，从而最终终止<!--
-->处理器协程正在执行的此通道上的迭代。

还有，注意我们如何使用 `for` 循环显式迭代通道以在 `launchProcessor` 代码中执行扇出。
与 `consumeEach` 不同，这个 `for` 循环是安全完美地使用多个协程的。如果其中一个处理器<!--
-->协程执行失败，其它的处理器协程仍然会继续处理通道，而通过 `consumeEach`
编写的处理器始终在正常或非正常完成时消耗（取消）底层通道。

### 扇入

多个协程可以发送到同一个通道。
比如说，让我们创建一个字符串的通道，和一个在这个通道中<!--
-->以指定的延迟反复发送一个指定字符串的挂起函数：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

</div>

现在，我们启动了几个发送字符串的协程，让我们看看会发生什么
（在示例中，我们在主线程的上下文中作为主协程的子协程来启动它们）：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
//sampleStart
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // 接收前六个
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // 取消所有子协程来让主协程结束
//sampleEnd
}

suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-07.kt)获取完整代码。

输出如下：

```text
foo
foo
BAR!
foo
foo
BAR!
```

<!--- TEST -->

### 带缓冲的通道

到目前为止展示的通道都是没有缓冲区的。无缓冲的通道在发送者和接收者相遇时<!--
-->传输元素（也称“对接”）。如果发送先被调用，则它将被挂起直到接收被调用，
如果接收先被调用，它将被挂起直到发送被调用。

[Channel()] 工厂函数与 [produce] 建造器通过一个可选的参数 `capacity`
来指定 _缓冲区大小_ 。缓冲允许发送者在被挂起前发送多个元素，
就像 `BlockingQueue` 有指定的容量一样，当缓冲区被占满的时候将会引起阻塞。

看看如下代码的表现：


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
//sampleStart
    val channel = Channel<Int>(4) // 启动带缓冲的通道
    val sender = launch { // 启动发送者协程
        repeat(10) {
            println("Sending $it") // 在每一个元素发送前打印它们
            channel.send(it) // 将在缓冲区被占满时挂起
        }
    }
    // 没有接收到东西……只是等待……
    delay(1000)
    sender.cancel() // 取消发送者协程
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-08.kt)获取完整代码。

使用缓冲通道并给 capacity 参数传入 _四_ 它将打印“sending” _五_ 次：

```text
Sending 0
Sending 1
Sending 2
Sending 3
Sending 4
```

<!--- TEST -->

前四个元素被加入到了缓冲区并且发送者在试图发送第五个元素的时候被挂起。

### 通道是公平的

发送和接收操作是 _公平的_ 并且尊重调用它们的<!--
-->多个协程。它们遵守先进先出原则，可以看到第一个协程调用 `receive`
并得到了元素。在下面的例子中两个协程“乒”和“乓”都<!--
-->从共享的“桌子”通道接收到这个“球”元素。


<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

//sampleStart
data class Ball(var hits: Int)

fun main() = runBlocking {
    val table = Channel<Ball>() // 一个共享的 table（桌子）
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // 乒乓球
    delay(1000) // 延迟 1 秒钟
    coroutineContext.cancelChildren() // 游戏结束，取消它们
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // 在循环中接收球
        ball.hits++
        println("$name $ball")
        delay(300) // 等待一段时间
        table.send(ball) // 将球发送回去
    }
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-09.kt)得到完整代码

“乒”协程首先被启动，所以它首先接收到了球。甚至虽然“乒”
协程在将球发送会桌子以后立即开始接收，但是球还是被“乓”
协程接收了，因为它一直在等待着接收球：

```text
ping Ball(hits=1)
pong Ball(hits=2)
ping Ball(hits=3)
pong Ball(hits=4)
```

<!--- TEST -->

注意，有时候通道执行时由于执行者的性质而看起来<!--
-->不那么公平。点击[这个提案](https://github.com/Kotlin/kotlinx.coroutines/issues/111)来查看更多细节。

### 计时器通道

计时器通道是一种特别的会合通道，每次经过特定的延迟都会从该通道进行消费并产生 `Unit`。
虽然它看起来似乎没用，它被用来构建分段来创建复杂的基于时间的 [produce]
管道和进行窗口化操作以及其它时间相关的处理。
可以在 [select] 中使用计时器通道来进行“打勾”操作。

使用工厂方法 [ticker] 来创建这些通道。
为了表明不需要其它元素，请使用 [ReceiveChannel.cancel] 方法。

现在让我们看看它是如何在实践中工作的：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0) //创建计时器通道
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // no initial delay

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() } // all subsequent elements have 100ms delay
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    // 模拟大量消费延迟
    println("Consumer pauses for 150ms")
    delay(150)
    // 下一个元素立即可用
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // 请注意，`receive` 调用之间的暂停被考虑在内，下一个元素的到达速度更快
    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() } 
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel() // 表明不再需要更多的元素
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-channel-10.kt)获取完整代码。

它的打印如下：

```text
Initial element is available immediately: kotlin.Unit
Next element is not ready in 50 ms: null
Next element is ready in 100 ms: kotlin.Unit
Consumer pauses for 150ms
Next element is available immediately after large consumer delay: kotlin.Unit
Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit
```

<!--- TEST -->

请注意，[ticker] 知道可能的消费者暂停，并且默认情况下会调整下一个生成的元素<!--
-->如果发生暂停则延迟，试图保持固定的生成元素率。
 
给可选的 `mode` 参数传入 [TickerMode.FIXED_DELAY] 可以保持固定<!--
-->元素之间的延迟。


<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[kotlin.coroutines.CoroutineContext.cancelChildren]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/kotlin.coroutines.-coroutine-context/cancel-children.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
<!--- INDEX kotlinx.coroutines.channels -->
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[SendChannel.send]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/send.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
[SendChannel.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/close.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[consumeEach]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/consume-each.html
[Channel()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel.html
[ticker]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/ticker.html
[ReceiveChannel.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/cancel.html
[TickerMode.FIXED_DELAY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-ticker-mode/-f-i-x-e-d_-d-e-l-a-y.html
<!--- INDEX kotlinx.coroutines.selects -->
[select]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.selects/select.html
<!--- END -->
