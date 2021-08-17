<!--- TEST_NAME ComposingGuideTest -->

**目录**

<!--- TOC -->

* [组合挂起函数](#组合挂起函数)
  * [默认顺序调用](#默认顺序调用)
  * [使用 async 并发](#使用-async-并发)
  * [惰性启动的 async](#惰性启动的-async)
  * [async 风格的函数](#async-风格的函数)
  * [使用 async 的结构化并发](#使用-async-的结构化并发)

<!--- END -->

## 组合挂起函数

本节介绍了将挂起函数组合的各种方法。

### 默认顺序调用

假设我们在不同的地方定义了两个进行某种调用远程服务或者进行计算的<!--
-->挂起函数。我们只假设它们都是有用的，但是实际上它们在这个示例中只是<!--
-->为了该目的而延迟了一秒钟：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了一些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了一些有用的事
    return 29
}
```

</div>


如果需要按 _顺序_ 调用它们，我们接下来会做什么——首先调用 `doSomethingUsefulOne` _接下来_
调用 `doSomethingUsefulTwo`，并且计算它们结果的和吗？
实际上，如果我们要根据第一个函数的结果来决定是否我们需要<!--
-->调用第二个函数或者决定如何调用它时，我们就会这样做。

我们使用普通的顺序来进行调用，因为这些代码是运行在协程中的，只要像常规的<!--
-->代码一样 _顺序_ 都是默认的。下面的示例展示了测量<!--
-->执行两个挂起函数所需要的总时间：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了一些有用的事
    return 29
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-01.kt)获取完整代码。

它的打印输出如下：

```text
The answer is 42
Completed in 2017 ms
```

<!--- TEST ARBITRARY_TIME -->

### 使用 async 并发

如果 `doSomethingUsefulOne` 与 `doSomethingUsefulTwo` 之间没有依赖，并且<!--
-->我们想更快的得到结果，让它们进行 _并发_ 吗？这就是 [async] 可以帮助我们的地方。
 
在概念上，[async] 就类似于 [launch]。它启动了一个单独的协程，这是一个轻量级的线程<!--
-->并与其它所有的协程一起并发的工作。不同之处在于 `launch` 返回一个 [Job] 并且<!--
-->不附带任何结果值，而 `async` 返回一个 [Deferred] —— 一个轻量级的非阻塞 future，
这代表了一个将会在稍后提供结果的 promise。你可以使用 `.await()` 在一个延期的值上得到它的最终结果，
但是 `Deferred` 也是一个 `Job`，所以如果需要的话，你可以取消它。
 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了些有用的事
    return 29
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-02.kt)获取完整代码。

它的打印输出如下：

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

这里快了两倍，因为两个协程并发执行。
请注意，使用协程进行并发总是显式的。

### 惰性启动的 async

可选的，[async] 可以通过将 `start` 参数设置为 [CoroutineStart.LAZY] 而变为惰性的。
在这个模式下，只有结果通过 [await][Deferred.await]
获取的时候协程才会启动，或者在 `Job` 的 [start][Job.start]
函数调用的时候。运行下面的示例：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // 执行一些计算
        one.start() // 启动第一个
        two.start() // 启动第二个
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了些有用的事
    return 29
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-03.kt)获取完整代码。

它的打印输出如下：

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

因此，在先前的例子中这里定义的两个协程没有执行，但是控制权在于<!--
-->程序员准确的在开始执行时调用 [start][Job.start]。我们首先
调用 `one`，然后调用 `two`，接下来等待这个协程执行完毕。

注意，如果我们只是在 `println` 中调用 [await][Deferred.await]，而没有在<!--
-->单独的协程中调用 [start][Job.start]，这将会导致顺序行为，直到 [await][Deferred.await] 启动该协程
执行并等待至它结束，这并不是惰性的预期用例。
在计算一个值涉及挂起函数时，这个 `async(start = CoroutineStart.LAZY)` 的用例用于替代<!--
-->标准库中的 `lazy` 函数。

### async 风格的函数

我们可以定义异步风格的函数来 _异步_ 的调用 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo`
并使用 [async] 协程建造器并带有一个显式的 [GlobalScope] 引用。
我们给这样的函数的名称中加上<!--
-->“……Async”后缀来突出表明：事实上，它们只做异步计算并且需要<!--
-->使用延期的值来获得结果。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
// somethingUsefulOneAsync 函数的返回值类型是 Deferred<Int>
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// somethingUsefulTwoAsync 函数的返回值类型是 Deferred<Int>
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

</div>

注意，这些 `xxxAsync` 函数**不是** _挂起_ 函数。它们可以在任何地方使用。
然而，它们总是在调用它们的代码中意味着<!--
-->异步（这里的意思是 _并发_ ）执行。
 
下面的例子展示了它们在协程的外面是如何使用的：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

//sampleStart
// 注意，在这个示例中我们在 `main` 函数的右边没有加上 `runBlocking`
fun main() {
    val time = measureTimeMillis {
        // 我们可以在协程外面启动异步执行
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // 但是等待结果必须调用其它的挂起或者阻塞
        // 当我们等待结果的时候，这里我们使用 `runBlocking { …… }` 来阻塞主线程
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
//sampleEnd

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了些有用的事
    return 29
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-04.kt)获取完整代码。

<!--- TEST ARBITRARY_TIME
The answer is 42
Completed in 1085 ms
-->

> 这种带有异步函数的编程风格仅供参考，因为这在其它编程语言中<!--
-->是一种受欢迎的风格。在 Kotlin 的协程中使用这种风格是**强烈不推荐**的，
原因如下所述。

考虑一下如果 `val one = somethingUsefulOneAsync()` 这一行和 `one.await()` 表达式这里在代码中有逻辑错误，
并且程序抛出了异常以及程序在操作的过程中中止，将会发生什么。
通常情况下，一个全局的异常处理者会捕获这个异常，将异常打印成日记并报告给开发者，但是反之<!--
-->该程序将会继续执行其它操作。但是这里我们的 `somethingUsefulOneAsync` 仍然在后台执行，
尽管如此，启动它的那次操作也会被终止。这个程序将不会进行结构化<!--
-->并发，如下一小节所示。

### 使用 async 的结构化并发

让我们使用[使用 async 的并发](#使用-async-的结构化并发)这一小节的例子并且提取出一个函数<!--
-->并发的调用 `doSomethingUsefulOne` 与 `doSomethingUsefulTwo` 并且返回它们两个的结果之和。
由于 [async] 被定义为了 [CoroutineScope] 上的扩展，我们需要将它写在<!--
-->作用域内，并且这是 [coroutineScope][_coroutineScope] 函数所提供的：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

</div>

这种情况下，如果在 `concurrentSum` 函数内部发生了错误，并且它抛出了一个异常，
所有在作用域中启动的协程都会被取消。

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
//sampleStart
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
//sampleEnd    
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // 假设我们在这里做了些有用的事
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // 假设我们在这里也做了些有用的事
    return 29
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-05.kt)获取完整代码。

从上面的 `main` 函数的输出可以看出，我们仍然可以同时执行这两个操作：

```text
The answer is 42
Completed in 1017 ms
```

<!--- TEST ARBITRARY_TIME -->

取消始终通过协程的层次结构来进行传递：

<!--- CLEAR -->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // 模拟一个长时间的运算
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-compose-06.kt)获取完整代码。

请注意，如果其中一个子协程（即 `two`）失败，第一个 `async` 以及等待中的父协程都会被取消<!--
-->：
```text
Second child throws an exception
First child was cancelled
Computation failed with ArithmeticException
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[CoroutineStart.LAZY]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-l-a-z-y.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[Job.start]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/start.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- END -->
