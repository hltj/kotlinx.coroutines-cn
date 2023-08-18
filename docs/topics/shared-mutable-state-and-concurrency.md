<!--- TEST_NAME SharedStateGuideTest -->

[//]: # (title: 共享的可变状态与并发)

协程可用多线程调度器（比如默认的 [Dispatchers.Default]）并行执行。这样就可以提出<!--
-->所有常见的并行问题。主要的问题是同步访问**共享的可变状态**。
协程领域对这个问题的一些解决方案类似于多线程领域中的解决方案，
但其它解决方案则是独一无二的。

## 问题

我们启动一百个协程，它们都做一千次相同的操作。<!--
-->我们同时会测量它们的完成时间以便进一步的比较：

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

我们从一个非常简单的动作开始：使用<!--
-->多线程的 [Dispatchers.Default] 来递增一个共享的可变变量。

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-01.kt)获取完整代码。
>
{type="note"}

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码最后打印出什么结果？它不太可能打印出“Counter = 100000”，因为一百个协程<!--
-->在多个线程中同时递增计数器但没有做并发处理。

## volatile 无济于事

有一种常见的误解：volatile 可以解决并发问题。让我们尝试一下：

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-02.kt)获取完整代码。
>
{type="note"}

<!--- TEST LINES_START
Completed 100000 actions in
Counter =
-->

这段代码运行速度更慢了，但我们仍然不总是最后得到“Counter = 100000”这个结果，因为 volatile 变量保证<!--
-->可线性化（这是“原子”的技术术语）读取和写入变量，但<!--
-->在大量动作（在我们的示例中即“递增”操作）发生时并不提供原子性。

## 线程安全的数据结构

一种对线程、协程都有效的常规解决方法，就是使用线程安全（也称为同步的、
可线性化、原子）的数据结构，它为需要在共享状态上执行的相应<!--
-->操作提供所有必需的同步处理。<!--
-->在简单的计数器场景中，我们可以使用具有 `incrementAndGet` 原子操作的 `AtomicInteger` 类：

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-03.kt)获取完整代码。
>
{type="note"}

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这是针对此类特定问题的最快解决方案。它适用于普通计数器、集合、队列和其他<!--
-->标准数据结构以及它们的基本操作。然而，它并不容易被扩展来应对复杂<!--
-->状态、或一些没有现成的线程安全实现的复杂操作。

## 以细粒度限制线程

_限制线程_ 是解决共享可变状态问题的一种方案：对特定共享<!--
-->状态的所有访问权都限制在单个线程中。它通常应用于 UI 程序中：所有 UI 状态都局限于<!--
-->单个事件分发线程或应用主线程中。这在协程中很容易实现，通过使用一个单线程上下文。

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-04.kt)获取完整代码。
>
{type="note"}

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行非常缓慢，因为它进行了 _细粒度_ 的线程限制。每个增量操作都得使用
[withContext(counterContext)] 块<!--
-->从多线程 [Dispatchers.Default] 上下文切换到单线程上下文。

## 以粗粒度限制线程

在实践中，线程限制是在大段代码中执行的，例如：状态更新类业务逻辑中大部分<!--
-->都是限于单线程中。下面的示例演示了这种情况，
在单线程上下文中运行每个协程。

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-05.kt)获取完整代码。
>
{type="note"}

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

这段代码运行更快而且打印出了正确的结果。

## 互斥

该问题的互斥解决方案：使用永远不会同时执行的 _关键代码块_
来保护共享状态的所有修改。在阻塞的世界中，你通常会为此目的使用 `synchronized` 或者 `ReentrantLock`。
在协程中的替代品叫做 [Mutex] 。它具有 [lock][Mutex.lock] 和 [unlock][Mutex.unlock] 方法，
可以隔离关键的部分。关键的区别在于 `Mutex.lock()` 是一个挂起函数，它不会阻塞线程。

还有 [withLock] 扩展函数，可以方便的替代常用的
`mutex.lock(); try { …… } finally { mutex.unlock() }` 模式：

<!--- CLEAR -->

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
{kotlin-runnable="true" kotlin-min-compiler-version="1.3"}

> 可以在[这里](../../kotlinx-coroutines-core/jvm/test/guide/example-sync-06.kt)获取完整代码。
>
{type="note"}

<!--- TEST ARBITRARY_TIME
Completed 100000 actions in xxx ms
Counter = 100000
-->

此示例中锁是细粒度的，因此会付出一些代价。但是<!--
-->对于某些必须定期修改共享状态的场景，它是一个不错的选择，但是没有自然线程可以<!--
-->限制此状态。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->

[Dispatchers.Default]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[withContext]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html

<!--- INDEX kotlinx.coroutines.sync -->

[Mutex]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html

<!--- END -->
