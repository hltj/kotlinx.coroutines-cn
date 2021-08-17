<!--- TEST_NAME SharedStateGuideTest -->
 
**目录**

<!--- TOC -->

* [共享的可变状态与并发](#共享的可变状态与并发)
  * [问题](#问题)
  * [volatile 无济于事](#volatile-无济于事)
  * [线程安全的数据结构](#线程安全的数据结构)
  * [以细粒度限制线程](#以细粒度限制线程)
  * [以粗粒度限制线程](#以粗粒度限制线程)
  * [互斥](#互斥)
  * [Actors](#actors)

<!--- END -->

## 共享的可变状态与并发

协程可用多线程调度器（比如默认的 [Dispatchers.Default]）并发执行。这样就可以提出<!--
-->所有常见的并发问题。主要的问题是同步访问**共享的可变状态**。
协程领域对这个问题的一些解决方案类似于多线程领域中的解决方案，
但其它解决方案则是独一无二的。

### 问题

我们启动一百个协程，它们都做一千次相同的操作。<!--
-->我们同时会测量它们的完成时间以便进一步的比较：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

</div>

我们从一个非常简单的动作开始：使用<!--
-->多线程的 [Dispatchers.Default] 来递增一个共享的可变变量。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-01.kt)获取完整代码。

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码最后打印出什么结果？它不太可能打印出“Counter = 100000”，因为一百个协程<!--
-->在多个线程中同时递增计数器但没有做并发处理。

### volatile 无济于事

有一种常见的误解：volatile 可以解决并发问题。让我们尝试一下：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
@Volatile // 在 Kotlin 中 `volatile` 是一个注解
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-02.kt)获取完整代码。

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码运行速度更慢了，但我们最后仍然没有得到“Counter = 100000”这个结果，因为 volatile 变量保证<!--
-->可线性化（这是“原子”的技术术语）读取和写入变量，但<!--
-->在大量动作（在我们的示例中即“递增”操作）发生时并不提供原子性。

### 线程安全的数据结构

一种对线程、协程都有效的常规解决方法，就是使用线程安全（也称为同步的、
可线性化、原子）的数据结构，它为需要在共享状态上执行的相应<!--
-->操作提供所有必需的同步处理。<!--
-->在简单的计数器场景中，我们可以使用具有 `incrementAndGet` 原子操作的 `AtomicInteger` 类：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-03.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这是针对此类特定问题的最快解决方案。它适用于普通计数器、集合、队列和其他<!--
-->标准数据结构以及它们的基本操作。然而，它并不容易被扩展来应对复杂<!--
-->状态、或一些没有现成的线程安全实现的复杂操作。

### 以细粒度限制线程

 _限制线程_ 是解决共享可变状态问题的一种方案：对特定共享<!--
-->状态的所有访问权都限制在单个线程中。它通常应用于 UI 程序中：所有 UI 状态都局限于<!--
-->单个事件分发线程或应用主线程中。这在协程中很容易实现，通过使用一个<!--
-->单线程上下文：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // 将每次自增限制在单线程上下文中
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-04.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行非常缓慢，因为它进行了 _细粒度_ 的线程限制。每个增量操作都得使用
[withContext(counterContext)] 块<!--
-->从多线程 [Dispatchers.Default] 上下文切换到单线程上下文。

### 以粗粒度限制线程

在实践中，线程限制是在大段代码中执行的，例如：状态更新类业务逻辑中大部分<!--
-->都是限于单线程中。下面的示例演示了这种情况，
在单线程上下文中运行每个协程。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    // 将一切都限制在单线程上下文中
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-05.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行更快而且打印出了正确的结果。

### 互斥

该问题的互斥解决方案：使用永远不会同时执行的 _关键代码块_
来保护共享状态的所有修改。在阻塞的世界中，你通常会为此目的使用 `synchronized` 或者 `ReentrantLock`。
在协程中的替代品叫做 [Mutex] 。它具有 [lock][Mutex.lock] 和 [unlock][Mutex.unlock] 方法，
可以隔离关键的部分。关键的区别在于 `Mutex.lock()` 是一个挂起函数，它不会阻塞线程。

还有 [withLock] 扩展函数，可以方便的替代常用的
`mutex.lock(); try { …… } finally { mutex.unlock() }` 模式：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

//sampleStart
val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // 用锁保护每次自增
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-06.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

此示例中锁是细粒度的，因此会付出一些代价。但是<!--
-->对于某些必须定期修改共享状态的场景，它是一个不错的选择，但是没有自然线程可以<!--
-->限制此状态。

### Actors

一个 [actor](https://en.wikipedia.org/wiki/Actor_model) 是由协程、
被限制并封装到该协程中的状态<!--
-->以及一个与其它协程通信的 _通道_ 组合而成的一个实体。一个简单的 actor 可以简单的写成一个函数，
但是一个拥有复杂状态的 actor 更适合由类来表示。

有一个 [actor] 协程构建器，它可以方便地将 actor 的邮箱通道组合到其<!--
-->作用域中（用来接收消息）、组合发送 channel 与结果集对象，这样<!--
-->对 actor 的单个引用就可以作为其句柄持有。

使用 actor 的第一步是定义一个 actor 要处理的消息类。
Kotlin 的[密封类](https://kotlinlang.org/docs/reference/sealed-classes.html)很适合这种场景。
我们使用 `IncCounter` 消息（用来递增计数器）和 `GetCounter` 消息（用来获取值）来定义 `CounterMsg` 密封类。
后者需要发送回复。[CompletableDeferred] 通信<!--
-->原语表示未来可知（可传达）的单个值，
这里被用于此目的。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// 计数器 Actor 的各种类型
sealed class CounterMsg
object IncCounter : CounterMsg() // 递增计数器的单向消息
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 携带回复的请求
```

</div>

接下来我们定义一个函数，使用 [actor] 协程构建器来启动一个 actor：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// 这个函数启动一个新的计数器 actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor 状态
    for (msg in channel) { // 即将到来消息的迭代器
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

</div>

main 函数代码很简单：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.system.*

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同个动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 计数器 Actor 的各种类型
sealed class CounterMsg
object IncCounter : CounterMsg() // 递增计数器的单向消息
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 携带回复的请求

// 这个函数启动一个新的计数器 actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor 状态
    for (msg in channel) { // 即将到来消息的迭代器
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

//sampleStart
fun main() = runBlocking<Unit> {
    val counter = counterActor() // 创建该 actor
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    // 发送一条消息以用来从一个 actor 中获取计数值
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // 关闭该actor
}
//sampleEnd
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-sync-07.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

actor 本身执行时所处上下文（就正确性而言）无关紧要。一个 actor 是<!--
-->一个协程，而一个协程是按顺序执行的，因此将状态限制到特定协程<!--
-->可以解决共享可变状态的问题。实际上，actor 可以修改自己的私有状态，
但只能通过消息互相影响（避免任何锁定）。

actor 在高负载下比锁更有效，因为在这种情况下它总是有工作要做，而且根本不<!--
-->需要切换到不同的上下文。

> 注意，[actor] 协程构建器是一个双重的 [produce] 协程构建器。一个 actor 与它<!--
-->接收消息的通道相关联，而一个 producer 与它<!--
-->发送元素的通道相关联。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
<!--- END -->
